## cpus.c逻辑分析

使用` rbreak cpus.c:.`给该文件打上所有断点，第一个被触发的入口断点在`qemu_init_cpu_loop`，由`main`函数调用。

这个函数是个组合函数

```c
void qemu_init_cpu_loop(void)
{
    qemu_init_sigbus();
    qemu_cond_init(&qemu_cpu_cond);
    qemu_cond_init(&qemu_pause_cond);
    qemu_mutex_init(&qemu_global_mutex);

    qemu_thread_get_self(&io_thread);
}
```

`qemu_init_sigbus`首先为`SIGBUS`信号提供了一个handler: `sigbus_handler`，该函数入口为`cpus.c:1160`。

然后调用了linux系统调用`prctl`，即`process_control`，目的是抑制进程自带的`MCE`(Memory Corruption Error)处理线程，当`SIGBUS`信号被发送时，内核会尽快通知线程处理。

初始化了两个Condition和一个Mutex，`qemu_cpu_cond`，`qemu_pause_cond`和`qemu_global_mutex`。

初始化了线程变量`io_thread`（可见`cpus.c`模块认为自身线程为`io_thread`，这个线程与主线程(`main`所启动的线程)是一个线程。

当`main`函数调用了`qemu_init_cpu_loop`以后，立马将调用`qemu_mutex_lock_iothread_impl`将`iothread`锁了起来，也即将`iothread_locked`设置为true。

接下来在初始化了一系列其他模块以后，又进入到`cpu_ticks_init`。

```c
void cpu_ticks_init(void)
{
    seqlock_init(&timers_state.vm_clock_seqlock);
    qemu_spin_init(&timers_state.vm_clock_lock);
    vmstate_register(NULL, 0, &vmstate_timers, &timers_state);
    throttle_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL_RT,
                                           cpu_throttle_timer_tick, NULL);
}
```

这里初始化了两个和计时器状态有关的变量，并将`timers_state`注册到迁移模块以备迁移。重要的是，初始化了`throttle_timer`，这个计时器我发现在QEMU Replay文档中有提及，正是所谓的`warp timer`，这个计时器用于增加`icount`变量。

如果开启了`tcg`模块，`main`会调用`qemu_tcg_configure`，默认该模块被开启。由于我们以默认参数启动，所以该函数只调用`default_mttcg_enabled`，相当于：

```c
void qemu_tcg_configure(NULL, Error **errp)
{
    if (t) {
    } else {
        mttcg_enabled = default_mttcg_enabled();
    }
}
```

这里的`mttcg`是多线程`tcg`的缩写，如果当前宿主机与虚拟机的Memory Order(所谓大端小端)一致，则开启`mttcg`。我这次Debug，返回了true，也就是多线程`tcg`被开启。

重点来了：

当初始化`cpu`实例时（参考`hw-core/machine.md`文档），实例化函数会调用`qemu_init_vcpu`。

`qemu_init_vcpu`接受一个`CPUState`结构体，并添加了一些新的参数。

最终branch到这个带有优先级的`switch`语句上，如果你添加了对应的参数，例如`enable-kvm`，那么会以不同的优先级选择虚拟CPU运行的方式。这里我什么都没做，所以以`tcg`解释器方式运行CPU，虽然它很慢，但是是在我们一开始所研究的论文中提到的，原生的QEMU运行方式。如果你已经添加了KVM选项，qemu会为CPU模拟做许多优化，当然也复杂得多。

```c
if (kvm_enabled()) {
        qemu_kvm_start_vcpu(cpu);
    } else if (hax_enabled()) {
        qemu_hax_start_vcpu(cpu);
    } else if (hvf_enabled()) {
        qemu_hvf_start_vcpu(cpu);
    } else if (tcg_enabled()) {
        qemu_tcg_init_vcpu(cpu);
    } else if (whpx_enabled()) {
        qemu_whpx_start_vcpu(cpu);
    } else {
        qemu_dummy_start_vcpu(cpu);
    }
```

