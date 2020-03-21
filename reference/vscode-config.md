
## ubuntu下配置QEMU相关的Visual Studio Code环境

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