基于master，版本大约在`5.0.0-rc3`。

### trace-events如何被应用到c代码中

trace-events并不属于任何一种语言，它仅仅是一个事件列表，每一行代表一个trace事件。

##### configure中与trace相关的逻辑

为了弄清楚这些文本如何被应用，我们先看`configure`如何配置`trace`相关的选项。

+ 437行中声明了`trace_backends`的默认值为`log`

+ 438行中声明了`trace_file`的默认值为`trace`

+ 1054/1055行中将命令行选项`--enable-trace-backends/--enable-trace-backend`参数列表赋给`trace_backends`

  ```
  --enable-trace-backends=B Set trace backend
                          Available backends: $trace_backend_list
  ```

+ 1056行将命令行选项`--with-trace-file`参数赋值给`trace_file`

  ```
  --with-trace-file=NAME   Full PATH,NAME of file to store traces
                           Default:trace-<pid>
  ```

+ 1702行调用命令得到目前支持的trace列表

  ```bash
  trace_backend_list=$(echo $(grep -le '^PUBLIC = True$' "$source_path"/scripts/tracetool/backend/*.py | sed -e 's/^.*\/\(.*\)\.py$/\1/'))
  ```

  该命令的意思是获取`scripts/tracetool/backend/*.py`中含有`PUBLIC = true`脚本的无扩展名称。

  在当前版本中执行获得列表：

  ```bash
  #等价于
  trace_backend_list=(dtrace ftrace log simple syslog ust)
  ```

+ 5391行起执行有效性检查

  ```bash
  ##########################################
  # check if trace backend exists
  
  $python "$source_path/scripts/tracetool.py" "--backends=$trace_backends" --check-backends  > /dev/null 2> /dev/null
  if test "$?" -ne 0 ; then
    error_exit "invalid trace backends" \
        "Please choose supported trace backends."
  fi
  
  # 省略与ust, dtrace有关的部分
  ```

+ 6676行打印了捕捉到的`trace_backends`

  ```bash
  echo "Trace backends    $trace_backends"
  ```

  所以如果你使用了`configure`，可以在terminal中看到配置的参数。

+ 6677行打印了捕捉到的`trace_file`

  ```bash
  if have_backend "simple"; then
  echo "Trace output file $trace_file-<pid>"
  fi
  ```

  可见如果后端不包括simple类型，那么该选项不生效（按常理）

+ 7444行开始将配置输出

  ```bash
  # 配置掩码文件
  config_host_mak="config-host.mak"
  # 省略非trace部分配置输出
  
  echo "TRACE_BACKENDS=$trace_backends" >> $config_host_mak
  if have_backend "nop"; then
    echo "CONFIG_TRACE_NOP=y" >> $config_host_mak
  fi
  if have_backend "simple"; then
    echo "CONFIG_TRACE_SIMPLE=y" >> $config_host_mak
    # Set the appropriate trace file.
    trace_file="\"$trace_file-\" FMT_pid"
  fi
  # 省略 dtrace ftrace log syslog ust 等的配置输出
  echo "CONFIG_TRACE_FILE=$trace_file" >> $config_host_mak
  ```

##### makefile生成trace目标文件并链接到C代码中

使用`configure`生成配置：

```
../configure --target-list=x86_64-softmmu --enable-debug --enable-trace-backends=simple
```

根据上一节我们知道配置都被输出到了`config-host.mak`中，我们查看其中`trace`相关的选项

```bash
kamiyoru@kamiya:~/work/c/qemu-latest/build$ cat config-host.mak | grep -i trace
TRACE_BACKENDS=simple
CONFIG_TRACE_SIMPLE=y
CONFIG_TRACE_FILE="trace-" FMT_pid
```

现在我们关注主Makefile，这里我们不关心一些配置变量的声明，主要是看`trace`文件如何被生成。

+ 180行声明了trace命令和依赖：

  ```makefile
  trace-group-name = $(shell dirname $1 | sed -e 's/[^a-zA-Z0-9]/_/g')
  
  tracetool-y = $(SRC_PATH)/scripts/tracetool.py
  tracetool-y += $(shell find $(SRC_PATH)/scripts/tracetool -name "*.py")
  ```