终于，我们清楚为什么会新创建两个线程了——这里有两个原因：

+ 使用`tcg`方式运行CPU。

+ `mttcg`为真——否则tcg会共享主线程。

好了，`cpus.c`里还有运行`cpu`的逻辑，虽然我们创建了两个新的线程。我们先暂时屏蔽其他的线程，我们继续看看主线程是否还对`cpus.c`动了其他手脚，等看完了主线程操作`cpus`的逻辑以后我们再回到`cpu`运行逻辑上来。

我写了一个gdb函数用于只给`cpus.c`打上线程1的断点：

```gdb
define condbatthread1
b configure_icount thread 1
...
b warp_timer_state_needed thread 1
end
```

这个函数是将`info b`中的断点读出来再重写的，没找到其他方法。

下一个被执行的函数是`cpu_synchronize_all_post_init`，这个函数只有cpu执行模式为：`kvm`、`hax`、`whpx`、`hvf`时起作用，对于`tcg`模式，实际上没有任何用处。与之类似的函数还有`cpu_synchronize_all_states`、`cpu_synchronize_all_post_reset`。

接下来，终于所有的初始化完成了，我们来到`os_host_main_loop_wait`，这里是`main`里最后直接与`cpus.c`相关的部分了：

```c

static int os_host_main_loop_wait(int64_t timeout)
{
    ...
	qemu_mutex_unlock_iothread();
    replay_mutex_unlock();

    ret = qemu_poll_ns((GPollFD *)gpollfds->data, gpollfds->len, timeout);

    replay_mutex_lock();
    qemu_mutex_lock_iothread();
    ...
}
```

`main_loop`会释放`iothread`的锁，然后做一次`glib`的主事件循环，循环结束后再重新加锁——意思是...`iothread`锁的外面都是与`io`相关的代码。现在我们可以猜想一下这个锁是做什么用的了：

```c
int main() {
	什么都不做，立即执行qemu_mutex_lock_iothread
    执行与io有关的代码
    ...
    main_loop(...)
    ...
}
int main_loop(...) {
    while(!main_loop_should_exit()) {
        ...
        ret = os_host_main_loop_wait();
    }
}
static int os_host_main_loop_wait(...) {
    ...
        io事件处理完毕，释放iothread的锁，把数据交给同线程的其他代码处理
        qemu_mutex_unlock_iothread();
        ...
        qemu_mutex_lock_iothread();
        回到io逻辑代码部分，给iothread加上锁。
    ...
}
```

### 其他几个线程与`cpus.c`的关系

经过调试，`tcg`子线程的主函数`qemu_tcg_cpu_thread_fn`与`cpus.c`没有太大关系——`cpus.c`仅仅提供了一些辅助判断函数；反而与`cpu-exec.c`有很大关系。因此我打算将这一小部分的辅助判断函数放到`cpu-exec.md`分析。

如果说`tcg`子线程的主函数在`qemu_cpu_thread_fn`，还能说与`cpus.c`有点关系，那么另外两个线程，`rcu`线程和`vnc`线程就与`cpus.c`毫无交集了。`rcu`线程仅仅有过一个与之相关的部分——即`memory_region_transaction_commit`会断言`iothread_locked`的值。

我们看`iothread_locked`值的声明：

```c
static __thread bool iothread_locked = false;
```

这个值有一个`gcc`提供的属性修饰符`__thread`，这个修饰符的意思是，这个变量在每个线程中都是独立的。

因此从`cpus.c`这个文件看，5个线程将来对`cpus.c`的影响可以单独分析。

### memory, net_slirp这两个模块部分也与`cpus.c`有关。

我们还注意到几个现象，下面这些代码都会触及`cpus.c`。

`memory`部分挺好理解，就是断言`iothread`锁已经上了：

