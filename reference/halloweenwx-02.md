
author: [Halloweenwx](https://gitee.com/halloweenwx)

本周
1. gdb调试运行上周的成果，单步调试QEMU运行过程
2. 参考一些博文了解一些重要的函数和数据结构
---
# QEMU
## gdb
* [gdb](https://www.gnu.org/software/gdb/)
* [gdb tutorial](https://www.cs.cmu.edu/~gilpin/tutorial/)

编译
`./configure --target-list=x86_64-softmmu --enable-debug `
`make -j2`

直接运行：

`sudo x86_64-softmmu/qemu-system-x86_64 -m 1024 -enable-kvm -drive if=virtio,file=test.qcow2,cache=none -cdrom ~/Downloads/Fedora-Live-Desktop-x86_64-20-1.iso`

转化为gdb运行：

`sudo gdb x86_64-softmmu/qemu-system-x86_64`
`set args -m 1024 -enable-kvm -drive if=virtio,file=test.qcow2,cache=none -cdrom ~/Downloads/Fedora-Live-Desktop-x86_64-20-1.iso`

`run` 

### 奇怪的地方

> Thread 4 "qemu-system-x86" received signal SIGUSR1, User defined signal 1.

[据说是别人定义的断点](https://peter.quantr.hk/2013/02/when-you-debugging-qemu-it-will-keep-firing-sigusr1-so-gdb-will-always-catched-in-some-breakpooints-that-are-not-defined-by-you/)

## 源码分析

[参考1](https://www.jianshu.com/p/db8c20aa6a69)

[参考2](https://blog.csdn.net/dj0379/article/details/54926443)

重要的函数
* softmmu/main.c main(int argc, char **argv, char **envp)
* softmmu/vl.c qemu_init(argc, argv, envp);
* qemu_main_loop();
* qemu_cleanup();
* tcg/ 微代码生成相关

重要的数据结构
* include/hw/core/cpu.h CPUState 
...
