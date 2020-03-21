

# 介绍、安装与使用

author: [duolifeqnl](https://gitee.com/duolifeqnl)

---

## 什么是QEMU
官网上的介绍：
> QEMU is a generic and open source machine emulator and virtualizer.

简单说就是一个开源的虚拟机，可以实现在一个操作系统上通过模拟硬件资源，运行一个虚拟的操作系统。

## 安装
QEMU支持多平台的应用。由于我们是小组一同完成课程任务，故统一选择在Linux上进行安装、使用和学习。

值得一提的是，由于习惯了Windows操作系统，故之前若要实现Linux的操作都会使用WSL（Windows Subsystem for Linux），于是准备在WSL使用QEMU。但是QEMU依赖于KVM，但WSL并不支持KVM（[参考链接](https://stackoverflow.com/questions/54287695/using-kvm-with-wsl-trying-to-run-android-emulator-using-wsl)）：
> KVM only runs on an actual Linux kernel, not Windows Subsystem for Linux (which runs on a thin Linux emulation layer over the Windows kernel).

故最终放弃了使用WSL。又由于我的电脑使用VMware + Ubuntu会有轻微卡顿（有的时候远不止轻微），故最终采用了安装Ubuntu双系统的方式。*在Linux上科学上网使用Electron-ssr。*

由于之后需要学习项目，故采用源码编译的方式。理论上为方便后续开发，应该从官方Git clone 项目，但由于Ubuntu上暂时还没有解决Git下载速率慢的问题，故直接使用`wget`获取最新版本的源码。

```
wget https://download.qemu.org/qemu-4.2.0.tar.xz
tar xvJf qemu-4.2.0.tar.xz
cd qemu-4.2.0
```
若使用官方Git获取源码，则：

```
git clone git://git.qemu-project.org/qemu.git
```

之前尝试了在WSL安装基于全部CPU的完整QEMU，编译速度感人，故这次只编译x86部分。[安装和测试的参考链接](https://wiki.qemu.org/Hosts/Linux)

首先安装必要的依赖：

```
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev
```
进入`qemu`源码目录并编译:

```
# Switch to the QEMU root directory
cd qemu
# Configure QEMU for x86_64 only - faster build
./configure --target-list=x86_64-softmmu --enable-debug
# Build in parallel - my system has 4 CPUs
make -j4
```
但是在./configure时报错`Permission denied`，参考[此处](https://askubuntu.com/questions/623461/bash-configure-permission-denied)，**注意：此处的安装过程中设计到两次configure文件，故若再次报错，需要转到报错的文件configre.sh所在的文件夹中再次使用上面链接中的方法解决。** 另外，此处可能需要使用sudo权限。

创建虚拟机文件

```
# Create a disk for the VM
./qemu-img create -f qcow2 test.qcow2 16G
```
其中最后一个参数为给虚拟机分配的空间，为最大允许虚拟机占用的硬盘空间。

使用Fedora 20 为虚拟机。[安装镜像下载地址](https://archives.fedoraproject.org/pub/archive/fedora/linux/releases/20/Live/x86_64/Fedora-Live-Desktop-x86_64-20-1.iso)

将虚拟机安装镜像复制到qemu目录下，执行：

```
x86_64-softmmu/qemu-system-x86_64 -m 1024 -enable-kvm \
-drive if=virtio,file=test.qcow2,cache=none \
-cdrom Fedora-Live-Desktop-x86_64-20-1.iso \
-vnc :1
```
此时会启动VNC连接，可以使用VNC连接工具krdc，安装方式：

```
sudo apt-get install -y krdc
```
启动krdc客户端：

```
krdc
```
之后在启动的页面中连接`localhost:1`并且设置分辨率水平及屏幕大小等即可开始进入虚拟机安装。

之后使用虚拟机时在`qemu`目录下执行：

```
x86_64-softmmu/qemu-system-x86_64 -m 1024 -enable-kvm \
-drive if=virtio,file=test.qcow2,cache=none \
-vnc :1
```
之后通过krdc连接即可：

![](https://user-gold-cdn.xitu.io/2020/3/15/170de9ab1b5e9a79?w=1849&h=1040&f=png&s=1182882)

# 论文阅读
Bellard, F. (2005). QEMU, a fast and portable dynamic translator. USENIX 2005 Annual Technical Conference, 41–46.

由于精力所限，本周并没有全部读完论文，这部分留到下周一起总结。
