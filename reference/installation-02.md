
## ubuntu安装qemu

author: [Myriad-Dreamin](https://github.com/Myriad-Dreamin)

---

安装好qemu的依赖以后，运行：

```bash
kamiyoru@kamiya:~/work/c/$ cd qemu
kamiyoru@kamiya:~/work/c/qemu$ mkdir build
kamiyoru@kamiya:~/work/c/qemu$ cd build
kamiyoru@kamiya:~/work/c/qemu/build$ ./configure
kamiyoru@kamiya:~/work/c/qemu/build$ make
```

为了方便查看有哪些二进制文件，我们运行下面两个命令：

```bash
kamiyoru@kamiya:~/work/c/qemu/build$ mkdir ../bin
kamiyoru@kamiya:~/work/c/qemu/build$ find ./ qemu-* -maxdepth 3	 | grep -v "\.[hod]" | grep qemu- | xargs -i cp {} ../bin
cp: -r not specified; omitting directory './tests/qemu-iotests'
```

`ls`一下:

```bash
kamiyoru@kamiya:~/work/c/qemu/build$ cd ..
kamiyoru@kamiya:~/work/c/qemu$ ls bin
check               qemu-hppa          qemu-mipsn32     qemu-riscv64         qemu-system-hppa          qemu-system-nios2    qemu-system-tricore
common.env          qemu-i386          qemu-mipsn32el   qemu-s390x           qemu-system-i386          qemu-system-or1k     qemu-system-unicore32
qemu-aarch64        qemu-img           qemu-nbd         qemu-sh4             qemu-system-lm32          qemu-system-ppc      qemu-system-x86_64
qemu-aarch64_be     qemu-io            qemu-nios2       qemu-sh4eb           qemu-system-m68k          qemu-system-ppc64    qemu-system-xtensa
qemu-alpha          qemu-m68k          qemu-or1k        qemu-sparc           qemu-system-microblaze    qemu-system-riscv32  qemu-system-xtensaeb
qemu-arm            qemu-microblaze    qemu-ppc         qemu-sparc32plus     qemu-system-microblazeel  qemu-system-riscv64  qemu-tilegx
qemu-armeb          qemu-microblazeel  qemu-ppc64       qemu-sparc64         qemu-system-mips          qemu-system-s390x    qemu-x86_64
qemu-bridge-helper  qemu-mips          qemu-ppc64abi32  qemu-system-aarch64  qemu-system-mips64        qemu-system-sh4      qemu-xtensa
qemu-cris           qemu-mips64        qemu-ppc64le     qemu-system-alpha    qemu-system-mips64el      qemu-system-sh4eb    qemu-xtensaeb
qemu-edid           qemu-mips64el      qemu-pr-helper   qemu-system-arm      qemu-system-mipsel        qemu-system-sparc    socket_scm_helper
qemu-ga             qemu-mipsel        qemu-riscv32     qemu-system-cris     qemu-system-moxie         qemu-system-sparc64  venv
```

这里添加一下虚拟环境，方便我们使用qemu指令，并避免污染全局环境：

```bash
kamiyoru@kamiya:~/work/c/qemu$ echo '/bin/bash --rcfile $(dirname "BASH_SOURCE[0]" && pwd)/.bashrc' >> bin/venv
kamiyoru@kamiya:~/work/c/qemu$ chmod +x bin/venv
kamiyoru@kamiya:~/work/c/qemu$ cat >> bin/.bashrc << EOF
> source ~/.bashrc
> current_path=$(cd $(dirname "${BASH_SOURCE[0]}") && pwd)
> export PATH=$PATH:$current_path
> export PS1=" \[\033[01;33m\](qemu-venv)\[\033[00m\] $PS1"
EOF
kamiyoru@kamiya:~/work/c/qemu$ bin/venv
```

#### 测试编译成功的二进制文件

```bash
 (qemu-venv) kamiyoru@kamiya:~/work/c/qemu$ mkdir work
 (qemu-venv) kamiyoru@kamiya:~/work/c/qemu$ cd work/
```

使用`qemu-img`创建镜像文件：

```bash
 (qemu-venv) kamiyoru@kamiya:~/work/c/qemu/work$ qemu-img create -f qcow2 test-vm-1.qcow2 10G
Formatting 'test-vm-1.qcow2', fmt=qcow2 size=10737418240 cluster_size=65536 lazy_refcounts=off refcount_bits=16
```

解析:

+ `-f`后跟随一个字符串，指定镜像的格式。
+ 接下来两个参数分别是镜像名和镜像大小。

下面以`x86_64`架构运行镜像：

```bash
 (qemu-venv) kamiyoru@kamiya:~/work/c/qemu/work$ qemu-system-x86_64 test-vm-1.qcow2
VNC server running on 127.0.0.1:5900
```

`qemu`启动了一个远程VNC服务器，我们可以用ubuntu自带的远程桌面连接这个服务器

Connect选项中选用VNC协议，Host填`127.0.0.1:5900`，这样就连接上服务器了。

![Screen Remote Desktop Viewer](/home/kamiyoru/Pictures/Screenshot from 2020-03-20 19-02-54.png)

这个结果并不出人意料，因为我们还没有安装系统镜像，但是已经说明了我们的二进制文件编译成功了。
