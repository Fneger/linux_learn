# 搭建QEMU模拟arm

## 1.uboot

### 1.1.下载

```
git clone https://gitlab.denx.de/u-boot/u-boot.git
```

 下载完后，可以看到 configs 目录下有针对这款开发板的配置文件 

```
vexpress_ca9x4_defconfig
```

### 1.2.编译uboot

```
make vexpress_ca9x4_defconfig
make CROSS_COMPILE=arm-linux-gnueabihf- all
```

 最终编译生成 elf 格式的可执行文件 u-boot 和纯二进制文件u-boot.bin，其中 QEMU 可以启动的为 elf 格式的可执行文件 u-boot 

## 2.下载内核

```
在https://www.kernel.org/pub/linux/kernel/v4.x/下载内核
或者
git clonegit://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

## 3.安装arm的交叉编译工具链

如果你订制一个交叉编译工具链，建议你使用 [crosstool-ng](http://crosstool-ng.org/)开源软件来构建 ， 但在这里建议直接安装arm的交叉编译工具链： 

```
sudo apt-get install gcc-arm-linux-gnueabi
```

## 4.编译Linux内核

 生成vexpress开发板子的config文件： 

```
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm vexpress_defconfig
```

编译：

```
make CROSS_COMPILE=arm-linux-gnueabi- ARCH=arm
```

 生成的内核镱像位于arch/arm/boot/zImage， 后续qemu启动时需要使用该镜像。 

## 5.下载和安装qemu模拟器

其实Ubuntu 12.04有qemu的安装包，但由于版本较低，对vexpress开发板支持不友好，建议下载高版本的qemu:

```
wget http://wiki.qemu-project.org/download/qemu-2.8.0.tar.bz2
```

 配置qemu前，需要安装几个软件包： 

```
sudo apt-get install zlib1g-dev
sudo apt-get install libglib2.0-0
sudo apt-get install libglib2.0-dev
```

 配置qemu，支持模拟arm架构下的所有单板： 

```
./configure --target-list=arm-softmmu --audio-drv-list=
```

编译和安装：

```
make
make install
```

## 6.测试qemu和内核能否运行成功

qemu已经安装好了，内核也编译成功了，到这里最好是测试一下，编译出来的内核是否OK，或者qemu对vexpress单板支持是否够友好。

运行命令很简单：

```
qemu-system-arm -M vexpress-a9 -m 512M -kernel /path/to/kernel/dir/arch/arm/boot/zImage -dtb  /path/to/kernel/dir/arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "console=ttyAMA0"
```

这里的 ***\*/path/to/kernel/dir/\**** 是内核的下载目录，或者编译目录。

如果看到内核启动过程中的打印，说明前面的搭建是成功的。

这里简单介绍下qemu命令的参数：

```
-M vexpress-a9 模拟vexpress-a9单板，你可以使用-M ?参数来获取该qemu版本支持的所有单板

-m 512M 单板运行物理内存512M

-kernel /path/to/kernel/dir/arch/arm/boot/zImage  告诉qemu单板运行内核镜像路径

-nographic 不使用图形化界面，只使用串口