+ 183行开始声明了编译规则，对应的`trace.h`需要被同目录下的文件引用，make会帮对应文件include到正确的头文件上。

  ```makefile
  %/trace.h: %/trace.h-timestamp
  	@cmp $< $@ >/dev/null 2>&1 || cp $< $@
  %/trace.h-timestamp: $(SRC_PATH)/%/trace-events $(tracetool-y) $(BUILD_DIR)/config-host.mak
  	$(call quiet-command,$(TRACETOOL) \
  		--group=$(call trace-group-name,$@) \
  		--format=h \
  		--backends=$(TRACE_BACKENDS) \
  		$< > $@,"GEN","$(@:%-timestamp=%)")
  		
  %/trace.c: %/trace.c-timestamp
  	@cmp $< $@ >/dev/null 2>&1 || cp $< $@
  %/trace.c-timestamp: $(SRC_PATH)/%/trace-events $(tracetool-y) $(BUILD_DIR)/config-host.mak
  	$(call quiet-command,$(TRACETOOL) \
  		--group=$(call trace-group-name,$@) \
  		--format=c \
  		--backends=$(TRACE_BACKENDS) \
  		$< > $@,"GEN","$(@:%-timestamp=%)")
  
  trace-root.h: trace-root.h-timestamp
  	@cmp $< $@ >/dev/null 2>&1 || cp $< $@
  trace-root.h-timestamp: $(SRC_PATH)/trace-events $(tracetool-y) $(BUILD_DIR)/config-host.mak
  	$(call quiet-command,$(TRACETOOL) \
  		--group=root \
  		--format=h \
  		--backends=$(TRACE_BACKENDS) \
  		$< > $@,"GEN","$(@:%-timestamp=%)")
  
  # 省略与 ust, dtrace有关的部分
  
  trace-root.c: trace-root.c-timestamp
  	@cmp $< $@ >/dev/null 2>&1 || cp $< $@
  trace-root.c-timestamp: $(SRC_PATH)/trace-events $(tracetool-y) $(BUILD_DIR)/config-host.mak
  	$(call quiet-command,$(TRACETOOL) \
  		--group=root \
  		--format=c \
  		--backends=$(TRACE_BACKENDS) \
  		$< > $@,"GEN","$(@:%-timestamp=%)")
  
  # 省略与 ust, dtrace有关的部分
  ```

很多人都不熟悉makefile，这里择取其中一个：

```makefile
# 任意文件夹的trace.h的源文件为trace.h-timestamp
%/trace.h: %/trace.h-timestamp
    # 如果 trace.h-timestamp 和 trace.h的内容不相同，则copy trace.h-timestamp trace.h
	@cmp $< $@ >/dev/null 2>&1 || cp $< $@

# 任意文件夹的dir/trace.h-timestamp的源文件为dir/trace-events，以及编译工具和配置文件
%/trace.h-timestamp: $(SRC_PATH)/%/trace-events $(tracetool-y) $(BUILD_DIR)/config-host.mak
	# 这里TRACETOOL等于/usr/bin/python3 -B $(SRC_PATH)/scripts/tracetool.py
	$(call quiet-command,$(TRACETOOL) \
	    # 这里group等于目标路径转下划线，可以取看trace-group-name定义
	    # 例如hw/scsi-abc转义结果为hw_scsi_abc
		--group=$(call trace-group-name,$@) \
		--format=h \
		--backends=$(TRACE_BACKENDS) \
		# 将 dir/trace-events 作为输入 ($<)，将结果重定向到dir/trace.h-timestamp
		$< > $@,"GEN","$(@:%-timestamp=%)")
```

例子：

```bash
kamiyoru@kamiya:~/work/c/qemu-latest/test$ ../scripts/tracetool.py --group=abc --format=h --backends=simple ../backends/trace-events > abc.h
```

直观感受一下`group=abc`的作用：

```bash
kamiyoru@kamiya:~/work/c/qemu-latest/test$ cat abc.h | grep -i abc
#ifndef TRACE_ABC_GENERATED_TRACERS_H
#define TRACE_ABC_GENERATED_TRACERS_H
#endif /* TRACE_ABC_GENERATED_TRACERS_H */
```

`trace-tool.py`是一个入口文件，以下两个地方调用了外部模块

```python
# 141行
for arg in args:
    with open(arg, "r") as fh:
        events.extend(tracetool.read_events(fh, arg))
# 146行
tracetool.generate(events, arg_group, arg_format, arg_backends,                               binary=binary, probe_prefix=probe_prefix)

```

`args`是getopt以后剩下的参数，全部丢进`tracetool.readevents`获取反序列化的事件列表，`tracetool.generate`则接受所有处理过的参数生成文件。

##### 关于tracetool模块

`tracetool.readevents`将每一行非注释的文本转化为`Event`对象。

`tracetool.generate`将`binary=None, probe_prefix=None`丢给`tracetool.backend.dtrace`，其他参数向内传递给`tracetool.format.generate`，`tracetool.format.generate`检查`format`选项，并将剩余参数传递给`tracetool.format.$(format).generate`，具体内容就不分析了，如何使用我们可以直接看trace使用文档。

