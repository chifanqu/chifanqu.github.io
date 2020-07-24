---
layout: post
title: '逆向安卓内核绕过TracerPID反调'
date: 2020-07-22
author: 吃饭去
cover: '../assert'
tags: android
typora-root-url: a\a
typora-copy-images-to: ..\assets\img
---

# 逆向安卓内核绕过TracerPID反调

## 1、背景

​	最近开始研究android逆向，发现有种反调试方法是判断/proc/self/status里面的TracerPid，如果被调试的话会变成调试进程的pid。当然可以通过逆向so来patch掉，但感觉每次都做的话有点麻烦。在网上看到可以通过修改内核，使得TracerPid的值始终为0，绕过反调试。方法有两种：[1、编译内核](https://blog.csdn.net/u012417380/article/details/73353670)，[2、逆向修改内核](https://www.xmsec.cc/re-modify-kernel-bypass-antidebug/)。手上的测试机是三丧S5 G9008v，一直找不到内核源码，所以只能使用第二种方法逆向修改内核了。



## 2、逆向修改内核

#### 1、获取boot.img

​	获取内核文件的话需要root权限，先使用命令找到具体的文件路径，即boot指向的文件：

```shell
ls -l /dev/block/platform/msm*/by-name
dd if=/dev/block/mmcblk0p15 of=/data/local/tmp/boot.img
adb pull /data/local/tmp/boot.img boot.img
```

![image-20200722223212167](/../../../assets/img/image-20200722223212167.png)



#### 2、获取kernel

​	这一步是解开boot.img，有很多工具都可以做到，我用的是BOOTIMG.exe。解压后可以得到kernel，就是内核文件了。

​	其实不用工具也可以得到的，直接使用`binwalk -e boot.img`也可以自动分解出来。内核文件即第一个发现的压缩包，可能的格式为gzip、lzo、xz等。因为网上的教程都是gzip格式的，然后找gzip的文件头导致一直找不到，后来才发现kernel的压缩有很多种，这里也算一个坑。

![image-20200722224024372](/../../../assets/img/image-20200722224024372.png)



#### 3、修改kernel

​	在修改kernel前还需要找两个函数的地址：

```
echo 0 > /proc/sys/kernel/kptr_restrict
cat /proc/kallsyms | grep proc_pid_status
cat /proc/kallsyms | grep __task_pid_nr_ns
```

![image-20200722224634553](/../../../assets/img/image-20200722224634553.png)

​	binwalk自动解压kernel后就可以直接拖到ida里面分析了。选择ARM Little-endian，然后在`ROM start address`和`Loading address`里面填上0xc0008000，点ok即可。

![image-20200722224810873](/../../../assets/img/image-20200722224810873.png)

![image-20200722224923034](/../../../assets/img/image-20200722224923034.png)

​	接下来跳转到`proc_pid_status`函数的起始位置（刚刚查询得到的地址），然后按`p`将这个函数解析出来。再跳转到`__task_pid_nr_ns`函数的起始位置，按`x`分析调用关系。可以在最下面找到`proc_pid_status`函数被调用的位置，双击进入即可。

![image-20200722225454195](/../../../assets/img/image-20200722225454195.png)

​	直到这里和其余教程的都差不多，但是进入函数后发现并没有其余教程提到的`MOVEQ R10, R0`，后来研究了下函数的作用，直接修改调用结果为0应该也可以。所以把`BL sub_C01CBC58 `(即双击后进入的位置)直接patch为`MOV R0,#0`再保存下就行了。

![image-20200722232652506](/../../../assets/img/image-20200722232652506.png)



#### 4、重新压缩kernel

​	patch后的kernel还需要再压缩后复制回boot.img。如果是gzip就按refz中教程的方法压缩，这里就不赘述了，主要来讲下另外两种情况lzo和xz。

##### lzo压缩

​	lzo压缩相对比较简单，在linux下使用lzop -9 kernel即可。但要注意由于lzop压缩后是带文件名的，会导致压缩后多出几个字节，可能会比原压缩后的kernel大，所以要删掉文件名。使用winhex打开，在最上面找到文件名，比如这里是kernel，前面的`06`表示文件名有6个字节。所以这需要将`06`改成`00`，然后再删掉`kernel`。

![image-20200722230659690](/../../../assets/img/image-20200722230659690.png)

​	压缩后必须要保证比原先的压缩包要小，不然回刷的话会有问题无法开机。

##### xz压缩

​	xz就相对比较麻烦了，之前不管怎么压缩都比原先的大。后来找到xz kernel压缩的代码，发现是使用xzkern进行压缩，看下源码吧：

```shell
BCJ=
LZMA2OPTS=
case $SRCARCH in
	x86)            BCJ=--x86 ;;
	powerpc)        BCJ=--powerpc ;;
	ia64)           BCJ=--ia64; LZMA2OPTS=pb=4 ;;
	arm)            BCJ=--arm ;;
	sparc)          BCJ=--sparc ;;
esac
exec xz --check=crc32 $BCJ --lzma2=$LZMA2OPTS,dict=32MiB
```

​	所以压缩的命令是

```shell
xz -9 --check=crc32 --arm --lzma2=,dict=32MiB kernel
```

​	这里同样需要保障压缩后的文件比原先的压缩文件小，xz文件以0x595A(YZ)结尾，需要自己计算下原压缩文件和修改后压缩文件的大小，如果大了的话是不行的。



#### 5、修改boot.img

​	压缩完并确定比原先的压缩文件小的话，就直接在`boot.img`的原位置覆盖回去就行了，一般是binwalk显示的第二个压缩头的位置。如果使用winhex的话，直接ctrl+b覆盖回去就行，很方便。

![image-20200722231623059](/../../../assets/img/image-20200722231623059.png)



## 3、回刷boot

​	在刷boot前记得先备份下原先的boot.img，这样即使开不了机也可以再刷回去，没啥损失。

​	使用TWRP将修改后的boot.img刷入boot区，重新开机就可以了。我自己试过两个rom，一个xz一个lzo，只要保证重新压缩的kernel和原先的一样或者更小都成功了。



## ref

1.[逆向修改内核，绕过TracerPID反调试](https://www.xmsec.cc/re-modify-kernel-bypass-antidebug/)

2.[逆向修改手机内核，绕过反调试](https://bbs.pediy.com/thread-207538.htm)

3.[Makefile.lib](https://review.lineageos.org/c/LineageOS/android_kernel_samsung_universal9810/+/213797/1/scripts/Makefile.lib)

4.[xz_wrap.sh](https://android.googlesource.com/kernel/common/+/android-trusty-4.4/scripts/xz_wrap.sh)