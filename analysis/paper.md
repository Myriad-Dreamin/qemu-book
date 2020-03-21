## 对QEMU模拟原理论文的注解与评估

author: [Myriad-Dreamin](https://github.com/Myriad-Dreamin)

---

QEMU的动态翻译技术原理参考[QEMU, a Fast and Portable Dynamic Translator-Fabrice Bellard](http://archives.cse.iitd.ernet.in/~sbansal/csl862-virt/2010/readings/bellard.pdf)。下面这段话是作者对QEMU的概述。

> We present the internals of QEMU, a fast machine emulator using an original portable dynamic translator. It emulates several CPUs (x86, PowerPC, ARM and Sparc) on several hosts (x86, PowerPC, ARM, Sparc, Alpha and MIPS). QEMU supports full system emulation in which a complete and unmodiﬁed operating system is run in a virtual machine and Linux user mode emulation where a Linux process compiled for one target CPU can be run on another CPU.

在概述中，作者很清楚地将QEMU所使用的核心技术命名为"dynamic translate"，`QEMU`通过模拟不同cpu的行为直接支持全系统的仿真。在第二章中，作者给出了例子说明所谓的动态翻译技术是什么。

首先，无论QEMU想要模拟什么操作系统，首先都需要模拟操作系统所在的CPU，因为QEMU的目标是对操作系统透明的全仿真模拟。而QEMU所作的就是针对CPU的各种指令对其再做一层封装，也就是在硬件领域中人们熟知的微指令(micro operation)的概念。但是这些微指令的目标不是直接依赖硬件，而是C语言。这就是QEMU能够在不同架构上都能很好地完成仿真任务的原因之一。因为C语言悠久的历史，它在各个平台上都有极好的编译支持，从而QEMU也能享此优势。

QEMU直接会使用`dyngen`这一编译时(compile-time)工具将微指令模拟方法生成一个运行时编译工具——也就是我们熟知的解释器(interpreter)概念(原文中说的是dynamic code generator，但我觉得大家可能都倾向于称这种工具为解释器)。我列出了QEMU实现这个解释器的一些比较重要的细节：

+ 用c语言gcc的语言特性将基于主机特性的寄存器组映射到c语言变量上
+ 通过编写c语言，并将函数指针强转地址拷贝和内联常量生成中间代码
+ 解释器将分块编译代码，其中需要使用分支预测、代码缓存等技术。QEMU软件解释器的这种行为是一种典型的jit compiler行为。这种编译器行为介于传统interpreter和compiler之间。
+ 使用mmap映射内存，并使用物理地址直接索引TB(Translated Block)缓存。
+ 使用特定函数`longjmp`捕捉异常
+ 使用异步调用被动获取软件需求的硬件异常

JIT编译行为（基于论文）我将之梳理为下面这个转移图：

![](https://g.gravizo.com/svg?digraph%20G%20{%20ttb%20-%3E%20execute%20-%3E%20c;%20c%20-%3E%20evaluate%20-%3E%20f;%20f%20-%3E%20ttb%20[label=%22or%20not%22];%20f%20-%3E%20execute%20[label=%22if%20in%20hash%20table%22];%20ttb%20[label=%22translate%20target%20code%20block%22];%20c%20[label=%22check%20cond-codes%20at%20runtime%22]%20f%20[label=%22find%20possible/needed%20target%20code%20block%22]%20})

如果图片无法加载，看这个，可以在心里渲染出这个图：

```diagraph
digraph G {
   ttb -> execute -> c; // ttb到c有这样一条路径
   c -> evaluate -> f;
   f -> ttb [label="or not"]; // label是对状态转移的补充说明
   f -> execute [label="if in hash table"];
   ttb [label="translate target code block"]; // label是对节点上操作的补充说明
   c [label="check cond-codes at runtime"]
   f [label="find possible/needed target code block"]
}
```

#### 总结

QEMU使用了常规的jit编译技术将cpu指令编译为中间码再将中间码转化为主机相关(host speciﬁc)的机器码。这种技术在如jvm(企业版为Oracle的多个版本)和js v8(Google领导开发的js引擎)等大家熟知的项目中被发扬光大。代价是显见的，增加了一层包装虽然提供了封装，也是加重了运行时负担。但也可能正如作者所说的，QEMU大概寻找到了性能和优雅之间恰到好处的那个点。

> The QEMU approach seems a good compromise between performance and complexity.