### trace的runtime配置

5.0.0版本以后`vl.c`被移动到了`softmmu`文件夹：

##### 虚拟机配置trace选项

+ 2884行添加了选项的描述

  ```c
  qemu_add_opts(&qemu_trace_opts);
  ```

  `qemu_trace_opts`是`trace/control.c`中的全局变量：

  ```c
  QemuOptsList qemu_trace_opts = {
      .name = "trace",
      .implied_opt_name = "enable",
      .head = QTAILQ_HEAD_INITIALIZER(qemu_trace_opts.head),
      .desc = {
          {
              .name = "enable",
              .type = QEMU_OPT_STRING,
          },
          {
              .name = "events",
              .type = QEMU_OPT_STRING,
          },{
              .name = "file",
              .type = QEMU_OPT_STRING,
          },
          { /* end of list */ }
      },
  };
  ```
  tracing.txt中这样描述

  > == Quickstart ==
    ...
    3. Run the virtual machine to produce a trace file:
        qemu --trace events=/tmp/events ... # your normal QEMU invocation
  
  生成的`build/qemu-options.def`中这样描述
  
  ```
  DEF("trace", HAS_ARG, QEMU_OPTION_trace,
  "-trace [[enable=]<pattern>][,events=<file>][,file=<file>]\n"
  "                specify tracing options\n",
  QEMU_ARCH_ALL)
  ```
  
  其中`file`选项会被转换成文件指针赋值给`trace_file`变量。

+ 3902行初始化`trace_backends`相关变量：

  ```c
  if (!trace_init_backends()) {
      exit(1);
  }
  trace_init_file(trace_file);
  ```

  `trace_backends`的初始化紧随`MACHINE`对象的初始化，处于初始化的尽可能早的位置。

  `trace_init_backends`是条件编译，如果配置的backend类型为`simple`，那么会调用`st_init`。

  st_init主要生效的是三个语句：

  ```c
  bool st_init(void)
  {
      trace_pid = getpid();
      GThread *thread = trace_thread_create(writeout_thread);
      atexit(st_flush_trace_buffer);
  }
  ```

  `trace_pid`用于区别进程，在`trace_file`后会附加这个值以作区分。

  启动了一个io线程用于写出trace记录，并在exit hook上添加了刷新文件的handler

##### trace io线程

读写端共享内存`trace_buf`，目前该内存大小固定为64KB无法修改。

每一个record对应这样一个内存分布

```c
/* * Trace buffer entry */
typedef struct {
    uint64_t event; /* event ID value */
    uint64_t timestamp_ns;
    uint32_t length;   /*    in bytes */
    uint32_t pid;
    uint64_t arguments[];
} TraceRecord;
```

`trace_buf`读端（IO线程侧）主要使用的是`get_trace_record(unsigned int idx, TraceRecord **recordptr)`函数。`get_trace_record`是这样工作的

+ 读取`idx`开始的event大小的字节，如果event段告诉我们这是一个无效事件（`TRACE_RECORD_VALID`位为0，则是一个无效事件）则停止读取。
+ 读取`idx`开始的TraceRecord大小的字节（为什么不只读8*3个字节？）
+ 根据刚刚读到的length，读取`idx`开始的length大小的字节（为什么不从`idx+8*3`开始读？）
+ 处理record，移动buf指针，并将copy出来的数据移交给写线程。

`trace_buf`写端主要涉及两个函数`int trace_record_start(TraceBufferRecord *rec, uint32_t event, size_t datasize)`和`void trace_record_finish(TraceBufferRecord *rec)`。`trace_record_start`负责写header，`trace_record_finish`负责修改该区域字节为`TRACE_RECORD_VALID`和一些输入输出的callback。

示例：

`dbus_vmstate_post_load(int version_id) "version_id: %d"`被编译成：

```c
void _simple_trace_dbus_vmstate_post_load(int version_id)
{
    TraceBufferRecord rec;
	
    // 如果不支持该事件trace，以及一些其他条件
    if (!trace_event_get_state(TRACE_DBUS_VMSTATE_POST_LOAD)) {
        return;
    }
	
    // 写入数据
    if (trace_record_start(&rec, _TRACE_DBUS_VMSTATE_POST_LOAD_EVENT.id, 8)) {
        return; /* Trace Buffer Full, Event Dropped ! */
    }
    trace_record_write_u64(&rec, (uint64_t)version_id);
    trace_record_finish(&rec);
}
```





