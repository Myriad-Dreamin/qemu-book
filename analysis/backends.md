## backends

我们在很多地方都能够碰到backend这个概念。例如web中后端的概念是负责处理数据的服务器或集群、编译原理中后端的概念是负责将前端的处理好的结构化语言dump成机器代码。我们注意到backend概念总是与frontend概念成对出现。在qemu中，frontend即为虚拟机众多虚拟化设备，而backend则为支持庞大虚拟系统的底层中间件/软件/硬件接口。

该部分一共包含五个backend大类。

+ `cryptodev`: 根据名称可以猜测，这是Crypto Device的缩写，是一种硬件加速加密技术。
+ `hostmem`: 根据名称可以猜测，这是Host Memory的缩写，大概负责对接虚拟机操纵宿主机内存。宿主机内存可以是真实内存，也可以是虚存，也可以是QEMU模拟的内存文件。
+ `rng`: 随机数生成器。
+ `tpm`: Trusted Platform Module，这个概念还很陌生，但是据网上讲，这个部分是QEMU4.0最重要的更新之一。
+ `vhost-user`: 是virtio的用户后端驱动。

### cryptodev

#### 类的初始化

由于在C语言中不存在类，这里使用模拟的方法。首先创建一个静态全局变量`cryptodev_backend_info`，并通过gcc的基础设施`__attribute__((constructor))`在`main`函数前执行`cryptodev_backend_register_types`方法根据`type_info`初始化整个类。

```c

// class init handler
static void
cryptodev_backend_class_init(ObjectClass *oc, void *data)
{
    UserCreatableClass *ucc = USER_CREATABLE_CLASS(oc);

    ucc->complete = cryptodev_backend_complete;
    ucc->can_be_deleted = cryptodev_backend_can_be_deleted;

    QTAILQ_INIT(&crypto_clients);
}

// type info
static const TypeInfo cryptodev_backend_info = {
    ...
    .parent = TYPE_OBJECT,
    .class_init = cryptodev_backend_class_init,
    .interfaces = (InterfaceInfo[]) {
        { TYPE_USER_CREATABLE },
        { }
    }
};

// type init handler
static void
cryptodev_backend_register_types(void)
{
    type_register_static(&cryptodev_backend_info);
}

// gcc constructor
type_init(cryptodev_backend_register_types);
```

`type_init`是qom的宏函数，未来可以专门写一篇文章介绍`qom`模型。

#### `CryptoDevBackend`中转接口

以下5个方法都是首先根据`handler`即`CryptoDevBackend backend`动态获取类对象，再根据类对象中保存的指针完成对应操纵。
+ `cryptodev_backend_cleanup`: 调用`cleanup`
+ `cryptodev_backend_sym_create_session`: 调用`create_session`
+ `cryptodev_backend_sym_close_session`: 调用`close_session`
+ `cryptodev_backend_sym_operation`: 调用`do_sym_op`
+ `cryptodev_backend_complete`: 调用`init`

如：

```c
void cryptodev_backend_cleanup(CryptoDevBackend *backend, Error **errp) {
    CryptoDevBackendClass *bc = CRYPTODEV_BACKEND_GET_CLASS(backend);
    if (bc->cleanup) bc->cleanup(backend, errp);
}
```

###### `int cryptodev_backend_crypto_operation(CryptoDevBackend *backend, void *opaque, uint32_t queue_index, Error **errp)`

是`cryptodev_backend_sym_operation`的包装接口，从`opaque`指针中获取`cryptodev_backend_sym_operation`需要的数据然后调用。

###### `static void cryptodev_backend_finalize(Object *obj)`

是`cryptodev_backend_cleanup`的包装接口。 从`obj`中用`CRYPTODEV_BACKEND`获取`CryptoDev`，并调用`cryptodev_backend_cleanup`方法。

#### `CryptoDevBackend`的getter和setter

+ `cryptodev_backend_get_queues(Object *obj, ...)`: 从`obj`中用`CRYPTODEV_BACKEND`获取`CryptoDev`，并访问其`backend->conf.peers.queues`属性。
+ `cryptodev_backend_set_queues(Object *obj, ...)`，访问其`backend->conf.peers.queues`属性。
+ 其他: 自行查看源码

#### `CryptoDevBackend`类的实现

`CryptoDevBackend`在QEMU中一共有两个实现，一个是`CryptoDevBackendBuiltin`，另一个是`CryptoDevBackendVhostUser`。

`CryptoDevBackendBuiltin`的实现在`cryptodev-builtin.c`中，这个文件中只存放了该类的方法实现。

`CryptoDevBackendVhostUser`的实现在`cryptodev-vhost-user.c`中，`cryptodev-vhost.c`中存放了该类针对`vhost`的辅助函数。

#### 线索

以下六个函数在`hw/virtio/virtio-crypto.c`中被调用。

