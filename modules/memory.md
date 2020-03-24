## memory.c

### memory.c

###### `memory_region_init_io`

签名及说明如下：

```c
/**
 * @mr: the #MemoryRegion to be initialized.
 * @owner: the object that tracks the region's reference count
 * @ops: a structure containing read and write callbacks to be used when
 *       I/O is performed on the region.
 * @opaque: passed to the read and write callbacks of the @ops structure.
 * @name: used for debugging; not visible to the user or ABI
 * @size: size of the region.
 */
void memory_region_init_io(MemoryRegion *mr,
                           struct Object *owner,
                           const MemoryRegionOps *ops,
                           void *opaque,
                           const char *name,
                           uint64_t size);
```

负责初始化`mr`，后面的值都是要绑定到`mr`上的参数。

如果`ops`为空，那么赋默认值`unassigned_mem_ops`

如果`name`不为空，那么用`object_property_add_child`将`mr`绑定给`Object`对象`(owner || container_get(qdev_get_machine(), "/unattached"))`到属性`g_strdup_printf("%s[*]", memory_region_escape_name(name))`上。

`mr`的字段`terminates`会被赋为`true`。

###### ``

