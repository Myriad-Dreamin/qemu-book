author: [Myriad-Dreamin](https://github.com/Myriad-Dreamin)

---

## ubuntu中git的配置

#### 配置代理

假定你已经配置好了electron ssr配置，

查看`electron ssr > 配置 > 选项设置 > 通用设置`中你设置的局域网共享端口。一般所有的ssr都会选择`1080`端口作为网关。

运行下面两个命令

```bash
git config --add http.proxy socks5://127.0.0.1:1080
git config --add https.proxy socks5://127.0.0.1:1080
```

查看是否添加成功

```bash
kamiyoru@kamiya:~$ git config --get http.proxy 
socks5://127.0.0.1:1080
kamiyoru@kamiya:~$ git config --get https.proxy 
socks5://127.0.0.1:1080
```

重启终端，测试是否配置成功。