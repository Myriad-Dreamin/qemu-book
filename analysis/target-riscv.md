+ 尝试编译源码验证源码完整性。
+ 梳理`target/riscv`文件中CPU的初始化，以及其他函数之间的关系与它们的功能。

首先检验源码是否可编译：

```bash
../configure --enable-debug --target-list=riscv64-linux-user,riscv64-softmmu
make
```

全局常量说明：

+ `cpu.c`:
    + `char[] riscv_exts`: 每个下标字符对应一个扩展。例如`riscv_exts[0]`（`'I'`）对应RVXXI基本指令集。这个字符和下表的关系是乱的(顺序为`IEMAFDQCLBJTPVNSUHKORWXYZG`)不知道有何用意
    + `char*[] riscv_int_regnames`每项描述对应标号的一个整数寄存器，例如`riscv_int_regnames[0]`（`x0/zero`）对应零寄存器。
    + `char*[] riscv_fpr_regnames`每项描述对应标号的一个浮点寄存器。
    + `char*[] riscv_excp_names`每项描述对应标号的异常文字说明
    + `char*[] riscv_intr_names`，用途不明

## `cpu.c`内容梳理

### CPU实例化和初始化

这部分是对李鸿政分析riscv实例化cpu函数的补充。

代码入口，和我们在[cpus.c逻辑分析](analysis/cpus.md)中分析x86 vcpu实例化的过程类比，riscv-cpu的实例化对应函数为`target/riscv/cpu.c:riscv_cpu_realize`，当`realized`属性被设置为true时该hook被触发。例如：

```c

// riscv hart machine
static void riscv_hart_realize(RISCVHartArrayState *s, int idx,
                               char *cpu_type, Error **errp)
{
    ...
    object_property_set_bool(OBJECT(&s->harts[idx]), true,
                             "realized", &err);
    ...
}

```

这里每个`harts[idx]`都是一个riscv vcpu实例。

具体是不是真的如此我就不追踪了，为了节省时间，我们现在就假设实例化函数仅有这个。

`riscv_cpu_realize`首先会调用的是`cpu_exec_realizefn`，是不是很熟悉，这个函数会完成vcpu的基本初始化，包括tcg线程初始化等。

然后，初始化一系列的cpu属性。如：

+ `set_priv_version`: 指定特权版本。
+ `set_resetvec`: 指定复位向量地址。
+ `set_feature`: 指定运行方式
+ `set_misa`: 指定支持的指令架构，MISA即多指令集架构(multiple instruction sets architecture)

最后

+ 执行`qemu_init_vcpu`，完成最终的vcpu初始化。

+ 执行`cpu_reset`，重置vcpu状态。

+ 执行父类实例化函数`parent_realize`，这个函数源于传入的`DeviceClass.realize`。

  ```c
  // riscv_cpu_class_init
  device_class_set_parent_realize(dc, riscv_cpu_realize, &mcc->parent_realize);
  // save parent_realize, replace dev->realize by riscv_cpu_realize
  void device_class_set_parent_realize(...) {
      *parent_realize = dc->realize;
      dc->realize = dev_realize;
  }
  ```

### CPU tcg初始化

在上一节，我们谈到了`cpu_exec_realizefn`，我们知道当我们使用默认的模拟方法时，该函数会调用CPUClass的`tcg_initialize`方法。

```c
    if (tcg_enabled() && !tcg_target_initialized) {
        tcg_target_initialized = true;
        cc->tcg_initialize();
    }
```

对于`RISCVCPUClass`，该函数实际对应于`riscv_translate_init`

该函数初始化了`translate.c`中的一些全局变量，如：`cpu_gpr`、`cpu_fpr`、`cpu_pc`、`load_res`、`load_val`

### 其他CPU类方法

+ `reset = riscv_cpu_reset`

+ `class_by_name = riscv_cpu_class_by_name`
+ `has_work = riscv_cpu_has_work`
+ `do_interrupt = riscv_cpu_do_interrupt`
+ `cpu_exec_interrupt = riscv_cpu_exec_interrupt`
+ `dump_state = riscv_cpu_dump_state`
+ `set_pc = riscv_cpu_set_pc`
+ `synchronize_from_tb = riscv_cpu_synchronize_from_tb`
+ `gdb_read_register = riscv_cpu_gdb_read_register`
+ `gdb_write_register = riscv_cpu_gdb_write_register`
+ `disas_set_info = riscv_cpu_disas_set_info`

### 其他`cpu.c`中未提到的函数

