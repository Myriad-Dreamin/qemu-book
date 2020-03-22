## Replay模块

### replay.c/replay.h/replay-internal.h

存在下面三个枚举量：

```c
/* replay clock kinds */
enum ReplayClockKind {
    /* host_clock */
    REPLAY_CLOCK_HOST,
    /* virtual_rt_clock */
    REPLAY_CLOCK_VIRTUAL_RT,
    REPLAY_CLOCK_COUNT
};
typedef enum ReplayClockKind ReplayClockKind;

/* IDs of the checkpoints */
enum ReplayCheckpoint {
    CHECKPOINT_CLOCK_WARP_START,
    CHECKPOINT_CLOCK_WARP_ACCOUNT,
    CHECKPOINT_RESET_REQUESTED,
    CHECKPOINT_SUSPEND_REQUESTED,
    CHECKPOINT_CLOCK_VIRTUAL,
    CHECKPOINT_CLOCK_HOST,
    CHECKPOINT_CLOCK_VIRTUAL_RT,
    CHECKPOINT_INIT,
    CHECKPOINT_RESET,
    CHECKPOINT_COUNT
};
typedef enum ReplayCheckpoint ReplayCheckpoint;

/* Any changes to order/number of events will need to bump REPLAY_VERSION */
enum ReplayEvents {
    /* for instruction event */
    EVENT_INSTRUCTION,
    /* for software interrupt */
    EVENT_INTERRUPT,
    /* for emulated exceptions */
    EVENT_EXCEPTION,
    /* for async events */
    EVENT_ASYNC,
    /* for shutdown requests, range allows recovery of ShutdownCause */
    EVENT_SHUTDOWN,
    EVENT_SHUTDOWN_LAST = EVENT_SHUTDOWN + SHUTDOWN_CAUSE__MAX,
    /* for character device write event */
    EVENT_CHAR_WRITE,
    /* for character device read all event */
    EVENT_CHAR_READ_ALL,
    EVENT_CHAR_READ_ALL_ERROR,
    /* for audio out event */
    EVENT_AUDIO_OUT,
    /* for audio in event */
    EVENT_AUDIO_IN,
    /* for clock read/writes */
    /* some of greater codes are reserved for clocks */
    EVENT_CLOCK,
    EVENT_CLOCK_LAST = EVENT_CLOCK + REPLAY_CLOCK_COUNT - 1,
    /* for checkpoint event */
    /* some of greater codes are reserved for checkpoints */
    EVENT_CHECKPOINT,
    EVENT_CHECKPOINT_LAST = EVENT_CHECKPOINT + CHECKPOINT_COUNT - 1,
    /* end of log event */
    EVENT_END,
    EVENT_COUNT
};
```

replay所有的状态都被存在这样一个全局变量中：

```c
typedef struct ReplayState {
    /*! Cached clock values. */
    int64_t cached_clock[REPLAY_CLOCK_COUNT];
    /*! Current icount - number of processed instructions. */
    uint64_t current_icount;
    /*! Number of instructions to be executed before other events happen. */
    int instruction_count;
    /*! Type of the currently executed event. */
    unsigned int data_kind;
    /*! Flag which indicates that event is not processed yet. */
    unsigned int has_unread_data;
    /*! Temporary variable for saving current log offset. */
    uint64_t file_offset;
    /*! Next block operation id.
        This counter is global, because requests from different
        block devices should not get overlapping ids. */
    uint64_t block_request_id;
    /*! Prior value of the host clock */
    uint64_t host_clock_last;
    /*! Asynchronous event type read from the log */
    int32_t read_event_kind;
    /*! Asynchronous event id read from the log */
    uint64_t read_event_id;
    /*! Asynchronous event checkpoint id read from the log */
    int32_t read_event_checkpoint;
} ReplayState;
extern ReplayState replay_state;
```

```c
extern ReplayMode replay_mode;

/* Name of the initial VM snapshot */
extern char *replay_snapshot;
```

`replay.h`中几个比较重要或典型的函数：

