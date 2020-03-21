

author: [Halloweenwx](https://gitee.com/halloweenwx)

本周
1. Ubuntu下编译QEMU并简单测试
2. 阅读并翻译qemu-atc05.pdf
[TOC]
---
# QEMU编译
## 测试
### KVM
[kvm介绍](https://www.redhat.com/zh/topics/virtualization/what-is-KVM)

## QEMU上手
[官网](https://qemu.org)
[wiki](https://wiki.qemu.org)
[documentation](https://wiki.qemu.org/Documentation)

[github(镜像)](https://github.com/qemu/qemu)
[官方git](https://git.qemu.org/git/qemu.git)

1. 加入社区
[wiki:Maillist](https://wiki.qemu.org/Contribute/MailingLists)

2. 获取源码

wiki中建议使用linux平台

> Linux is QEMU's main host platform. Therefore it is the platform which gets most support.

>Please note that less developers work on QEMU for W32 / W64 hosts, so it might be less stable (but I don't think it is).

`git clone git.qemu.org/git/qemu.git`

3. 编译及测试

[wiki上的例子(Simple build and test with KVM)](https://wiki.qemu.org/Hosts/Linux)

[ubuntu配置kvm](https://www.sysgeek.cn/install-configure-kvm-ubuntu-18-04/)

* 注意
    * 量力而行，套娃（虚拟机中跑虚拟机）可能会很卡
    * vnc连接可以放在其他设备（例如手机或主系统，减轻虚拟机的负担）

![vnc-qemu](https://gitee.com/duolifeqnl/qemu_learning/raw/master/img/vnc-qemu.png)

* 如果想用SDL（当年OS课程那种）却发现只有一行`VNC server...`，那是少安装libsdl2-dev（wiki上说lidsdl1.2-dev,不知道有什么区别 `sudo apt install libsdl2-dev` 然后重新`configure`（会看到SDL support yes(2.0.8)）和`make`）

* 如果permission deny...kvm 看一下ubuntu配置kvm

![sdl-qemu](https://gitee.com/duolifeqnl/qemu_learning/raw/master/img/sdl-qemu.png)

* 小工具
    * [下载工具uGet+aira2](https://blog.csdn.net/fengyulinde/article/details/78309314)
    * [krdc: vnc客户端](https://kde.org/applications/internet/org.kde.krdc)
    * [Fedora-Live-Desktop-x86_64-20-1.iso](https://archives.fedoraproject.org/pub/archive/fedora/linux/releases/20/Live/x86_64/Fedora-Live-Desktop-x86_64-20-1.iso)

# 论文翻译
## 摘要：
介绍QEMU内部实现
* QEMU利用**可移植动态翻译器**实现高速的机器模拟，在一系列平台上(x86, PowerPC, ARM, Sparc, Alpha and MIPS)模拟一系列CPU（x86, PowerPC, ARM and Sparc)。
* 支持全系统模拟 —— 在虚拟机和Linux用户模式模拟（为一个目标CPU编译Linux进程可运行在另一个CPU）下运行一个完整的系统。

## 1. 介绍
* QEMU是一种机器模拟器：它可以在虚拟机中运行未修改的*目标*操作系统（例如Windows或Linux）及其所有应用程序。QEMU本身可以在多种主机操作系统（例如Linux，Windows和Mac OS X）上运行。和目标CPU可以不同。
* QEMU的主要用途是在另一个操作系统上运行一个操作系统，例如Linux上的Windows或Windows上的Linux。另一种用法是调试，因为可以轻松地停止虚拟机，并且可以检查、保存和还原其状态。可以通过添加新的机器描述和新的仿真设备来模拟特定的嵌入式设备。
* QEMU还集成了Linux用户模式仿真器，它是机器仿真器的子集，可在另一个CPU上为一个目标CPU运行Linux进程，主要用于测试交叉编译器的结果或测试CPU仿真器而无需启动完整的虚拟机。

QEMU由几个子系统组成：
* CPU仿真器
* 仿真的设备（例如VGA显示屏，16450串行端口，PS / 2鼠标和键盘，IDE硬盘，NE2000网卡等）
* 用于将模拟设备连接到相应主机设备的通用设备（例如，块设备，字符设备，网络设备）
* 实例化仿真设备的机器描述（例如PC，PowerMac，Sun4m）
* 调试器
* 用户界面

本文研究了QEMU使用的动态转换器的实现，该动态转换器将目标CPU指令在运行时翻译为主机指令集，并将生成的二进制代码存储在翻译缓存中以便可以重复使用。与解释器相比，目标指令仅被提取和解码一次。
通常动态翻译器很难从一台主机移植到另一台主机，因为必须重写整个代码生成器，它代表的工作量与将新目标添加到C编译器中的工作量差不多.QEMU更为简单，因为它只是连接了几台机器由GNU C编译器离线生成的代码。

CPU仿真器还面临其他更经典但困难的问题[2]：

* 管理翻译后的代码缓存
* 寄存器分配
* 状态码优化
* 直接块链接
* 内存管理
* 自修改代码支持
* 异常
* 硬件中断
* 用户模式仿真

## 2. 可移植动态翻译器
### 2.1 描述
* 第一步划分微操作。将目标CPU指令划分为若干更简单的操作——微操作(micro operations)。微操作由小段C语言编写，经GCC编译为目标文件。经过特别挑选后，只需要更少的微操作（几百个）就能组成目标CPU所有的指令和操作数的组合。由指令翻译为微操作的过程由完全由纯手工代码编写。这一阶段中，执行速度相比解释器部分要求不那么高，源码（微操作）优化了易读性和简洁性（?compactness）。
* 编译期工具`dyngen`，以包含微操作的目标文件作为输入，生成一个动态代码的产生器。该产生器运行时调用并产生一个完整的宿主机器可用的函数（host function），用来连接微操作。
* 以上过程类似于参考文献[1]（），但在编译期做了更多的工作以达到更好的效果。特别地，一个关键思想是，QEMU的常参可以传递给微操作。为了这个目的，由GCC为每个常参生成伪代码重定位（dummy code relocations），这使得`dyngen`可以找到重定位并产生合适的C代码来解析他们并构建动态代码。重定位同样是为了能够找到静态数据和其他微操作函数的引用。
### 2.2 例子（略）
### 2.3 Dyngen 实现
`dyngen`是QEMU翻译过程的核心，在包含微操作的目标文件上运行`dyngen`包含以下步骤——
* 解析（parse）目标文件并获得符号表、重定位入口和代码区。这个过程取决于宿主机的目标文件格式（支持ELF...）
* 微操作位于代码区，利用符号表定位。通过执行一个特定的方法（method）获得代码副本的开始和结束。通常，函数的前言（prologue）和后语（epilogue）都会被跳过。

> In assembly language programming, the function prologue is a few lines of code at the beginning of a function, which prepare the stack and registers for use within the function. Similarly, the function epilogue appears at the end of the function, and restores the stack and registers to the state they were in before the function was called.

* 检查每个微操作的的重定位以取得常参的数量。检测常参的重定位基于它们特定的的命名方式，如 __op_paramN.

* ？？C代码中生成一份内存的副本，用来复制微操作代码。使用每个微操作的重定位作为补充，用来正确重定位复制后的代码。重定位*补充*因宿主机不同而不同。

* 对于某些主机（例如ARM），必须将常量存储在生成的代码附近，因为常量是通过PC相对负载以较小的位移进行访问的。完成了主机特定的遍历以在生成的代码中重新定位这些常量。

当编译微操作代码时，利用一组GCC标记来操纵函数前言和后语的生成，使得它们的形式易于解析。虚拟程序集宏会强制GCC始终通过单个返回指令终止与每个微操作相对应的功能。如果在单个微操作中生成了多个返回指令，则代码的拼接将不起作用。

## 3 实现细节
### 3.1 翻译块和翻译缓存

当QEMU第一次遇到一段目标代码时，它会将其转换为主机代码，直到下一个跳转/指令，或者以无法在转换时推断的方式修改静态CPU状态的指令。我们称这些基本块为翻译块（TB）。

16 MB的高速缓存用来保存最近使用的TB。为简单起见，当高速缓存满时将其完全清空。

**静态CPU状态**定义为进入TB时在转换时被视为已知的CPU状态的一部分。例如，在翻译时所有目标上都知道程序计数器（PC）。在x86上，静态CPU状态包括更多数据，以便能够生成更好的代码。例如，重要的是要知道CPU是处于保护模式还是实模式，用户模式还是内核模式，或者默认操作数大小是16位还是32位。

### 3.2 寄存器分配

* QEMU使用*固定的*寄存器分配。这意味着每个目标CPU寄存器都映射到固定的主机寄存器或内存地址。在大多数主机上，我们仅将所有目标寄存器映射到内存，并只在主机寄存器中存储一些临时变量。临时变量的分配在每个目标CPU描述中进行硬编码。这种方法的优点是简单和可移植。

* QEMU的未来版本将使用动态临时寄存器分配器来消除在目标寄存器直接存储在主机寄存器中的情况下的一些不必要的移动。

### 3.3 状态码优化

* 良好的CPU状态码仿真（x86上的eflags寄存器）是获得良好性能的关键点。
* QEMU计算状态码时使用懒求值，它只存储一个操作数（称为CC_SRC），结果（称为CC_DST）和操作类型（称为CC_OP），而不是在每个x86指令之后计算条件代码。对于R = A + B这样的32位加法，有
CC_SRC=A;
CC_DST=R;
CC_OP=CC_OP_ADDL;
由CC_OP=CC_OP_ADDL得知这是个32位加法，同理从CC_SRC和CC_DST中得知A，B和R，然后仅在接下来的指令中需要状态码时，才恢复（recover）出所需的状态码，例如零结果（ZF），非正数（SF)，进位（CF）或越界（OF）。

可以在翻译时进一步优化状态码的计算，基于事实——一个翻译块TB的代码是一次性生成完整的。对生成的代码向后检查（backward pass），以查看接下来的代码是否未使用CC_OP，CC_SRC或CC_DST。在TB的末尾，我们认为这些变量被使用了，然后删除在接下来的代码中未使用的值。（*删除-？->优化*）
### 3.4 直接块链接（块间直接跳转）
* （通常）执行完每个TB之后，QEMU使用模拟的程序计数器（PC）和静态CPU状态等其他信息通过哈希表查找下一个TB。如果尚未翻译下一个TB，则启动新的翻译。 否则，将跳转到下一个TB。

* 为了加速最常见情况——新模拟的PC已知（例如在有条件的跳转之后），QEMU可以修补TB，使得它可以直接跳转到下一个TB。（*看起来是省略了查表过程*）

* 可移植最好的代码使用间接跳转。在某些主机（例如x86或PowerPC）上，直接对分支指令进行了修补，因此块间跳转就没有开销了。（*效率Vs可移植性*）

### 3.5 内存管理
* 对于系统仿真，QEMU使用`mmap()`系统调用来仿真目标MMU，只要仿真的OS不使用主机OS保留的区域即可工作。

* 为了能够启动任何操作系统，QEMU还支持软MMU，在这种模式下，每次内存访问都经过MMU的虚实地址转换，QEMU使用地址转换缓存来加速转换。

* 为了避免每次MMU映射更改时都会刷新翻译后的代码，QEMU使用了物理索引的翻译缓存，这意味着每个TB均以其物理地址进行索引。当MMU映射更改时，将重置TB的链接（即TB不能再直接跳转到另一个TB），因为跳转目标的物理地址可能会更改。
### 3.6 自修改代码(self-modifying code)和翻译后的代码无效
> self-modifying code is code that alters its own instructions while it is executing 
* 在大多数CPU上，自修改代码很容易处理，因为可以执行特定的代码缓存无效指令以发出代码已被修改的信号。它足以使对应的翻译代码无效。
* 但是，在诸如x86之类的CPU上，修改代码后，应用程序未发出指令高速缓存无效的信号，自修改代码是一个特殊的挑战.
* 当为TB生成翻译后的代码时，如果相应的主机内的页不是只读的，则对其进行写保护。如果对该页进行了写访问，则QEMU将使其中的所有翻译代码无效并重新启用对该页面的写访问权限。
* 通过维护给定页面中包含的每个翻译块的链接列表，可以有效地完成正确的翻译代码无效。维护其他链接列表，可以撤消直接块链接。

当使用软件MMU时，代码无效会更有效：如果给定的代码页由于写访问而经常失效，则将构建表示页面内所有代码的位图，每个存储在该页面中的内容都会检查该位图以查看如果确实需要使代码无效，则可以避免仅在页面中修改数据时使代码无效。

### 3.7 异常
* 当遇到诸如除数为零之类的异常时，`longjmp()`用于跳转到异常处理代码。当不使用软MMU时，由主机信号处理程序捕获无效的内存访问。

* QEMU在发生异常时始终能够检索确切的目标CPU状态，因此它支持精确的异常。对于大多数目标CPU状态，无需执行任何操作，因为它已由翻译后的代码显式存储和修改。对于未明确存储的目标CPU状态S（例如当前的程序计数器），需要通过重新翻译发生异常时之前的目标指令中S均已记录的TB（*该TB是第一个未记录S的TB？*）来检索。

### 3.8 硬件中断
* 为了提高速度，QEMU不会在每个TB上检查是否有硬件中断处于挂起状态，而是需要用户异步调用指定函数来告知某个中断正在挂起，此功能将重置当前正在执行的TB链。 确保执行将很快在CPU仿真器的主循环中返回，然后主循环测试是否有挂起的中断并处理该中断。

### 3.9 用户模式仿真
* QEMU支持用户模式仿真，以便使得为一个目标CPU编译的Linux进程能在另一个CPU上运行。
* 在CPU级别，用户模式仿真只是整个系统仿真的一个子集。由于QEMU假定用户内存映射是由主机OS处理的，因此不执行MMU仿真。QEMU包括一个通用的Linux系统调用转换器来处理字节序问题（大小端）和32/64位转换，因为QEMU支持异常，所以它精确地模拟了目标信号，每个目标线程都在一个主机线程中运行。

## 4. 移植工作

如何移植QMEU到一个新的CPU——

1. 移植`dyngen`
2. 微操作所使用的临时变量可以映射到主机专用寄存器，以优化性能。
3. 大多数主机CPU需要特定的指令，以保持指令缓存和内存之间的一致性。
4. 如果使用补丁的分支指令实现直接块链接，则必须提供某些特定的汇编宏。

QEMU的总体移植复杂度估计和动态链接器相同。

### 5. 性能
为了测量由于仿真开销，我们比较了[BYTEmark benchmark for linux](https://en.wikipedia.org/wiki/NBench)(*一个测试计算机性能的程序*)在原生模式下的x86主机上和在x86目标用户模式下的性能。

用户模式QEMU（版本0.4.2）在整数代码上比原生模式慢大约4倍，在浮点代码上，它慢了10倍，这可以理解为在静态CPU状态中缺少x86 FPU堆栈指针。处于静态CPU状态。在完整系统仿真中，软MMU的成本导致速度降低2倍。

在完整的系统仿真中，QEMU比Bochs快30倍。
用户模式QEMU比valgrind --skin = none版本1.9.6快1.2倍，这是手动编码的x86到x86动态转换器，通常用于调试程序。--skin = none选项可确保Valgrind不生成调试信息代码。

### 6. 结论和未来的工作

QEMU已经到了可以在日常工作中使用的地步，尤其是在模拟Windows等商业x86操作系统时。 [PowerPC]((https://wiki.qemu.org/Documentation/Platforms/PowerPC#Pre-built_binaries_for_Windows_and_OSX))的目标是运行Mac OS X，而[Sparc](https://wiki.qemu.org/Documentation/Platforms/SPARC)的目标是运行Linux。迄今为止，没有其他动态翻译器支持这么多主机上的这么多目标，这主要是因为低估了移植的复杂性。 QEMU方法似乎是性能和复杂性之间的良好折衷。

将来仍需要解决以下几点：

* 移植：PowerPC和x86主机上很好地支持QEMU。 Sparc，Alpha，ARM和MIPS上的其他端口？则需要完善。 QEMU还非常依赖于用于编译微操作定义的确切GCC版本。

* 完整的系统仿真：[ARM](https://wiki.qemu.org/Documentation/Platforms/ARM)和[MIPS](https://wiki.qemu.org/Documentation/Platforms/MIPS)目标需要添加。

* 性能：软MMU的性能还能再提高。一些关键的微操作也可以在**汇编程序**（*先前是C*）中进行手动编码，而无需在当前翻译框架中进行大量修改。 CPU主循环也可以在汇编器中手动编码。

* 虚拟化：当主机和目标相同时，可以按原样运行大多数代码。最简单的实现是像往常一样模拟目标内核代码，但照原样运行目标用户代码。

* 调试：可以添加高速缓存模拟和周期计数器以像SIMICS中那样构成调试器

