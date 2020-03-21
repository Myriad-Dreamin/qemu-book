## 分析vl.c#main的行为

author: [Myriad-Dreamin](https://github.com/Myriad-Dreamin)

---

#### 理清`qemu`启动过程

如果需要加载符号，需要在`configure`中加入`enable-debug`的选项，我们给所有的debug版的二进制文件加上`d`后缀。用`gdb`加载并运行文件：

```bash
 (qemu-venv) kamiyoru@kamiya:~/work/c/qemu/work$ sudo gdb ../bin/qemu-system-x86_64d
 ...
Reading symbols from ../bin/qemu-system-x86_64d...done.
```

我们知道任意一个C语言程序都会由`main`函数进入C程序代码，在`main`处打上一个断点然后运行。

```bash
(gdb) set args -kernel ./linux/bzImage -initrd ./linux/.config -smp 2 -gdb tcp::1234 -S
(gdb) b main
Breakpoint 1 at 0x4a9991: file vl.c, line 2829.
(gdb) run
Starting program: bin/qemu-system-x86_64d -kernel ./linux/bzImage -initrd ./linux/.config -smp 2 -gdb tcp::1234 -S
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[New Thread 0x7ffff63b8700 (LWP 17132)]

Thread 1 "qemu-system-x86" hit Breakpoint 1, main (argc=10, argv=0x7fffffffe3f8, envp=0x7fffffffe450) at vl.c:2829
2829	{
(gdb) where
#0  main (argc=10, argv=0x7fffffffe3f8, envp=0x7fffffffe450) at vl.c:2829
```

现在获得了第一个信息: `vl.c`是程序入口。

打开`vl.c`文件，从2829行开始往后大量代码是配置代码，在2962行开始parse options，这里吐槽一下QEMU一main到底的风格实在不让人喜欢。第一次调试，我们暂时直接跳过这些过程，往下看：

```bash
(gdb) b vl.c:3836
Breakpoint 2 at 0x555555a00099: file vl.c, line 3836.
(gdb) c
Continuing.

Thread 1 "qemu-system-x86" hit Breakpoint 2, main (argc=10, argv=0x7fffffffe3f8, envp=0x7fffffffe450) at vl.c:3836
3836	    loc_set_none();
```

记录一下之后的这些函数在哪些地方

```bash
(gdb) s ...
loc_set_none () at util/qemu-error.c:118
user_register_global_props () at vl.c:2824
replay_configure (opts=0x0) at replay/replay.c:298
configure_rtc() at vl.c:861
select_machine () at vl.c:2590
os_daemonize () at os-posix.c:255
rcu_disable_atfork () at util/rcu.c:341
qemu_add_exit_notifier (notify=0x5555567d3940 <qemu_unlink_pidfile_notifier>) at vl.c:2505
qemu_init_main_loop (errp=0x7fffffffe158) at util/main-loop.c:148
object_class_get_name (klass=0x555556882db0) at qom/object.c:919
object_new (typename=0x5555567ff4f0 "pc-i440fx-4.2-machine") at qom/object.c:644
cpu_exec_init_all () at exec.c:3404
trace_init_backends () at trace/control.c:315
trace_init_file (file=0x0) at trace/control.c:265
os_find_datadir () at os-posix.c:94
qemu_add_data_dir (path=0x555556929c20 "pc-bios") at vl.c:2131
qemu_plugin_load_list (head=0x7fffffffe2e0) at include/qemu/plugin.h:46
qemu_display_early_init (opts=0x5555567d36c0 <dpy>) at ui/console.c:2306
qemu_console_early_init () at ui/console.c:2394
page_size_init () at exec.c:3952
socket_init () at util/osdep.c:504
qemu_semihosting_connect_chardevs () at stubs/semihost.c:71
configure_blockdev (bdo_queue=...snapshot=0) at vl.c:1191
configure_accelerator (ms=0x555556916630, progname=0x7fffffffe6bb "bin/qemu-system-x86_64d") at accel/accel.c:79
migration_object_init () at migration/migration.c:138
qemu_spice_init () at include/ui/qemu-spice.h:88
cpu_ticks_init () at cpus.c:863
qemu_tcg_configure (opts=0x0, errp=0x5555567df990 <error_fatal>) at cpus.c:213
net_init_clients (errp=0x7fffffffe160) at net/net.c:1563
tpm_init () at tpm.c:156
blk_mig_init () at migration/block.c:1029
ram_mig_init () at migration/ram.c:4593
dirty_bitmap_mig_init () at migration/block-dirty-bitmap.c:731
replay_checkpoint (checkpoint=CHECKPOINT_INIT) at replay/replay.c:178
qdev_machine_init () at qdev-monitor.c:924
main_loop () at vl.c:1806
audio_init_audiodevs () at audio/audio.c:2054
machine_run_board_init (machine=0x555556916630) at hw/core/machine.c:1103

[New Thread 0x7ffff59d5700 (LWP 21732)]
[New Thread 0x7ffff51d4700 (LWP 21733)]

realtime_init () at vl.c:1255
soundhw_init () at hw/audio/soundhw.c:134
igd_gfx_passthru () at vl.c:1274
rom_set_order_override (order=110) at hw/core/loader.c:1200
cpu_synchronize_all_post_init () at cpus.c:1075
rom_reset_order_override () at hw/core/loader.c:1207
drive_check_orphaned () at blockdev.c:246
init_displaystate () at ui/console.c:1856
qemu_display_init (ds=0x55555704e770, opts=0x5555567d36c0 <dpy>) at ui/console.c:2325
os_setup_signal_handling () at os-posix.c:72
> 4406行
[New Thread 0x7fffe7fff700 (LWP 21841)]
VNC server running on 127.0.0.1:5900
qdev_machine_creation_done () at hw/core/qdev.c:378
qemu_register_reset (func=0x555555a71d1d <qbus_reset_all_fn>, opaque=0x55555691e080) at hw/core/reset.c:43
qemu_run_machine_init_done_notifiers () at vl.c:2545
replay_start () at replay/replay.c:340
qemu_system_reset (reason=SHUTDOWN_CAUSE_NONE) at vl.c:1546
register_global_state () at migration/global_state.c:145
accel_setup_post (ms=0x555556916630) at accel/accel.c:135
os_setup_post () at os-posix.c:304
main_loop () at vl.c:1806
> 4471行以后所有的代码都是处理关闭的代码
```

清理一下数据：

| 函数名  | 位置 |
| ------ | ---- |
|`loc_set_none`|`util/qemu-error.c:118`|
|`user_register_global_props`|`vl.c:2824`|
|`replay_configure`|`replay/replay.c:298`|
|`configure_rtc`|`vl.c:861`|
|`select_machine`|`vl.c:2590`|
|`os_daemonize`|`os-posix.c:255`|
|`rcu_disable_atfork`|`util/rcu.c:341`|
|`qemu_add_exit_notifier`|`vl.c:2505`|
|`qemu_init_main_loop`|`util/main-loop.c:148`|
|`object_class_get_name`|`qom/object.c:919`|
|`object_new`|`qom/object.c:644`|
|`cpu_exec_init_all`|`exec.c:3404`|
|`trace_init_backends`|`trace/control.c:315`|
|`trace_init_file`|`trace/control.c:265`|
|`os_find_datadir`|`os-posix.c:94`|
|`qemu_add_data_dir`|`vl.c:2131`|
|`qemu_plugin_load_list`|`include/qemu/plugin.h:46`|
|`qemu_display_early_init`|`ui/console.c:2306`|
|`qemu_console_early_init`|`ui/console.c:2394`|
|`page_size_init`|`exec.c:3952`|
|`socket_init`|`util/osdep.c:504`|
|`qemu_semihosting_connect_chardevs`|`stubs/semihost.c:71`|
|`configure_blockdev`|`vl.c:1191`|
|`configure_accelerator`|`accel/accel.c:79`|
|`migration_object_init`|`migration/migration.c:138`|
|`qemu_spice_init`|`include/ui/qemu-spice.h:88`|
|`cpu_ticks_init`|`cpus.c:863`|
|`qemu_tcg_configure`|`cpus.c:213`|
|`net_init_clients`|`net/net.c:1563`|
|`tpm_init`|`tpm.c:156`|
|`blk_mig_init`|`migration/block.c:1029`|
|`ram_mig_init`|`migration/ram.c:4593`|
|`dirty_bitmap_mig_init`|`migration/block-dirty-bitmap.c:731`|
|`replay_checkpoint`|`replay/replay.c:178`|
|`qdev_machine_init`|`qdev-monitor.c:924`|
|`main_loop`|`vl.c:1806`|
|`audio_init_audiodevs`|`audio/audio.c:2054`|
|`machine_run_board_init`|`hw/core/machine.c:1103`|
|`realtime_init`|`vl.c:1255`|
|`soundhw_init`|`hw/audio/soundhw.c:134`|
|`igd_gfx_passthru`|`vl.c:1274`|
|`rom_set_order_override`|`hw/core/loader.c:1200`|
|`cpu_synchronize_all_post_init`|`cpus.c:1075`|
|`rom_reset_order_override`|`hw/core/loader.c:1207`|
|`drive_check_orphaned`|`blockdev.c:246`|
|`init_displaystate`|`ui/console.c:1856`|
|`qemu_display_init`|`ui/console.c:2325`|
|`os_setup_signal_handling`|`os-posix.c:72`|
|`qdev_machine_creation_done`|`hw/core/qdev.c:378`|
|`qemu_register_reset`|`hw/core/reset.c:43`|
|`qemu_run_machine_init_done_notifiers`|`vl.c:2545`|
|`replay_start`|`replay/replay.c:340`|
|`qemu_system_reset`|`vl.c:1546`|
|`register_global_state`|`migration/global_state.c:145`|
|`accel_setup_post`|`accel/accel.c:135`|
|`os_setup_post`|`os-posix.c:304`|
|`main_loop`|`vl.c:1806`|

#### 分析

###### 启动了3个线程

一共启动了3个线程：
在`machine_run_board_init`启动了两个，在`vnc_init_func`(没有追踪到，在4406行)启动了一个。这两个函数应该优先分析。

###### 一些模块必然会被初始化

这些函数在main函数中被`init`的顺序为：

| 函数名  | 位置 |
| ------ | ---- |
|`qemu_init_main_loop`|`util/main-loop.c:148`|
|`cpu_exec_init_all`|`exec.c:3404`|
|`trace_init_backends`|`trace/control.c:315`|
|`trace_init_file`|`trace/control.c:265`|
|`qemu_display_early_init`|`ui/console.c:2306`|
|`qemu_console_early_init`|`ui/console.c:2394`|
|`page_size_init`|`exec.c:3952`|
|`socket_init`|`util/osdep.c:504`|
|`migration_object_init`|`migration/migration.c:138`|
|`qemu_spice_init`|`include/ui/qemu-spice.h:88`|
|`cpu_ticks_init`|`cpus.c:863`|
|`net_init_clients`|`net/net.c:1563`|
|`tpm_init`|`tpm.c:156`|
|`blk_mig_init`|`migration/block.c:1029`|
|`ram_mig_init`|`migration/ram.c:4593`|
|`dirty_bitmap_mig_init`|`migration/block-dirty-bitmap.c:731`|
|`qdev_machine_init`|`qdev-monitor.c:924`|
|`audio_init_audiodevs`|`audio/audio.c:2054`|
|`machine_run_board_init`|`hw/core/machine.c:1103`|
|`realtime_init`|`vl.c:1255`|
|`soundhw_init`|`hw/audio/soundhw.c:134`|
|`cpu_synchronize_all_post_init`|`cpus.c:1075`|
|`qemu_display_init`|`ui/console.c:2325`|
|`qemu_run_machine_init_done_notifiers`|`vl.c:2545`|

搞清楚是哪些模块被初始化很重要。

在这次调试中，有一些函数没有被追踪到，也需要纳入考虑中：

+ `qemu_init_cpu_list`: 2870行
+ `qemu_init_cpu_loop`: 2871行
+ `qemu_init_exec_dir`: 2876行
+ `module_call_init`: 2878行
+ `runstate_init`: 2914行
+ `precopy_infrastructure_init`: 2915行
+ `postcopy_infrastructure_init`: 2916行
+ `monitor_init_globals`: 2917行
+ `qcrypto_init`: 2919行
+ `QTAILQ_INIT`: 2924行
+ `bdrv_init_with_whitelist`: 2932行


###### 多次出现replay

这些函数在main函数中出现的顺序为：

| 函数名  | 位置 |
| ------ | ---- |
|`replay_configure(icount_opts)`|`replay/replay.c:298`|
|`replay_checkpoint(CHECKPOINT_INIT)`|`replay/replay.c:178`|
|`replay_start()`|`replay/replay.c:340`|
|`replay_checkpoint(CHECKPOINT_RESET)`|`replay/replay.c:178`|

还有一些以`replay_`开头未被追踪的函数：`replay_add_blocker`(3032行)，`replay_finish`(4494行)。

还有一些被包装在非`main`函数但是在`vl.c`中的`replay`相关函数，例如1494行和1505行的两个函数`qemu_reset_requested`和`qemu_suspend_requested`是`replay`的包装，也应该被分析到。


###### main_loop

在第3866行中被调用了初始化`qemu_init_main_loop`，分别在4341行和4471行中调用了`main_loop`，并且在4471行中`main`函数进入轮询模式直到`main_loop`终止。

| 函数名  | 位置 |
| ------ | ---- |
|`qemu_init_main_loop`|`util/main-loop.c:148`|
|`main_loop`|`vl.c:1806`|

###### options的辅助函数

一些比较常用的options辅助函数的参数应该也纳入第一轮分析中，例如`qemu_opts_foreach`。

#### 其他未追踪到的可能重要的函数

+ `qemu_mutex_lock_iothread`: 2872行
+ `qemu_run_exit_notifiers`: 2875行
+ `os_setup_early_signal_handling`: 2925行
+ `drive_add`: 2967行
+ `replay_add_blocker`: 3032行
+ `qemu_add_data_dir`: 3197行
+ `select_watchdog_action`: 3415行
+ `acpi_table_add`: 3467行
+ `smbios_entry_add`: 3475行
+ `runstate_set`: 3655行
+ `qsp_enable`: 3819行

#### 代码统计

第一轮可能分析的模块代码统计如下：

| 模块         | 有效代码行数 |
| ------------ | ------------ |
| `replay`     | 1334         |
| `exec.c`     | 2960         |
| `migration`  | 16668        |
| `cpus.c`     | 1699         |
| `hw/core`    | 8027         |
| `util`       | 19245        |
| `os-posix.c` | 272          |
| `vl.c`       | 3724         |

根据代码量和重要程度合理控制阅读强度吧:)

#### 注:

`MACHINE`是一个宏，用于检查`object_new`的结果。

```c
#define MACHINE(obj) \
    OBJECT_CHECK(MachineState, (obj), TYPE_MACHINE)
#define MACHINE_GET_CLASS(obj) \
    OBJECT_GET_CLASS(MachineClass, (obj), TYPE_MACHINE)
#define MACHINE_CLASS(klass) \
    OBJECT_CLASS_CHECK(MachineClass, (klass), TYPE_MACHINE)
```