```c
/* Replay locking
 *
 * The locks are needed to protect the shared structures and log file
 * when doing record/replay. They also are the main sync-point between
 * the main-loop thread and the vCPU thread. This was a role
 * previously filled by the BQL which has been busy trying to reduce
 * its impact across the code. This ensures blocks of events stay
 * sequential and reproducible.
 */

void replay_mutex_lock(void);
void replay_mutex_unlock(void);

/* Replay process control functions */

/*! Enables recording or saving event log with specified parameters */
void replay_configure(struct QemuOpts *opts);
/*! Initializes timers used for snapshotting and enables events recording */
void replay_start(void);
/*! Closes replay log file and frees other resources. */
void replay_finish(void);
/*! Adds replay blocker with the specified error description */
void replay_add_blocker(Error *reason);

/*! Should be called at check points in the execution.
    These check points are skipped, if they were not met.
    Saves checkpoint in the SAVE mode and validates in the PLAY mode.
    Returns 0 in PLAY mode if checkpoint was not found.
    Returns 1 in all other cases. */
bool replay_checkpoint(ReplayCheckpoint checkpoint);
/*! Used to determine that checkpoint is pending.
    Does not proceed to the next event in the log. */
bool replay_has_checkpoint(void);

/*! Disables storing events in the queue */
void replay_disable_events(void);
/*! Enables storing events in the queue */
void replay_enable_events(void);
/*! Returns true when saving events is enabled */
bool replay_events_enabled(void);

/*! Called at the start of execution.
    Loads or saves initial vmstate depending on execution mode. */
void replay_vmstate_init(void);
/*! Called to ensure that replay state is consistent and VM snapshot
    can be created */
bool replay_can_snapshot(void);    
```

###### `replay_mutex_lock/unlock`

一些重要的函数被`replay_mutex`封装

###### `replay_enable/disable_events`

是否`replay`可以被这两个函数控制

#### replay控制函数组

###### `void replay_configure(struct QemuOpts *opts)`

opts为空时，此函数无效。

`opts`包括:

+ `rr`
  + `record`: `mode`被设置为`REPLAY_MODE_RECORD`
  + `replay`: `mode`被设置为`REPLAY_MODE_REPLAY`
  + 默认: `mode`被设置为`REPLAY_MODE_NONE`
+ `rrfile`
  + 必要，指定了replay日志的文件。

随后调用`replay_vmstate_register`和`replay_enable`。在这个版本中`replay_vmstate_register`没有任何用处。`replay_enable`会：

+ 注册离开程序时的回调函数`replay_finish`
+ 初始化`replay_file`、`replay_mutex`、`replay_state`

###### `void replay_start(void)`

这个函数约等于`replay_enable_event`，但是会做一些边界检查。

如果`replay_mode`仍然是`REPLAY_MODE_NONE`，那么直接返回。如果`replay_blockers`中被报告了错误，程序会直接退出，如果`user_icount`选项值为`false`，那么replay将无法进行，程序直接退出。

否则调用`replay_enable_event`，现在`replay`模块将开始接受事件的记录。

###### `void replay_finish(void)`

如果`replay_mode`仍然是`REPLAY_MODE_NONE`，那么直接返回。否则会：

+ 跟进刷新指令事件（如果`icount > 0`，那么新增一个执行指令的replay事件到日志中）
+ 刷新`replay_file`的`version`字段并关闭
+ 清空`replay_snapshot`

###### `void replay_add_blocker(Error *reason)`

`blockers`是个链表，这个函数将`reason`插入到链表中。

###### `replay_vmstate_init`

如果`replay_mode`为`REPLAY_MODE_RECORE`就保存快照(`save_snapshot`)，否则读取快照(`read_snapshot`)，快照模块在`migration/snapshot.h`中。

###### `bool replay_next_event_is(int event)`

如果当前状态仍有剩余的指令未完成，那么直接断言当前状态事件和event等于`EVENT_INSTRUCTION`。

否则一边处理关机事件一边移动，直到当前状态的事件不再是关机状态。如果中途遇到了与`event`类型相同的事件，那么最终返回true。

附`replay.h`部分声明：