+ `cryptodev_backend_sym_create_session`
+ `cryptodev_backend_sym_close_session`
+ `cryptodev_backend_crypto_operation`
+ `cryptodev_backend_set_used`
+ `cryptodev_backend_is_used`
+ `cryptodev_backend_is_ready`

`CryptoDevBackend`类仅被`hw/virtio/virtio-crypto.c`直接依赖。

#### 工作方式

`hw/virtio/virtio-crypto.c`使用`cryptodev_backend_sym_create_session`创建一个`CryptoDevBackend`的session，获得一个session id。得到session id后，将session id连同request一起传入`cryptodev_backend_crypto_operation`中，执行多次加密操作。所有操作执行完以后调用`cryptodev_backend_sym_close_session`关闭session。

### hostmem

`HostMemoryBackend`是一个抽象类，提供了大量而平凡的`getter`和`setter`，而其真正有用的方法只有一个`alloc`，这个方法负责将成员`MemoryRegion mr`初始化。

#### 类的初始化

与`cryptodev`相同。

#### `HostMemoryBackend`类的实现

`HostMemoryBackend`有三种实现：

+ `memory-backend-file`: 模拟虚存，实际对应读写一个内存文件，使用的是`qemu`原生api。入口是`memory_region_init_ram_from_file`，具体行为自行翻阅。

+ `memory-backend-memfd`: 使用linux的`memfd_create`创建`MemoryRegion`对象，实际上还是读写文件，但是使用了`mmap`、file sealing等技术。
+ `memory-backend-ram`: 直接在内存中开辟空间，读写内存。

#### 线索

非常奇怪，并没有任何显式的初始化，这让`HostMemoryBackend`的使用变得扑朔迷离。经过多次寻找，发现了在option中存在`memory-backend`。猜测`memory-backend`以字符串形式被配置，通过反射的方式初始化。

#### 工作方式

`HostMemoryBackend`相当于一个abstract provider，它包装了`mr`，当该对象初始化时，直接调用`alloc`方法，使得`mr`在被使用的时候能懒惰地初始化。`HostMemoryBackend`没有实现`alloc`，所以它是抽象的，无法直接实例化。

### rng

`RngBackend`是一个抽象类，大致代码思路与hostmem相同。

#### 类的初始化

与`cryptodev`相同。

#### `RngBackend`中转接口

+ `rng_backend_request_entropy`: 调用`request_entropy`，如果该方法没有被实现，直接忽略。

#### 工作方式

提供`rng_backend_request_entropy`，这个函数接受`opaque`数据和一个callback`receive_entropy`，当`opaque`任务计算完成以后调用`receive_entropy`。

#### `RngBackend`类的实现

`RngBackend`有三种实现：

+ `rng-builtin`: 使用硬件计算，当硬件计算完以后，在behind half调用callback

+ `rng-egd`: 从Entropy Gathering Daemon中获取随机字节流，当计算完成以后，异步callback。

+ `rng-random`: 没看懂

  下面这个方法，根本没有使用到`req`这个变量，包括宏函数中。

  ```c
  static void rng_random_request_entropy(RngBackend *b, RngRequest *req)
  {
      RngRandom *s = RNG_RANDOM(b);
  
      if (QSIMPLEQ_EMPTY(&s->parent.requests)) {
          /* If there are no pending requests yet, we need to
           * install our fd handler. */
          qemu_set_fd_handler(s->fd, entropy_available, NULL, s);
      }
  }
  ```

### tpm

`tpm`是Trusted Platform Module的缩写，是一种硬件，在该硬件上运行的命令是可信赖，可验证的。tpm模块用于向硬件TPM芯片请求命令。

该文件中只定义了抽象类`TPMBackend`和接口类`TPMIf`。

```c
typedef struct TPMIfClass {
    InterfaceClass parent_class;
    enum TpmModel model;
    void (*request_completed)(TPMIf *obj, int ret);
    enum TPMVersion (*get_version)(TPMIf *obj);
} TPMIfClass;
```

在`hw/tpm`中，实现了两个`TPMBackend`的子类：`TPMEmulator`、`TPMPassthruState`。

#### tpm重要接口介绍

+ `tpm_backend_init`: 将`TPMIf`绑定到`TPMBackend`上。
+ `tpm_backend_startup_tpm`: 重新启动tpm模块。
+ `tpm_backend_deliver_request`: 以aio异步的方式完成tpm命令。
+ `tpm_backend_reset`: 重置tpm模块硬件状态。
+ `tpm_backend_cancel_cmd`: 取消运行的命令。
+ `tpm_backend_query_tpm`: 查询tpm模块状态。

#### 工作方式

通过接口控制tpm芯片执行命令。

### vhost-user

vhost是virtio的后端驱动。当人们深感一部分虚拟化不需要放到内核时，诞生了用户态的vhost，也就是vhost-user。vhost-user的介绍可以看[virtio, vhost和vhost-user](https://blog.csdn.net/qq_15437629/article/details/77899905)

**未完待续**