-append "console=ttyAMA0" 内核启动参数，这里告诉内核vexpress单板运行，串口设备是那个tty。
```

注意：

我每次搭建，都忘了内核启动参数中的console=参数应该填上哪个tty，因为不同单板串口驱动类型不尽相同，创建的tty设备名当然也是不相同的。那vexpress单板的tty设备名是哪个呢？ 其实这个值可以从生成的.config文件CONFIG_CONSOLE宏找到。

如果搭建其它单板，需要注意内核启动参数的console=参数值，同样地，可从生成的.config文件中找到。

## 7.制作根文件系统

 到这里是否大功告成了呢？ 其实在上面的测试中，你会发现内核报panic，因为内核找不到根文件系统，无法启init进程。 

根文件系统要考虑两个方面：

1. 根文件系统的内容

    如果你看过《Linux From Scratch》，相信你会对这一步产生恐惧感，但如果一直从事嵌入式开发，就可以放下心来。根文件系统就是简单得不能再简单的几个命令集和态动态而已。为什么Linux From Scratch会有那么复杂，是因为它要制作出一个Linux发生版。但在嵌入式领域，几乎所有的东西，都是mini版本，根文件系统也不例外。

   本文制本的根文件系统 = busybox(包含基础的Linux命令)  + 运行库 + 几个字符设备

2. 根文件系统放在哪里

    其实依赖于每个开发板支持的存储设备，可以放到Nor Flash上，也可以放到SD卡，甚至外部磁盘上。最关键的一点是你要清楚知道开发板有什么存储设备。

    本文直接使用SD卡做为存储空间，文件格式为ext3格式



## 8.下载、编译和安装busybox/ **Buildroot** 

```
wget http://www.busybox.net/downloads/busybox-1.33.1.tar.bz2
make defconfig
make CROSS_COMPILE=arm-linux-gnueabi-
make install CROSS_COMPILE=arm-linux-gnueabi-
```

安装完成后，会在busybox目录下生成_install目录，该目录下的程序就是单板运行所需要的命令。

## 9.形成根目录结构

先在Ubuntu主机环境下，形成目录结构，里面存放的文件和目录与单板上运行所需要的目录结构完全一样，然后再打包成镜像（在开发板看来就是SD卡），这个临时的目录结构称为根目录

1.  创建rootfs目录（根目录），根文件系统内的文件全部放到这里

```
mkdir -p rootfs/{dev,etc/init.d,lib}
mkdir  dev  etc  lib  usr  var  proc  tmp  home  root  mnt   sys
```

2. 拷贝busybox命令到根目录下

```
sudo cp busybox-1.20.2/_install/* -ra rootfs/
```

3. 从工具链中拷贝运行库到lib目录下

```
sudo cp -P /usr/arm-linux-gnueabi/lib/* rootfs/lib/
```

4. 创建4个tty端终设备

```
sudo mknod rootfs/dev/tty1 c 4 1
sudo mknod rootfs/dev/tty2 c 4 2
sudo mknod rootfs/dev/tty3 c 4 3
sudo mknod rootfs/dev/tty4 c 4 4
```

5. 构建etc目录：（主要有etc/inittab文件 、etc/init.d/rcs、etc/fstab）

   ```
   cp –r busybox-1.16.1/examples/bootfloopy/etc/* rootfs/etc
   ```

   6.修改profile
   
   　　
   
   ```
   PATH=/bin:/sbin:/usr/bin:/usr/sbin      //可执行程序 环境变量
   
   　　export LD_LIBRARY_PATH=/lib:/usr/lib     //动态链接库 环境变量
   
   　　/bin/hostname osee
   
   　　USER="`id -un`"
   
   　　LOGNAME=$USER
   
   　　HOSTNAME='/bin/hostname'
   
   　　PS1='[\u@\h \W]# '              //显示主机名、当前路径等信息：
   ```
   
   7.修改 etc/init.d/rc.S文件
   
   ```
   /bin/mount -n -t ramfs ramfs /var
   
   　　/bin/mount -n -t ramfs ramfs /tmp
   
   　　/bin/mount -n -t sysfs none /sys
   
   　　/bin/mount -n -t ramfs none /dev
   
   　　/bin/mkdir /var/tmp
   
   　　/bin/mkdir /var/modules
   
   　　/bin/mkdir /var/run
   
   　　/bin/mkdir /var/log
   
   　　/bin/mkdir -p /dev/pts          //telnet服务需要
   
   　　/bin/mkdir -p /dev/shm          //telnet服务需要
   
   　　#echo /sbin/mdev > /proc/sys/kernel/hotplug//USB自动挂载需要
   
   　　/sbin/mdev -s     //启动mdev在/dev下自动创建设备文件节点
   	/sbin/mdev -d
   　　/bin/mount -a
   
   　　 #######配置网络################################
   
   　　 */sbin/ifconfig lo 127.0.0.1 netmask 255.0.0.0
   
   　　/sbin/ifconfig eth0 192.168.1.70
   　　/sbin/ifconfig eth0 netmask 255.255.255.0
   　　/sbin/route add default gw 192.168.1.1 eth0
   
   　　/sbin/ifconfig eth1 192.168.1.71 netmask 255.255.255.0
   　　/sbin/route add default gw 192.168.1.1 eth1*
   ```
   
   8.修改etc/fstab文件，增加以下文件
   
   ```
   none  /dev/pts  devpts  mode=0622    0 0
       tmpfs  /dev/shm  tmpfs  defaults    0 0
   ```
   
   

## 10.制作根文件系统镜像

 [移植busybox构建最小根文件系统的步骤详解_Linux_脚本之家 (jb51.net)](https://www.jb51.net/article/164845.htm) 

1. 生成32M大小的镜像

```
 dd if=/dev/zero of=a9rootfs.ext3 bs=1M count=32
