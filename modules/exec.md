## exec.c

### exec.c

###### `void cpu_exec_init_all(void)`

初始化了内存操作互斥锁`ram_list.mutex`，和一个暂时未知的锁`map_client_list_lock`

调用了`finalize_target_page_bits`，该函数的作用是确定一页的大小。

`memory_map_init`。

内存`io_mem_unassigned`初始化大小为`UINT64_MAX`、terminates为`true`、其他值默认。内存`system_memory`初始化大小为`UINT64_MAX`、名称为`"system"`、其他值默认。初始化内存`system_io`的名为`"io"`，大小为65536、其他值默认。默认值参考`memory_region_init`。

将地址空间`address_space_memory`绑定给`system_memory`，名称为`"memory"`，将地址空间`address_space_io`绑定给`system_io`，名称为`"I/O"`。初始化过程参考`address_space_init`

###### `void page_size_init(void)`

初始化全局变量`qemu_host_page_size`和`qemu_host_page_mask`。

如果`qemu_host_page_size`为0，那么其值被赋为`qemu_real_host_page_size`，如果`qemu_host_page_size`小于`TARGET_PAGE_SIZE`，那么重新赋值为`TARGET_PAGE_SIZE`，`TARGET_PAGE_SIZE`在`cpu-all.h`中被确定，例如在`i386`上，其值为4KB。`qemu_host_page_mask`是`qemu_host_page_size`的在`intptr_t`上的相反数。

### exec-vary.c

`void finalize_target_page_bits(void)`

如果定义了`TARGET_PAGE_BITS_VARY`，那么初始化的是`init_target_page`，成员`bit`会根据目标架构的不同有不同的值，例如在`i386`上，bit等于12，相当于一页有4KB（$2^{12}$个字节）

否则，说明目标的页大小已经被常数的确认了，什么都不用做。

函数使用到了`TARGET_PAGE_BfITS_MIN`，该宏被定义于`target/xxx/cpu-param.h`，`xxx`是目标架构的名字。