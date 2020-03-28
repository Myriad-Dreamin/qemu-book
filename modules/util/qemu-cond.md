## QemuCond

是一种条件锁，当条件不成立时，线程会阻塞，直到条件成立锁才会被重新释放。

```c
struct QemuCond {
    pthread_cond_t cond;
    bool initialized;
};
```

可见它继承了`pthread_cond_t`。在旧版本中`initialized`并不存在。

```c
typedef union
{
  struct __pthread_cond_s __data;
  char __size[__SIZEOF_PTHREAD_COND_T];
  __extension__ long long int __align;
} pthread_cond_t;
```

有关`pthread_cond_t`的文章参考[线程同步机制条件变量的使用与思考](https://answerywj.com/2017/12/15/condition-variables-of-thread-synchronization/)，关于`while(cond) pthread_cond_wait`的概念，参考[Spurious wakeups](http://blog.vladimirprus.com/2005/07/spurious-wakeups.html)

`QemuCond`的方法如下，它与`pthread_cond_t`所有的方法一一对应：

#### `void qemu_cond_init(QemuCond *cond)`

调用`pthread_cond_init`后如果错误直接退出程序，否则置`initialized`为true。

#### `void qemu_cond_destroy(QemuCond *cond)`

断言`initialized`为true，将`initialized`置为false后调用`pthread_cond_destroy`。如果发生错误直接退出程序。

#### `void qemu_cond_signal(QemuCond *cond)`

断言`initialized`为true，调用`pthread_cond_signal`。如果发生错误直接退出程序。

#### `void qemu_cond_broadcast(QemuCond *cond)`

断言`initialized`为true，调用`pthread_cond_broadcast`。如果发生错误直接退出程序。

#### `void qemu_cond_wait_impl(QemuCond *cond, QemuMutex *mutex, const char *file, const int line)`

断言`initialized`为true，调用`pthread_cond_wait`。如果发生错误直接退出程序。`file`/`line`用于trace`cond&mutex`造成的阻塞情况


#### `bool qemu_cond_timedwait_impl(QemuCond *cond, QemuMutex *mutex, int ms, const char *file, const int line)`

断言`initialized`为true，调用`pthread_cond_timedwait`。如果发生错误且错误不为deadline超时，则直接退出程序。`file`/`line`用于trace`cond&mutex`造成的阻塞情况