```c
/* Replay locking
 *
 * The locks are needed to protect the shared structures and log file
 * when doing record/replay. They also are the main sync-point between
 * the main-loop thread and the vCPU thread. This was a role
 * previously filled by the BQL which has been busy trying to reduce
 * its impact across the code. This ensures blocks of events stay
 * sequential and reproducible.
 */

void replay_mutex_lock(void);
void replay_mutex_unlock(void);

/* Replay process control functions */

/*! Enables recording or saving event log with specified parameters */
void replay_configure(struct QemuOpts *opts);
/*! Initializes timers used for snapshotting and enables events recording */
void replay_start(void);
/*! Closes replay log file and frees other resources. */
void replay_finish(void);
/*! Adds replay blocker with the specified error description */
void replay_add_blocker(Error *reason);

/* Processing the instructions */

/*! Returns number of executed instructions. */
uint64_t replay_get_current_icount(void);
/*! Returns number of instructions to execute in replay mode. */
int replay_get_instructions(void);
/*! Updates instructions counter in replay mode. */
void replay_account_executed_instructions(void);

/* Interrupts and exceptions */

/*! Called by exception handler to write or read
    exception processing events. */
bool replay_exception(void);
/*! Used to determine that exception is pending.
    Does not proceed to the next event in the log. */
bool replay_has_exception(void);
/*! Called by interrupt handlers to write or read
    interrupt processing events.
    \return true if interrupt should be processed */
bool replay_interrupt(void);
/*! Tries to read interrupt event from the file.
    Returns true, when interrupt request is pending */
bool replay_has_interrupt(void);

/* Processing clocks and other time sources */

/*! Save the specified clock */
int64_t replay_save_clock(ReplayClockKind kind, int64_t clock,
                          int64_t raw_icount);
/*! Read the specified clock from the log or return cached data */
int64_t replay_read_clock(ReplayClockKind kind);
/*! Saves or reads the clock depending on the current replay mode. */
#define REPLAY_CLOCK(clock, value)                                      \
    (replay_mode == REPLAY_MODE_PLAY ? replay_read_clock((clock))       \
        : replay_mode == REPLAY_MODE_RECORD                             \
            ? replay_save_clock((clock), (value), cpu_get_icount_raw()) \
        : (value))
#define REPLAY_CLOCK_LOCKED(clock, value)                               \
    (replay_mode == REPLAY_MODE_PLAY ? replay_read_clock((clock))       \
        : replay_mode == REPLAY_MODE_RECORD                             \
            ? replay_save_clock((clock), (value), cpu_get_icount_raw_locked()) \
        : (value))

/* Events */

/*! Called when qemu shutdown is requested. */
void replay_shutdown_request(ShutdownCause cause);
/*! Should be called at check points in the execution.
    These check points are skipped, if they were not met.
    Saves checkpoint in the SAVE mode and validates in the PLAY mode.
    Returns 0 in PLAY mode if checkpoint was not found.
    Returns 1 in all other cases. */
bool replay_checkpoint(ReplayCheckpoint checkpoint);
/*! Used to determine that checkpoint is pending.
    Does not proceed to the next event in the log. */
bool replay_has_checkpoint(void);

/* Asynchronous events queue */

/*! Disables storing events in the queue */
void replay_disable_events(void);
/*! Enables storing events in the queue */
void replay_enable_events(void);
/*! Returns true when saving events is enabled */
bool replay_events_enabled(void);
/*! Adds bottom half event to the queue */
void replay_bh_schedule_event(QEMUBH *bh);
/* Adds oneshot bottom half event to the queue */
void replay_bh_schedule_oneshot_event(AioContext *ctx,
    QEMUBHFunc *cb, void *opaque);
/*! Adds input event to the queue */
void replay_input_event(QemuConsole *src, InputEvent *evt);
/*! Adds input sync event to the queue */
void replay_input_sync_event(void);
/*! Adds block layer event to the queue */
void replay_block_event(QEMUBH *bh, uint64_t id);
/*! Returns ID for the next block event */
uint64_t blkreplay_next_id(void);

/* Character device */

/*! Registers char driver to save it's events */
void replay_register_char_driver(struct Chardev *chr);
/*! Saves write to char device event to the log */
void replay_chr_be_write(struct Chardev *s, uint8_t *buf, int len);
/*! Writes char write return value to the replay log. */
void replay_char_write_event_save(int res, int offset);
/*! Reads char write return value from the replay log. */
void replay_char_write_event_load(int *res, int *offset);
/*! Reads information about read_all character event. */
int replay_char_read_all_load(uint8_t *buf);
/*! Writes character read_all error code into the replay log. */
void replay_char_read_all_save_error(int res);
/*! Writes character read_all execution result into the replay log. */
void replay_char_read_all_save_buf(uint8_t *buf, int offset);

/* Network */

/*! Registers replay network filter attached to some backend. */
ReplayNetState *replay_register_net(NetFilterState *nfs);
/*! Unregisters replay network filter. */
void replay_unregister_net(ReplayNetState *rns);
/*! Called to write network packet to the replay log. */
void replay_net_packet_event(ReplayNetState *rns, unsigned flags,
                             const struct iovec *iov, int iovcnt);

/* Audio */

/*! Saves/restores number of played samples of audio out operation. */
void replay_audio_out(size_t *played);
/*! Saves/restores recorded samples of audio in operation. */
void replay_audio_in(size_t *recorded, void *samples, size_t *wpos, size_t size);

/* VM state operations */

/*! Called at the start of execution.
    Loads or saves initial vmstate depending on execution mode. */
void replay_vmstate_init(void);
/*! Called to ensure that replay state is consistent and VM snapshot
    can be created */
bool replay_can_snapshot(void);

```

