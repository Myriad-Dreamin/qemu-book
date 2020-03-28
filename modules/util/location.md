## Location

定义如下

```c
typedef struct Location {
    /* all members are private to qemu-error.c */
    enum { LOC_NONE, LOC_CMDLINE, LOC_FILE } kind;
    int num;
    const void *ptr;
    struct Location *prev;
} Location;
```

这是一个链表，或者目的是实现一个程序所处位置的栈。

`prev`为栈指针。

当`kind`等于`LOC_NONE`时，`num`, `ptr`都无效。

当`kind`等于`LOC_CMDLINE`时，`Location`保存了命令行`argv[offset...offset+cnt-1]`，`num`为命令行参数个数, `ptr`等于参数数组指针`argv+offset`，`ptr`类型为`char**`。

当`kind`等于`LOC_NONE`时，`Location`保存了文件名和文件编号，`num`为文件编号，`ptr`为文件名，`ptr`类型为`const char*`。

`qemu-error.c`中包含两个全局变量：

```c
static Location std_loc = {
    .kind = LOC_NONE
};
static Location *cur_loc = &std_loc;
```

`cur_loc`为一个全局`stack<Location>`的栈顶。

```c
Location *loc_push_restore(Location *loc);
Location *loc_push_none(Location *loc);
Location *loc_pop(Location *loc);
Location *loc_save(Location *loc);
void loc_restore(Location *loc);
void loc_set_none(void);
void loc_set_cmdline(char **argv, int idx, int cnt);
void loc_set_file(const char *fname, int lno);
```

#### `Location *loc_push_restore(Location *loc)`

把`loc`压到`cur_loc`栈中，新的栈顶指向`loc`,`loc`本身不变，返回`loc`

#### `Location *loc_push_none(Location *loc)`

将`loc`初始化为一个`kind`为`LOC_NONE`的`Location`，并调用`loc_push_restore`

#### `Location *loc_pop(Location *loc)`

强制要求`loc==cur_loc`，当栈非空时，新的栈顶指向`loc->prev`,`loc`本身不变，返回`loc`

#### `Location *loc_save(Location *loc)`

将`cur_loc`赋值给`loc`，并令`loc->prev`为空，返回`loc`。

#### `void loc_restore(Location *loc)`

强制要求`loc->prev`为空。并将`loc`除了`prev`的字段都赋值给`cur_loc`

#### `void loc_set_xxx`

xxx可以为`none`, `cmdline`或`file`