## Machine模块

###### `void machine_run_board_init(MachineState *machine)`

`machine_run_board_init`接受的参数为已经收集到的目标虚拟机的信息，首先它做一些边界检查。并且如果根据`machine`得到的类支持`numa_mem_supported`，那么会调用相关的`numa`初始化指令。

```c
numa_complete_configuration(machine);
if (machine->numa_state->num_nodes) {
    machine_numa_finish_cpu_init(machine);
}
```

否则，确保`machine`的类可以实例化，然后调用`machine_class->init(machine)`。

对于我的代码`x86_64`架构会首先进入`hw/i386/pc_piix.c#pc_init1`。这个函数将虚拟机的各个虚拟设备初始化，而之前追踪到的两个线程是由其`pc_init1:156`的`x86_cpus_init`创建的。

当调用`x86_cpus_init`时，会调用需要虚拟化的CPU核数次`x86_cpu_new`(`/hw/i386/x86.c#86`)，而在`x86_cpu_new`中`cpu`类的`realized`属性的`setter`，也就是`/hw/core/qdev.c`的`device_set_realized`会被触发，在这里`cpu`设备被实例化。

```c
qemu_tcg_init_vcpu cpus.c:2008
qemu_init_vcpu cpus.c:2141
x86_cpu_realizefn cpu.c:6499
device_set_realized qdev.c:876
```