```



2. 格式化成ext3文件系统

```
mkfs.ext3 a9rootfs.ext3
```



3.  将文件拷贝到镜像中

```
sudo mkdir tmpfs

sudo mount -t ext3 a9rootfs.ext3 tmpfs/ -o loop

sudo cp -r rootfs/*  tmpfs/

sudo umount tmpfs
```

## 11.系统启动运行

完成上述所有步骤之后，就可以启动qemu来模拟vexpress开发板了，命令参数如下：

```
qemu-system-arm -M vexpress-a9 -m 512M -kernel /path/to/kernel/dir/arch/arm/boot/zImage -dtb  /path/to/kernel/dir/arch/arm/boot/dts/vexpress-v2p-ca9.dtb -nographic -append "root=/dev/mmcblk0  console=ttyAMA0" -sd a9rootfs.ext3
```

 

从内核启动打印，到命令行提示符出现，激动人心的时刻出现了……



# busybox支持mdev配置

##  在系统执行`mdev -s`前需要执行如下操作： 

###  1.配置内核 

```
make menuconfig
General setup ---->
Configure standard kernel features (for small systems) ---->
   [*] load all symbols for debugging/ksymoops
   [*] Include all symbols in kallsyms
   [*] Support for hot-pluggable devices
   [*] Enable support for printk
```

### 2.配置busybox 

```
make menuconfig
Linux System Utilities ---->
[*] mdev
[*] Support /etc/mdev.conf
[*] Support subdirs/symlinks
[*] Support regular expressions substitutions when renaming device
[*] Support command execution at device addition/removal
[*] Support loading of firmwares
```

###  3.系统启动时 

```
Vi  /etc/init.d/rcS
    mount -t tmpfs tmpfs /dev 
    mkdir /dev/pts
    mount -t devpts devpts /dev/pts
    mount -t proc proc /proc 
    mount -t sysfs sysfs /sys
    echo /sbin/mdev>/proc/sys/kernel/hotplug//启动热插拔事件；
    mdev –s
```

 首先挂载`/dev`、`/dev/pts`、`/proc`和`/sys`文件系统，mdev需要用到这些文件系统。然后告诉系统当有设备热插拔时，使用mdev来处理。最后执行`mdev -s`来扫描系统中的设备和驱动等。 

## 配置文件/etc/mdev.conf

 系统中的hotplug是通过mdev.conf文件来生成设备节点的，该配置文件格式如下： 

### 1.基本格式

```
<device regex>   <uid>:<gid>  <octal permissions>
<device regex>       :设备名称，支持正则表达式如hd[a-z][0-9]*等
<uid>:<gid>          :用户ID和组ID
<octal permissions>  :八进制表示的设备属性
```

###  2.执行脚本格式 

```
<device regex>   <uid>:<gid>  <octal permissions> [=|>path] [@|$|*]
[=|>path]:这个选项可以更改设备节点的命名和路径，如：
      <1> =/driver: 可以将设备节点移动到driver目录下
      <2> =newname: 可以讲设备节点改为newname命名
      <3> >/driver/newname: 可以在/driver目录下创建一个设备节点的链接，并命名为newname
[@|$|*]:这个选项当设备匹配成功时，执行指令，这个指令可以是自己编写的脚本。前面的符号含义如下：
      <1>@:在设备节点创建完执行
      <2>$:在设备节点删除前执行
      <3>*:在设备节点创建完和删除前执行
此外在mdev成功匹配设备后会设置两个系统变量$MDEV和$ACTION。其中$MDEV用来存放匹配到的设备名，$ACTION用来存放设备插拔状态其值为add和remove。这两个变量可以在脚本中使用。
```

### 脚本实例

```
mdev.conf
# system all-writable devices
full    0:0 0666
null    0:0 0666
ptmx    0:0 0666
random  0:0 0666
tty 0:0 0666
zero    0:0 0666
 
# console devices
tty[0-9]*   0:5 0660
vc/[0-9]*   0:5 0660
 
# serial port devices
s3c2410_serial0 0:5 0666    =ttySAC0
s3c2410_serial1 0:5 0666    =ttySAC1
s3c2410_serial2 0:5 0666    =ttySAC2
s3c2410_serial3 0:5 0666    =ttySAC3
 
# loop devices 
loop[0-9]*  0:0 0660    =loop/
 
# i2c devices
i2c-0   0:0 0666    =i2c/0
i2c-1   0:0 0666    =i2c/1
 
# frame buffer devices
fb[0-9] 0:0 0666
 
# input devices
mice    0:0 0660    =input/
mouse.* 0:0 0660    =input/
event.* 0:0 0660    =input/
ts.*    0:0 0660    =input/
 
# rtc devices
rtc0    0:0 0644    >rtc
rtc[1-9]    0:0 0644
 
# misc devices
mmcblk0p1   0:0 0600    =sdcard */bin/hotplug.sh
mmcblk0 0:0 0600    =mmcblk0 */bin/hotplug.sh
sda1    0:0 0600    =udisk * /bin/hotplug.sh
```

```
/bin/hotplug.sh
#!/bin/sh
case $MDEV in
sda1)
    DEVNAME=udisk
    MOUNTPOINT=/udisk
    ;;
