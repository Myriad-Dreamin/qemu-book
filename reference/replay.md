

## QEMU Feature: 确定性重放

reference: [replay document on QEMU-master](https://github.com/qemu/qemu/blob/master/docs/replay.txt)

---

Record/replay
-------------

Record/replay functions are used for the deterministic replay of qemu execution. Execution recording writes a non-deterministic events log, which can be later used for replaying the execution anywhere and for unlimited number of times.It also supports checkpointing for faster rewind to the specific replay moment. Execution replaying reads the log and replays all non-deterministic events including external input, hardware clocks, and interrupts.

记录/重放(record/replay)函数被用作QEMU执行指令的确定性重放。每一次指令的执行都会将操作写入一个不确定性的事件日志中，用作将来的无限次重放操作。QEMU还支持设置checkpoint来快速回溯到指定的系统状态上。当指令重放时，QEMU会读取日志并将所有的包括外部输入、硬件时钟、终端等不确定性事件都重新模拟一边。

Deterministic replay has the following features:
 * Deterministically replays whole system execution and all contents of
   the memory, state of the hardware devices, clocks, and screen of the VM.
 * Writes execution log into the file for later replaying for multiple times
   on different machines.
 * Supports i386, x86_64, and Arm hardware platforms.
 * Performs deterministic replay of all operations with keyboard and mouse
   input devices.

确定性重放包括以下几个特点：

+ 确定性地将系统的所有状态重放，包括执行过的指令，每个指令执行时虚拟机所处的内存、硬件、时钟、屏幕。
+ 在实时模拟时，将执行日志写入文件以备将来可以在不同的机器上多次重放。
+ 支持 i386, x86_64和Arm硬件平台。
+ 键盘、鼠标的操作也能确定性重放。

Usage of the record/replay:
 * First, record the execution with the following command line:
    qemu-system-i386 \
     -icount shift=7,rr=record,rrfile=replay.bin \
     -drive file=disk.qcow2,if=none,snapshot,id=img-direct \
     -drive driver=blkreplay,if=none,image=img-direct,id=img-blkreplay \
     -device ide-hd,drive=img-blkreplay \
     -netdev user,id=net1 -device rtl8139,netdev=net1 \
     -object filter-replay,id=replay,netdev=net1
 * After recording, you can replay it by using another command line:
    qemu-system-i386 \
     -icount shift=7,rr=replay,rrfile=replay.bin \
     -drive file=disk.qcow2,if=none,snapshot,id=img-direct \
     -drive driver=blkreplay,if=none,image=img-direct,id=img-blkreplay \
     -device ide-hd,drive=img-blkreplay \
     -netdev user,id=net1 -device rtl8139,netdev=net1 \
     -object filter-replay,id=replay,netdev=net1
   The only difference with recording is changing the rr option
   from record to replay.
 * Block device images are not actually changed in the recording mode, because all of the changes are written to the temporary overlay file. This behavior is enabled by using blkreplay driver. It should be usedfor every enabled block device, as described in 'Block devices' section.
 * '-net none' option should be specified when network is not used, because QEMU adds network card by default. When network is needed, it should be configured explicitly with replay filter, as describedin 'Network devices' section.
 * Interaction with audio devices and serial ports are recorded and replayed automatically when such devices are enabled.

记录/重放的使用方法：

+ (省略)

Academic papers with description of deterministic replay implementation:
http://www.computer.org/csdl/proceedings/csmr/2012/4666/00/4666a553-abs.html
http://dl.acm.org/citation.cfm?id=2786805.2803179

确定性重放的学术论文参见

+ http://www.computer.org/csdl/proceedings/csmr/2012/4666/00/4666a553-abs.html
+ http://dl.acm.org/citation.cfm?id=2786805.2803179

Modifications of qemu include:
 * wrappers for clock and time functions to save their return values in the log
 * saving different asynchronous events (e.g. system shutdown) into the log
 * synchronization of the bottom halves execution
 * synchronization of the threads from thread pool
 * recording/replaying user input (mouse, keyboard, and microphone)
 * adding internal checkpoints for cpu and io synchronization
 * network filter for recording and replaying the packets
 * block driver for making block layer deterministic
 * serial port input record and replay
 * recording of random numbers obtained from the external sources

QEMU对其做出的改变有：

+ 修改了时间相关函数以利于保存其返回值到日志中
+ 将不同的异步事件(例如关机)写到日志中
+ bottom harves的同步方法([bottom harves介绍][BH])
+ 对来自线程池的线程的同步方法。
+ 记录/重放用户输入 (mouse, keyboard, and microphone)
+ 为CPU/IO同步增加外部checkpoint
+ 添加了用于重放网络报文的network filter
+ 添加了为了使block layer确定的块设备
+ 记录/重放串行接口输入
+ 记录外源提供的随机数

Locking and thread synchronisation
----------------------------------

Previously the synchronisation of the main thread and the vCPU thread was ensured by the holding of the BQL. However the trend has been to reduce the time the BQL was held across the system including under TCG system emulation. As it is important that batches of events are kept in sequence (e.g. expiring timers and checkpoints in the main thread while instruction checkpoints are written by the vCPU thread) we need another lock to keep things in lock-step. This role is now handled by the **replay_mutex_lock**. It used to be held only for each event being written but now it is held for a whole execution period. This results in a deterministic ping-pong between the two main threads.

As the BQL is now a finer grained lock than the replay_lock it is almost certainly a bug, and a source of deadlocks, to take the replay_mutex_lock while the BQL is held. This is enforced by an assert. While the unlocks are usually in the reverse order, this is not necessary; you can drop the replay_lock while holding the BQL, without doing a more complicated unlock_iothread/replay_unlock/lock_iothread sequence.

Non-deterministic events
------------------------

Our record/replay system is based on saving and replaying non-deterministic events (e.g. keyboard input) and simulating deterministic ones (e.g. reading from HDD or memory of the VM). Saving only non-deterministic events makes log file smaller and simulation faster.

The following non-deterministic data from peripheral devices is saved into the log: mouse and keyboard input, network packets, audio controller input, serial port input, and hardware clocks (they are non-deterministic too, because their values are taken from the host machine). Inputs from simulated hardware, memory of VM, software interrupts, and execution of instructions are not saved into the log, because they are deterministic and can be replayed by simulating the behavior of virtual machine starting from initial state.

mouse and keyboard input, network packets, audio controller input,serial port input, and hardware clocks会被记录到重放日志中，而模拟输入包括hardware, memory of VM, software interrupts, and execution of instructions不会。

We had to solve three tasks to implement deterministic replay: recording non-deterministic events, replaying non-deterministic events, and checking that there is no divergence between record and replay modes.

We changed several parts of QEMU to make event log recording and replaying. Devices' models that have non-deterministic input from external devices were changed to write every external event into the execution log immediately. E.g. network packets are written into the log when they arrive into the virtual network adapter.

All non-deterministic events are coming from these devices. But to replay them we need to know at which moments they occur. We specify these moments by counting the number of instructions executed between every pair of consecutive events.

Instruction counting
--------------------

QEMU should work in icount mode to use record/replay feature. icount was designed to allow deterministic execution in absence of external inputs of the virtual machine. We also use icount to control the occurrence of the non-deterministic events. The number of instructions elapsed from the last event is written to the log while recording the execution. In replay mode we can predict when to inject that event using the instruction counter.

Timers
------

Timers are used to execute callbacks from different subsystems of QEMU at the specified moments of time. There are several kinds of timers:
 * Real time clock. Based on host time and used only for callbacks that do not change the virtual machine state. For this reason real time clock and timers does not affect deterministic replay at all.
 * Virtual clock. These timers run only during the emulation. In icount mode virtual clock value is calculated using executed instructions counter. That is why it is completely deterministic and does not have to be recorded.
 * Host clock. This clock is used by device models that simulate real time sources (e.g. real time clock chip). Host clock is the one of the sources of non-determinism. Host clock read operations should be logged to make the execution deterministic.
 * Virtual real time clock. This clock is similar to real time clock but it is used only for increasing virtual clock while virtual machine is sleeping. Due to its nature it is also non-deterministic as the host clock and has to be logged too.

Checkpoints
-----------

Replaying of the execution of virtual machine is bound by sources of non-determinism. These are inputs from clock and peripheral devices, and QEMU thread scheduling. Thread scheduling affect on processing events from timers, asynchronous input-output, and bottom halves.

Invocations of timers are coupled with clock reads and changing the state of the virtual machine. Reads produce non-deterministic data taken from host clock. And VM state changes should preserve their order. Their relative order in replay mode must replicate the order of callbacks in record mode. To preserve this order we use checkpoints. When a specific clock is processed in record mode we save to the log special "checkpoint" event. Checkpoints here do not refer to virtual machine snapshots. They are just record/replay events used for synchronization.

定时器的相关调用往往伴随着读取事件和改变虚拟机状态。读取系统时钟会产生不确定性数据。虚拟机在做状态转移时应该保留这些调用和读取的顺序。在重放时，它们的顺序也应该被完美复制。为了保持这个顺序，完美采用了checkpoint。当某一个时钟在记录模式被处理完毕后完美就会在日志中记录一个特殊的checkpoint事件。这里检查点与虚拟机快照无关。他们仅仅是为了方便记录/重放事件的同步。

QEMU in replay mode will try to invoke timers processing in random moment of time. That's why we do not process a group of timers until the checkpoint event will be read from the log. Such an event allows synchronizing CPU execution and timer events.

Two other checkpoints govern the "warping" of the virtual clock. While the virtual machine is idle, the virtual clock increments at 1 ns per *real time* nanosecond.  This is done by setting up a timer (called the warp timer) on the virtual real time clock, so that the timer fires at the next deadline of the virtual clock; the virtual clock is then incremented (which is called "warping" the virtual clock) as soon as the timer fires or the CPUs need to go out of the idle state. Two functions are used for this purpose; because these actions change virtual machine state and must be deterministic, each of them creates a checkpoint.  qemu_start_warp_timer checks if the CPUs are idle and if so starts accounting real time to virtual clock.  qemu_account_warp_timer is called when the CPUs get an interrupt or when the warp timer fires, and it warps the virtual clock by the amount of real time that has passed since qemu_start_warp_timer.

还有很多的检查点管理虚拟时钟的"扭曲"(warping)。当虚拟机空闲的时候，虚拟时钟每现实中的一纳秒增加一纳秒，我们通过在虚拟实时时钟上设置一个warp定时器来完成这个任务，使得定时器在虚拟时钟下一次超时前被触发；虚拟时钟会在定时器触发前或CPU需要脱离闲置状态时立即增加；为此我们编写了两个函数。因为这些操作改变了虚拟机状态并且约定是确定的，所以每次触发都创建了一个检查点。`qemu_start_warp_timer`检查CPU是否闲置，如果是那么马上开始记录将现实时间加和虚拟时钟的差值（这里不太确定？）。`qemu_account_warp_timer`会在CPU被中断或者warp定时器被触发时被调用，并且它会根据`qemu_start_warp_timer`记录的时间将使虚拟时钟跳变相应的次数。

Bottom halves
-------------

Disk I/O events are completely deterministic in our model, because in both record and replay modes we start virtual machine from the same disk state. But callbacks that virtual disk controller uses for reading and writing the disk may occur at different moments of time in record and replay modes.

Reading and writing requests are created by CPU thread of QEMU. Later these requests proceed to block layer which creates "bottom halves". Bottom halves consist of callback and its parameters. They are processed when main loop locks the global mutex. These locks are not synchronized with replaying process because main loop also processes the events that do not affect the virtual machine state (like user interaction with monitor).

That is why we had to implement saving and replaying bottom halves callbacks synchronously to the CPU execution. When the callback is about to execute it is added to the queue in the replay module. This queue is written to the log when its callbacks are executed. In replay mode callbacks are not processed until the corresponding event is read from the events log file.

Sometimes the block layer uses asynchronous callbacks for its internal purposes (like reading or writing VM snapshots or disk image cluster tables). In this case bottom halves are not marked as "replayable" and do not saved into the log.

Block devices
-------------

Block devices record/replay module intercepts calls of bdrv coroutine functions at the top of block drivers stack. To record and replay block operations the drive must be configured as following:
 -drive file=disk.qcow2,if=none,snapshot,id=img-direct
 -drive driver=blkreplay,if=none,image=img-direct,id=img-blkreplay
 -device ide-hd,drive=img-blkreplay

blkreplay driver should be inserted between disk image and virtual driver controller. Therefore all disk requests may be recorded and replayed.

All block completion operations are added to the queue in the coroutines. Queue is flushed at checkpoints and information about processed requests is recorded to the log. In replay phase the queue is matched with events read from the log. Therefore block devices requests are processed deterministically.

Snapshotting
------------

New VM snapshots may be created in replay mode. They can be used later to recover the desired VM state. All VM states created in replay mode are associated with the moment of time in the replay scenario. After recovering the VM state replay will start from that position.

Default starting snapshot name may be specified with icount field rrsnapshot as follows:
 -icount shift=7,rr=record,rrfile=replay.bin,rrsnapshot=snapshot_name

This snapshot is created at start of recording and restored at start of replaying. It also can be loaded while replaying to roll back the execution.

'snapshot' flag of the disk image must be removed to save the snapshots in the overlay (or original image) instead of using the temporary overlay.
 -drive file=disk.ovl,if=none,id=img-direct
 -drive driver=blkreplay,if=none,image=img-direct,id=img-blkreplay
 -device ide-hd,drive=img-blkreplay

Use QEMU monitor to create additional snapshots. 'savevm <name>' command created the snapshot and 'loadvm <name>' restores it. To prevent corruption of the original disk image, use overlay files linked to the original images. Therefore all new snapshots (including the starting one) will be saved in overlays and the original image remains unchanged.

Network devices
---------------

Record and replay for network interactions is performed with the network filter. Each backend must have its own instance of the replay filter as follows:
 -netdev user,id=net1 -device rtl8139,netdev=net1
 -object filter-replay,id=replay,netdev=net1

Replay network filter is used to record and replay network packets. While recording the virtual machine this filter puts all packets coming from the outer world into the log. In replay mode packets from the log are injected into the network device. All interactions with network backend in replay mode are disabled.

Audio devices
-------------

Audio data is recorded and replay automatically. The command line for recording and replaying must contain identical specifications of audio hardware, e.g.:
 -soundhw ac97

Serial ports
------------

Serial ports input is recorded and replay automatically. The command lines for recording and replaying must contain identical number of ports in record and replay modes, but their backends may differ.
E.g., '-serial stdio' in record mode, and '-serial null' in replay mode.

Replay log format
-----------------

Record/replay log consists of the header and the sequence of execution events. The header includes 4-byte replay version id and 8-byte reserved field. Version is updated every time replay log format changes to prevent using replay log created by another build of qemu.

记录/重放日志由头部和一系列的执行事件组成。日志头部包括一个4字节的重放版本ID和一个8字节的保留域。每次日志格式改变时版本ID都会更新，这样这些日志才能被不同的QEMU版本所使用。

The sequence of the events describes virtual machine state changes. It includes all non-deterministic inputs of VM, synchronization marks and instruction counts used to correctly inject inputs at replay.

事件流描述了虚拟机状态。这些事件记录了虚拟机所有的不确定输入、通不过标记和重放时注入这些输入之间所需要间隔的指令条数。

Synchronization marks (checkpoints) are used for synchronizing qemu threads that perform operations with virtual hardware. These operations may change system's state (e.g., change some register or generate interrupt) and therefore should execute synchronously with CPU thread.

同步记号（也就是checkpoint）被用于同步操作虚拟机的QEMU线程。线程执行的这些操作可能会改变系统状态（比如改变某个寄存器或者产生终端），所以需要与CPU线程同步。

Every event in the log includes 1-byte event id and optional arguments. When argument is an array, it is stored as 4-byte array length and corresponding number of bytes with data. Here is the list of events that are written into the log:

 - EVENT_INSTRUCTION. Instructions executed since last event.
   Argument: 4-byte number of executed instructions.
 - EVENT_INTERRUPT. Used to synchronize interrupt processing.
 - EVENT_EXCEPTION. Used to synchronize exception handling.
 - EVENT_ASYNC. This is a group of events. They are always processed
   together with checkpoints. When such an event is generated, it is
   stored in the queue and processed only when checkpoint occurs.
   Every such event is followed by 1-byte checkpoint id and 1-byte
   async event id from the following list:
     - REPLAY_ASYNC_EVENT_BH. Bottom-half callback. This event synchronizes
       callbacks that affect virtual machine state, but normally called
       asynchronously.
       Argument: 8-byte operation id.
     - REPLAY_ASYNC_EVENT_INPUT. Input device event. Contains
       parameters of keyboard and mouse input operations
       (key press/release, mouse pointer movement).
       Arguments: 9-16 bytes depending of input event.
     - REPLAY_ASYNC_EVENT_INPUT_SYNC. Internal input synchronization event.
     - REPLAY_ASYNC_EVENT_CHAR_READ. Character (e.g., serial port) device input
       initiated by the sender.
       Arguments: 1-byte character device id. Array with bytes were read.
     - REPLAY_ASYNC_EVENT_BLOCK. Block device operation. Used to synchronize
       operations with disk and flash drives with CPU.
       Argument: 8-byte operation id.
     - REPLAY_ASYNC_EVENT_NET. Incoming network packet.
       Arguments: 1-byte network adapter id. 4-byte packet flags. Array with packet bytes.
 - EVENT_SHUTDOWN. Occurs when user sends shutdown event to qemu,
   e.g., by closing the window.
 - EVENT_CHAR_WRITE. Used to synchronize character output operations.
   Arguments: 4-byte output function return value. 4-byte offset in the output array.
 - EVENT_CHAR_READ_ALL. Used to synchronize character input operations,
   initiated by qemu.
   Argument: Array with bytes that were read.
 - EVENT_CHAR_READ_ALL_ERROR. Unsuccessful character input operation,
   initiated by qemu.
   Argument: 4-byte error code.
 - EVENT_CLOCK + clock_id. Group of events for host clock read operations.
   Argument: 8-byte clock value.
 - EVENT_CHECKPOINT + checkpoint_id. Checkpoint for synchronization of
   CPU, internal threads, and asynchronous input events. May be followed
   by one or more EVENT_ASYNC events.
 - EVENT_END. Last event in the log.

Every event in the log includes 1-byte event id and optional arguments. When argument is an array, it is stored as 4-byte array length and corresponding number of bytes with data. Here is the list of events that are written into the log:

每个被记录的事件都包括一个一字节的事件id和一些可选的参数。如果参数是一个数组，那么还会存储一个4字节的数组商都和相对应数据的字节数，下面是可能会被写入日志的各种事件的列表：

 - EVENT_INSTRUCTION. Instructions executed since last event.
   Argument: 4-byte number of executed instructions.
 - EVENT_INTERRUPT. Used to synchronize interrupt processing.
 - EVENT_EXCEPTION. Used to synchronize exception handling.
 - EVENT_ASYNC. This is a group of events. They are always processed
   together with checkpoints. When such an event is generated, it is
   stored in the queue and processed only when checkpoint occurs.
   Every such event is followed by 1-byte checkpoint id and 1-byte
   async event id from the following list:
     - REPLAY_ASYNC_EVENT_BH. Bottom-half callback. This event synchronizes
       callbacks that affect virtual machine state, but normally called
       asynchronously.
       Argument: 8-byte operation id.
     - REPLAY_ASYNC_EVENT_INPUT. Input device event. Contains
       parameters of keyboard and mouse input operations
       (key press/release, mouse pointer movement).
       Arguments: 9-16 bytes depending of input event.
     - REPLAY_ASYNC_EVENT_INPUT_SYNC. Internal input synchronization event.
     - REPLAY_ASYNC_EVENT_CHAR_READ. Character (e.g., serial port) device input
       initiated by the sender.
       Arguments: 1-byte character device id. Array with bytes were read.
     - REPLAY_ASYNC_EVENT_BLOCK. Block device operation. Used to synchronize
       operations with disk and flash drives with CPU.
       Argument: 8-byte operation id.
     - REPLAY_ASYNC_EVENT_NET. Incoming network packet.
       Arguments: 1-byte network adapter id. 4-byte packet flags. Array with packet bytes.
 - EVENT_SHUTDOWN. Occurs when user sends shutdown event to qemu,
   e.g., by closing the window.
 - EVENT_CHAR_WRITE. Used to synchronize character output operations.
   Arguments: 4-byte output function return value. 4-byte offset in the output array.
 - EVENT_CHAR_READ_ALL. Used to synchronize character input operations,
   initiated by qemu.
   Argument: Array with bytes that were read.
 - EVENT_CHAR_READ_ALL_ERROR. Unsuccessful character input operation,
   initiated by qemu.
   Argument: 4-byte error code.
 - EVENT_CLOCK + clock_id. Group of events for host clock read operations.
   Argument: 8-byte clock value.
 - EVENT_CHECKPOINT + checkpoint_id. Checkpoint for synchronization of
   CPU, internal threads, and asynchronous input events. May be followed
   by one or more EVENT_ASYNC events.
 - EVENT_END. Last event in the log.

[BH]: https://www.xuebuyuan.com/3206293.html