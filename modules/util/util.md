### qemu-error.c/error-report.h

实现了`include/qemu/error-report.h`



###### log函数系列

顾名思义系列：

```c
int error_vprintf(const char *fmt, va_list ap) GCC_FMT_ATTR(1, 0);
int error_printf(const char *fmt, ...) GCC_FMT_ATTR(1, 2);
int error_vprintf_unless_qmp(const char *fmt, va_list ap) GCC_FMT_ATTR(1, 0);
int error_printf_unless_qmp(const char *fmt, ...) GCC_FMT_ATTR(1, 2);

void error_vreport(const char *fmt, va_list ap) GCC_FMT_ATTR(1, 0);
void warn_vreport(const char *fmt, va_list ap) GCC_FMT_ATTR(1, 0);
void info_vreport(const char *fmt, va_list ap) GCC_FMT_ATTR(1, 0);

void error_report(const char *fmt, ...) GCC_FMT_ATTR(1, 2);
void warn_report(const char *fmt, ...) GCC_FMT_ATTR(1, 2);
void info_report(const char *fmt, ...) GCC_FMT_ATTR(1, 2);

bool error_report_once_cond(bool *printed, const char *fmt, ...)
    GCC_FMT_ATTR(2, 3);
bool warn_report_once_cond(bool *printed, const char *fmt, ...)
    GCC_FMT_ATTR(2, 3);

void error_init(const char *argv0);

#define error_report_once(fmt, ...) ...
#define warn_report_once(fmt, ...) ...

const char *error_get_progname(void);
```

```c
static void qemu_log_func(const gchar *log_domain,
                          GLogLevelFlags log_level,
                          const gchar *message,
                          gpointer user_data)
void error_init(const char *argv0)
```

### error-report.h

```c
extern bool enable_timestamp_msg;
```

### qemu-option.c

#### QemuOpts/QemuOpt/QemuOptsList

```c
enum QemuOptType {
    QEMU_OPT_STRING = 0,  /* no parsing (use string as-is)                        */
    QEMU_OPT_BOOL,        /* on/off                                               */
    QEMU_OPT_NUMBER,      /* simple number                                        */
    QEMU_OPT_SIZE,        /* size, accepts (K)ilo, (M)ega, (G)iga, (T)era postfix */
};

typedef struct QemuOptDesc {
    const char *name;
    enum QemuOptType type;
    const char *help;
    const char *def_value_str;
} QemuOptDesc;

struct QemuOptsList {
    const char *name;
    const char *implied_opt_name;
    bool merge_lists;  /* Merge multiple uses of option into a single list? */
    QTAILQ_HEAD(, QemuOpts) head;
    QemuOptDesc desc[];
};

struct QemuOpt {
    char *name;
    char *str;

    const QemuOptDesc *desc;
    union {
        bool boolean;
        uint64_t uint;
    } value;

    QemuOpts     *opts;
    QTAILQ_ENTRY(QemuOpt) next;
};

struct QemuOpts {
    char *id;
    QemuOptsList *list;
    Location loc;
    QTAILQ_HEAD(, QemuOpt) head;
    QTAILQ_ENTRY(QemuOpts) next;
};
```

```c
const char *qemu_opt_get(QemuOpts *opts, const char *name)
```

###### `const char *qemu_opt_get(QemuOpts *opts, const char *name)`

根据`option`名字寻找`option`的值，如果没有该值，返回`opts`记录的默认值，如果都没有返回`NULL`