mmcblk0p1)
    DEVNAME=sdcard
    MOUNTPOINT=/sdcard
    ;;
mmcblk0)
    DEVNAME=mmcblk0
    MOUNTPOINT=/sdcard 
    ;;  
*)
    exit 0
    ;;
esac
 
case $ACTION in
remove)
    /bin/umount $MOUNTPOINT || true
    rmdir $MOUNTPOINT >/dev/null 2>&1 || true
    ;;
*)
    /bin/mkdir $MOUNTPOINT > /dev/null 2>&1 || true
    /bin/mount -o sync -o noatime -o nodiratime -t vfat /dev/$DEVNAME $MOUNTPOINT > /dev/null 2>&1 || true
    ;;
esac
 
exit 0
```

# linux 内核添加驱动步骤

```
1.修改arch/arm/Kconfig,包含所需添加驱动Kconfig，例如：
	source "drivers/globalmem/Kconfig"

2.修改drivers/Kconfig,包含所需添加驱动Kconfig，例如：
	source "drivers/globalmem/Kconfig"
	
3.修改drivers/Makefile,包含所需添加驱动源码目录，例如：
	obj-$(CONFIG_GLOBALMEM)        += globalmem/
	
4.修改所需添加驱动Kconfig

5.修改所需添加驱动Makefile
```

# github

fneger_john@163.com

270933662@qq.com

199291JIANG..tao