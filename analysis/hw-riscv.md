## riscv主板模拟

### boot.c

`riscv_find_and_load_firmware`: 先find option，再根据option find `firmware_filename`，再根据`firmware_filename` 调用`riscv_load_firmware`

`riscv_load_{firmware,kernel,initrd}`: 将firmware/kernel/initrd加载到模拟内存区域中。

根据`riscv_load_kernel`的调用，可以确定`hw/riscv`中一共有分别是Sifive U, Sifive E, Spike, VirtIO

### sifive_e.c

内存映射段

```C
static const struct MemmapEntry {
    hwaddr base;
    hwaddr size;
} sifive_e_memmap[] = {
    // 例如 DTIM 基地址为 0x8<<28U 区域大小为64KB
    [SIFIVE_E_DTIM] =     { 0x80000000,     0x4000 }
};

```

Machine类初始化：

+ 实例化片上系统`TYPE_RISCV_E_SOC`
+ 初始化64KB DTIM(Data Tightly-integrated Memory, or Data Tightly-Coupled Memory)，该内存区域仅用于数据存储，是Harvard架构的特征之一。
+ 初始化复位向量（`02b7 2040 8067 0002 `，`uint32_t` in little endian），到`SIFIVE_E_MROM`的基地址上。
+ load kernel代码到内存中。

SOC设备实例化：

+ 实例化多个HART（Hardware Thread）硬件线程。
+ 初始化MROM。
+ 初始化MMIO布局，包括PLIC，CLINT，AON。
+ 实例化PRCI设备。
+ 实例化GPIO设备，并将GPIO寄存器映射到MMIO内存中。
+ 将GPIO异常映射到PLIC中。
+ 初始化GPIO设备，包括UART，创建Mock QSPI/PWM（这里似乎是我们可以贡献的点）。
+ 初始化闪存设备XIP。

HART实例化：参考我们上周`target/riscv`中的cpu分析

GPIO实例化：

+ 读内存区域handler: `sifive_gpio_read`
+ 写内存区域handler:`sifive_gpio_write`，内部更新虚拟机状态`update_state`，并触发IRQ。
+ IRQ中断服务handler:`sifive_gpio_set`

PS：PLIC（Platform-Level Interrupt Controller）是一个软件实现的中断网关和路由，负责将中断源（各I/O设备）产生的IRQ发送到HART上。

### QSPI/PWM设备可能可以由我们来编写

这两个设备仍未实现，目前由`create_unimplemented_device`函数初始化了一个仅记录内存读写情况的设备。

#### QSPI(Quad SPI Flash Controller)

SPI的全称为Serial Peripheral Interface，由SiFive提出。但是在已有资料（SiFive E31文档）中并没有这个词出现。信息很少，待调研。

#### PWM(Pulse-Width Modulation)

通过配置使得读取PWM MMIO端口时可以获得数字脉冲信号（01序列），这个设备可能不太好模拟。信息很少，待调研。

#### 如何增加一个QSPI/PWM设备的实现

内存读写模拟设备的核心为下面的结构体：

```c
static const MemoryRegionOps unimp_ops = {
    .read = unimp_read, // 定义我们的read handler
    .write = unimp_write, // 定义我们的write handler
    .impl.min_access_size = 1,
    .impl.max_access_size = 8,
    .valid.min_access_size = 1,
    .valid.max_access_size = 8,
    .endianness = DEVICE_LITTLE_ENDIAN, // SIFIVE_E为little endian
};
```

思路：

+ 创建一个`TYPE_SIFIVE_E_QSPI`，并继承`TYPE_SYS_BUS_DEVICE`
+ 该类的实例方法中用`memory_region_init_io`注册`MemoryRegionOps`

#### 如何测试我们的QSPI/PWD设备

编写独立的test文件测试read/write函数。