`riscv_xxx_cpu_init`是RISCV CPU类`TYPE_RISCV_CPU`的子类`TYPE_RISCV_CPU_XXX`的初始化方法。例如：

```c

#define DEFINE_CPU(type_name, initfn)      \
    {                                      \
        .name = type_name,                 \
        .parent = TYPE_RISCV_CPU,          \
        .instance_init = initfn            \
    }


    DEFINE_CPU(TYPE_RISCV_CPU_ANY,              riscv_any_cpu_init),

```

剩余还有一个函数`restore_state_to_opc`，这个函数似乎只在`accel/tcg/translate.all.c:cpu_restore_state_from_tb`中被调用

## `cpu_helper.c`内容梳理

`cpu_helper.c`较为复杂，函数的内部依赖如下：

`riscv_cpu_exec_interrupt`  -> `riscv_cpu_local_irq_pending` 

`riscv_cpu_local_irq_pending` -> `riscv_cpu_virt_enabled`, `riscv_cpu_set_force_hs_excep`

`riscv_cpu_exec_interrupt` -> `riscv_cpu_local_irq_pending`, `riscv_cpu_do_interrupt`

`riscv_cpu_fp_enabled` -> `riscv_cpu_virt_enabled`

`riscv_cpu_swap_hypervisor_regs` -> `riscv_cpu_virt_enabled`, `riscv_has_ext`

`riscv_cpu_virt_enabled` -> `riscv_has_ext`

`riscv_cpu_set_virt_enabled` -> `riscv_has_ext`

`riscv_cpu_force_hs_excep_enabled` -> `riscv_has_ext`

`riscv_cpu_set_force_hs_excep` -> `riscv_has_ext`

`get_physical_address` -> `riscv_feature`, `riscv_has_ext`, `pmp_hart_has_privs`, `raise_mmu_exception`, `riscv_raise_exception`, `get_physical_address`

`riscv_cpu_get_phys_page_debug` -> `riscv_cpu_mmu_index`, `get_physical_address`, `riscv_cpu_virt_enabled`

`riscv_cpu_do_transaction_failed` -> `riscv_raise_exception`

`riscv_cpu_do_unaligned_access` -> `riscv_raise_exception`

`riscv_cpu_tlb_fill` -> `riscv_has_ext`, `riscv_cpu_virt_enabled`, `get_physical_address`, `riscv_feature`, `pmp_hart_has_privs`

`riscv_cpu_do_interrupt` -> `riscv_cpu_force_hs_excep_enabled`, `riscv_cpu_virt_enabled`, `riscv_cpu_swap_hypervisor_regs`, `riscv_cpu_set_virt_enabled`, `riscv_cpu_set_force_hs_excep`, `riscv_cpu_set_mode`

根据内部依赖，分类如下：

辅助函数：

+ `riscv_cpu_mmu_index`

+ `riscv_cpu_local_irq_pending`

+ `riscv_cpu_swap_hypervisor_regs`

+ `get_physical_address`

+ `raise_mmu_exception`

getter/setter:

+ `riscv_cpu_fp_enabled`

+ `riscv_cpu_virt_enabled`

+ `riscv_cpu_set_virt_enabled`

+ `riscv_cpu_force_hs_excep_enabled`

+ `riscv_cpu_set_force_hs_excep`

+ `riscv_cpu_claim_interrupts`

+ `riscv_cpu_set_rdtime_fn`

+ `riscv_cpu_set_mode`

+ `riscv_cpu_update_mip`

类方法：

+ `riscv_cpu_exec_interrupt`

+ `riscv_cpu_get_phys_page_debug`

+ `riscv_cpu_do_transaction_failed`

+ `riscv_cpu_do_unaligned_access`

+ `riscv_cpu_tlb_fill`

## `fpu_helper.c`内容梳理

getter/setter:

+ `riscv_cpu_get_fflags`

+ `riscv_cpu_set_fflags`

剩余的`helper_**`用于帮助`translate.c`

## `op_helper.c`内容梳理

辅助函数：

+ `riscv_raise_exception`

剩余的`helper_**`用于帮助`translate.c`

## `target/riscv`中其他文件的大致作用

+ `csr.c`: CSR是riscv特权架构中规定的各种控制状态寄存器（Control and Status Register），这个文件中的函数都只与读写CSR相关。所以，我们需要做的只是读特权架构手册，读完自然就能知道读写这些寄存器的意思。
+ `monitor.c`: 与显示器设备相关，我们不关心这些函数的作用。
+ `pmp.c`: 物理内存保护（Physical Memory Protection）相关，我们暂时不关心这些函数的作用。