```c
qemu_mutex_iothread_locked cpus.c:1887
memory_region_transaction_commit memory.c:1056
```

```c
cpu_get_clock cpus.c:392
qemu_clock_get_ns qemu-timer.c:655
net_slirp_clock_get_ns slirp.c:174
icmp6_init ip6_icmp.c:33
ip6_init ip6_input.c:16
slirp_init slirp.c:295
net_slirp_init slirp.c:562
net_init_slirp slirp.c:1116
net_client_init1 net.c:1055
net_client_init net.c:1155
net_init_client net.c:1489
qemu_opts_foreach qemu-option.c:1170
net_init_clients net.c:1576
main vl.c:4274
    
qemu_start_warp_timer cpus.c:581
timerlist_rearm qemu-timer.c:421
timer_mod_ns qemu-timer.c:451
timer_mod qemu-timer.c:481
net_slirp_timer_mod slirp.c:194
icmp6_init ip6_icmp.c:32
ip6_init ip6_input.c:16
slirp_init slirp.c:295
net_slirp_init slirp.c:562
net_init_slirp slirp.c:1116
net_client_init1 net.c:1055
net_client_init net.c:1155
net_init_client net.c:1489
qemu_opts_foreach qemu-option.c:1170
net_init_clients net.c:1576
main vl.c:4274
    
qemu_timer_notify_cb cpus.c:980
timerlist_notify qemu-timer.c:299
timerlist_rearm qemu-timer.c:423
timer_mod_ns qemu-timer.c:451
timer_mod qemu-timer.c:481
net_slirp_timer_mod slirp.c:194
...
main vl.c:4274

...
timer_mod qemu-timer.c:481
gui_setup_refresh console.c:250
register_displaychangelistener console.c:1459
vnc_display_init vnc.c:3209
vnc_init_func vnc.c:4119
qemu_opts_foreach qemu-option.c:1170
main vl.c:4406
    


cpu_get_clock cpus.c:391
qemu_clock_get_ns qemu-timer.c:655
serial_reset serial.c:898
qemu_devices_reset reset.c:69
pc_machine_reset pc.c:2140
qemu_system_reset vl.c:1551
main vl.c:4436

cpu_get_clock cpus.c:391
qemu_clock_get_ns qemu-timer.c:655
net_slirp_clock_get_ns slirp.c:174
slirp_pollfds_poll slirp.c:526
net_slirp_poll_notify slirp.c:301
notifier_list_notify notify.c:39
main_loop_wait main-loop.c:520
main_loop vl.c:1810
main vl.c:4471
    
cpu_get_clock cpus.c:391
qemu_clock_get_ns qemu-timer.c:655
net_slirp_clock_get_ns slirp.c:174
if_start if.c:142
slirp_pollfds_poll slirp.c:687
...
main vl.c:4471
    

```

可见net_slirp对`cpus.c`的依赖源自`cpu_get_clock`和`timer_mod`。

不过更奇怪的是主线程的`cpu_get_clock`始终返回0，不知是何原因。未来将继续追踪。



### 附

设置断点的脚本：

