
## ubuntu下配置QEMU相关的Visual Studio Code环境

author: [Myriad-Dreamin](https://github.com/Myriad-Dreamin)

---

#### 添加include路径

创建`.vscode/c_cpp_properties.json`，并写入下面的内容：

```json
{
    "configurations": [
        {
            "name": "linux",
            "includePath": [
                "${workspaceFolder}",
                "${workspaceFolder}/include",
                "${workspaceFolder}/build",
                "/usr/include",
                "/usr/include/pixman-1",
                "/usr/include/glib-2.0",
                "/usr/lib/x86_64-linux-gnu/glib-2.0/include"
            ]
        }
    ],
    "version": 4
}
```

`qemu`使用了`glib`,`glib`的配置使用`pkg-config`获知，请在你的环境下运行下面的命令并将include路径添加到配置文件中：

```bash
 (qemu-venv) kamiyoru@kamiya:~/work/c/qemu$ pkg-config --cflags --libs glib-2.0 
-I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -lglib-2.0
```

这样阅读源码的时候暂时就不会报错了，如果还有其他错误，到时再修复。

#### 添加debug配置

创建`.vscode/launch.json`，并写入下面内容：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "QEMU x86_64 Linux",
            "type": "cppdbg",
            "request": "launch",
            "targetArchitecture": "x86_64",
            "program": "${workspaceRoot}/bin/qemu-system-x86_64d",
            "args": [
                "-kernel",
                "${workspaceRoot}/work/linux/bzImage",
                "-initrd",
                "${workspaceRoot}/work/linux/.config",
                "-smp",
                "2",
                "-gdb",
                "tcp::1234",
                "-S"
            ],
            "stopAtEntry": true,
            "cwd": "${workspaceRoot}",
            "externalConsole": true
        }
    ]
}
```

`${workspaceRoot}/bin/qemu-system-x86_64d`填编译好的qemu二进制文件路径。

`args`填给`qemu`的参数，这里启动的是编译好的linux内核。

按`ctrl+shift+D`选择`QEMU x86_64 Linux`任务运行程序。