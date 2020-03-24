## Migration模块

### migration.c

###### `void migration_object_init(void)`

该函数只能被调用一次。首先初始化文件的全局变量`current_migration`和`current_incoming`，这两个对象都需要保证线程安全。所谓迁移，`current_incoming`即是源，`current_migration`是过程控制器，当前线程是目标。

初始化了`finish_lock`。

初始化了`block_mig_state`

### block.c

###### `void blk_mig_init(void)`

首先初始化`block_mig_state`，然后将`block`设备注册到热迁移模块中，使用的函数为`register_savevm_live`。

### ram.c

###### `void ram_mig_init(void)`

首先初始化`XBZRLE`，然后把`ram`设备注册到热迁移模块中，使用的函数为`register_savevm_live`。

### block-dirty-map.c

###### `void dirty_bitmap_mig_init(void)`

首先初始化`dirty_bitmap_mig_state`，然后把`register_savevm_live`设备注册到热迁移模块中，使用的函数为`register_savevm_live`。

### global_state.c

###### `register_migrate_state`

初始化`global_state`然后将其注册到迁移状态中，使用的函数为`vmstate_register`。