```
define condbatthread1
b configure_icount thread 1
b cpu_disable_ticks thread 1
b cpu_enable_ticks thread 1
b cpu_get_clock thread 1
b cpu_get_icount thread 1
b cpu_get_icount_raw thread 1
b cpu_get_ticks thread 1
b cpu_icount_to_ns thread 1
b cpu_is_stopped thread 1
b cpu_remove_sync thread 1
b cpu_resume thread 1
b cpu_stop_current thread 1
b cpu_synchronize_all_post_init thread 1
b cpu_synchronize_all_post_reset thread 1
b cpu_synchronize_all_pre_loadvm thread 1
b cpu_synchronize_all_states thread 1
b cpu_throttle_active thread 1
b cpu_throttle_get_percentage thread 1
b cpu_throttle_set thread 1
b cpu_throttle_stop thread 1
b cpu_ticks_init thread 1
b cpu_update_icount thread 1
b dump_drift_info thread 1
b hw_error thread 1
b list_cpus thread 1
b pause_all_vcpus thread 1
b qemu_cpu_is_self thread 1
b qemu_cpu_kick thread 1
b qemu_cpu_kick_self thread 1
b qemu_in_vcpu_thread thread 1
b qemu_init_cpu_loop thread 1
b qemu_init_vcpu thread 1
b qemu_mutex_iothread_locked thread 1
b qemu_mutex_lock_iothread_impl thread 1
b qemu_mutex_unlock_iothread thread 1
b qemu_start_warp_timer thread 1
b qemu_tcg_configure thread 1
b qemu_timer_notify_cb thread 1
b qmp_inject_nmi thread 1
b qmp_memsave thread 1
b qmp_pmemsave thread 1
b qtest_clock_warp thread 1
b resume_all_vcpus thread 1
b run_on_cpu thread 1
b vm_prepare_start thread 1
b vm_shutdown thread 1
b vm_start thread 1
b vm_stop thread 1
b vm_stop_force_state thread 1
b adjust_timers_state_needed thread 1
b all_cpu_threads_idle thread 1
b all_vcpus_paused thread 1
b check_tcg_memory_orders_compatible thread 1
b cpu_can_run thread 1
b cpu_get_clock_locked thread 1
b cpu_get_icount_executed thread 1
b cpu_get_icount_locked thread 1
b cpu_get_icount_raw_locked thread 1
b cpu_get_ticks_locked thread 1
b cpu_handle_guest_debug thread 1
b cpu_thread_is_idle thread 1
b cpu_throttle_thread thread 1
b cpu_throttle_timer_tick thread 1
b cpu_update_icount_locked thread 1
b deal_with_unplugged_cpus thread 1
b default_mttcg_enabled thread 1
b do_nothing thread 1
b do_vm_stop thread 1
b handle_icount_deadline thread 1
b icount_adjust thread 1
b icount_adjust_rt thread 1
b icount_adjust_vm thread 1
b icount_state_needed thread 1
b icount_timer_cb thread 1
b icount_warp_rt thread 1
b kick_tcg_thread thread 1
b prepare_icount_for_run thread 1
b process_icount_data thread 1
b qemu_account_warp_timer thread 1
b qemu_cpu_kick_rr_cpus thread 1
b qemu_cpu_kick_rr_next_cpu thread 1
b qemu_cpu_kick_thread thread 1
b qemu_cpu_stop thread 1
b qemu_dummy_cpu_thread_fn thread 1
b qemu_dummy_start_vcpu thread 1
b qemu_hax_cpu_thread_fn thread 1
b qemu_hax_start_vcpu thread 1
b qemu_hvf_cpu_thread_fn thread 1
b qemu_hvf_start_vcpu thread 1
b qemu_icount_round thread 1
b qemu_init_sigbus thread 1
b qemu_kvm_cpu_thread_fn thread 1
b qemu_kvm_destroy_vcpu thread 1
b qemu_kvm_start_vcpu thread 1
b qemu_tcg_cpu_thread_fn thread 1
b qemu_tcg_destroy_vcpu thread 1
b qemu_tcg_init_vcpu thread 1
b qemu_tcg_next_kick thread 1
b qemu_tcg_rr_cpu_thread_fn thread 1
b qemu_tcg_rr_wait_io_event thread 1
b qemu_wait_io_event thread 1
b qemu_wait_io_event_common thread 1
b qemu_whpx_cpu_thread_fn thread 1
b qemu_whpx_start_vcpu thread 1
b sigbus_handler thread 1
b sigbus_reraise thread 1
b start_tcg_kick_timer thread 1
b stop_tcg_kick_timer thread 1
b tcg_cpu_exec thread 1
b tcg_get_icount_limit thread 1
b warp_timer_state_needed thread 1
end
```

