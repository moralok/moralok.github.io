---
title: docker.sock 的测试和本质
date: 2026-10-26 09:43:59
tags: [docker]
---

在使用 `Docker` 的过程中，总会看到 `/var/run/docker.sock` 的身影。`docker` 是客户端程序，`dockerd` 是守护进程，两者的通信需要通过 `docker.sock`。

- 它究竟是什么？
- 为什么运行 `docker` 命令需要 `root` 权限或加入 `docker` 组？
- 为什么说映射 `docker.sock` 到容器中是不安全的行为？

<!-- more -->

## docker.sock 的测试

**如何手动、朴素地和 `docker.sock` 通信，戳破它的神秘感？**

### 使用第三方工具

#### 方法 A：使用 curl

`curl` 原生支持通过参数指定 `Unix Socket` 文件，并手动构造 `HTTP` 请求：

```bash
curl --unix-socket /var/run/docker.sock http://localhost/containers/json
```

参数说明：

- `--unix-socket /var/run/docker.sock`：**掐断网卡和 `TCP` 协议栈**，直接把数据打包塞进这个本地文件对应的内核内存里
- `http://localhost` 是一个占位符（因为 `UDS` 不需要 `IP`），而 `/containers/json` 就是 `Docker` 官方定义的“列出正在运行的容器”的 `HTTP REST API` 路由

返回结果：

```console
[{"Id":"919cacb14bff1838b3cb0c3c1332f53d0692c89018d5282571f497594171fe47","Names":["/reverent_meninsky"],"Image":"nginx","ImageID":"sha256:4c41e2d0a3a55b325ed572cd0200bdb72af1433c3daa2bca6ef8eea68ce504a2","ImageManifestDescriptor":{"mediaType":"application/vnd.oci.image.manifest.v1+json","digest":"sha256:59a9ae22b8933b536391e6c1708fa761297e922809c97675c5a56ba316be62af","size":2290,"annotations":{"com.docker.official-images.bashbrew.arch":"amd64","org.opencontainers.image.base.digest":"sha256:4876fdea29cac1a33a3195f1eb1952feb1fca7dd6dd00be3e2f1879cf20d34d2","org.opencontainers.image.base.name":"debian:trixie-slim","org.opencontainers.image.created":"2026-10-21T22:16:50Z","org.opencontainers.image.revision":"6fabb4c960a02a1e576d2789974c45f4cd2aa854","org.opencontainers.image.source":"https://github.com/nginx/docker-nginx.git#6fabb4c960a02a1e576d2789974c45f4cd2aa854:mainline/debian","org.opencontainers.image.url":"https://hub.docker.com/_/nginx","org.opencontainers.image.version":"1.31.8"},"platform":{"architecture":"amd64","os":"linux"}},"Command":"/docker-entrypoint.sh nginx -g 'daemon off;'","Created":1793025621,"Ports":[{"IP":"","PrivatePort":80,"Type":"tcp"}],"Labels":{"maintainer":"NGINX Docker Maintainers <docker-maint@nginx.com>"},"State":"running","Status":"Up 4 minutes","HostConfig":{"NetworkMode":"bridge"},"Health":{"Status":"none","FailingStreak":0},"NetworkSettings":{"Networks":{"bridge":{"IPAMConfig":null,"Links":null,"Aliases":null,"DriverOpts":null,"GwPriority":0,"NetworkID":"7990194027e5bd07a1a19a1e045adb037312e1a29c65d69153c5a94f29548d0b","EndpointID":"2377548cf9d62c084155f0873b3b56d46faf2e89dbe21da0a468049748cce234","Gateway":"172.17.0.1","IPAddress":"172.17.0.2","MacAddress":"c2:4e:9e:85:67:75","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"DNSNames":null}}},"Mounts":[]}]
```

#### 方法B：使用 nc（Netcat）

1. 连接 `Socket` 文件：
	```bash
	nc -U /var/run/docker.sock
	```
2. 手动输入 `HTTP` 请求报文（两次回车发送）
	```console
	GET /containers/json HTTP/1.1
	Host: docker.sock
	```
3. 返回结果
	```console
	HTTP/1.1 200 OK
	Api-Version: 1.56
	Content-Type: application/json
	Docker-Experimental: false
	Ostype: linux
	Server: Docker/29.9.2 (linux)
	Date: Mon, 26 Oct 2026 14:51:21 GMT
	Content-Length: 1856
	
	[{"Id":"919cacb14bff1838b3cb0c3c1332f53d0692c89018d5282571f497594171fe47","Names":["/reverent_meninsky"],"Image":"nginx","ImageID":"sha256:4c41e2d0a3a55b325ed572cd0200bdb72af1433c3daa2bca6ef8eea68ce504a2","ImageManifestDescriptor":{"mediaType":"application/vnd.oci.image.manifest.v1+json","digest":"sha256:59a9ae22b8933b536391e6c1708fa761297e922809c97675c5a56ba316be62af","size":2290,"annotations":{"com.docker.official-images.bashbrew.arch":"amd64","org.opencontainers.image.base.digest":"sha256:4876fdea29cac1a33a3195f1eb1952feb1fca7dd6dd00be3e2f1879cf20d34d2","org.opencontainers.image.base.name":"debian:trixie-slim","org.opencontainers.image.created":"2026-10-21T22:16:50Z","org.opencontainers.image.revision":"6fabb4c960a02a1e576d2789974c45f4cd2aa854","org.opencontainers.image.source":"https://github.com/nginx/docker-nginx.git#6fabb4c960a02a1e576d2789974c45f4cd2aa854:mainline/debian","org.opencontainers.image.url":"https://hub.docker.com/_/nginx","org.opencontainers.image.version":"1.31.8"},"platform":{"architecture":"amd64","os":"linux"}},"Command":"/docker-entrypoint.sh nginx -g 'daemon off;'","Created":1793025621,"Ports":[{"IP":"","PrivatePort":80,"Type":"tcp"}],"Labels":{"maintainer":"NGINX Docker Maintainers <docker-maint@nginx.com>"},"State":"running","Status":"Up 10 minutes","HostConfig":{"NetworkMode":"bridge"},"Health":{"Status":"none","FailingStreak":0},"NetworkSettings":{"Networks":{"bridge":{"IPAMConfig":null,"Links":null,"Aliases":null,"DriverOpts":null,"GwPriority":0,"NetworkID":"7990194027e5bd07a1a19a1e045adb037312e1a29c65d69153c5a94f29548d0b","EndpointID":"2377548cf9d62c084155f0873b3b56d46faf2e89dbe21da0a468049748cce234","Gateway":"172.17.0.1","IPAddress":"172.17.0.2","MacAddress":"c2:4e:9e:85:67:75","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"DNSNames":null}}},"Mounts":[]}]
	```

### docker 指令的本质

以 `docker ps` 为例：

1. 输入 `docker ps`
2. 终端执行 `/usr/bin/docker` 二进制程序。（这是一个 `Go` 语言编写的客户端 `CLI` 工具）
3. `CLI` 工具读取环境变量（默认寻找 `/var/run/docker.sock` 文件）
4. `CLI` 工具在内存中组装出一个标准的 `HTTP` 报文
	```console
	GET /containers/json HTTP/1.1\r\nHost: docker.sock\r\n\r\n
	```
5. `CLI` 工具通过内核管道，将这些字符写入 `/var/run/docker.sock`
6. `dockerd` 的内置 `HTTP` 路由解析器发现是 `GET /containers/json`，于是调用内部函数，向 `Linux` 内核（`Cgroups` 和 `Namespaces`）查询当前有哪些隔离进程在运行
7. `dockerd` 将查询到的结构体格式化为一段标准 `JSON` 文本，包裹在 `HTTP` 响应报文中返回
8. `CLI` 工具收到响应后，解析并显示到 `console`

### 使用编写的程序

#### 使用 `Java` 编写程序进行读写测试。

> `Java 16+` 官方原生引入了对 `Unix Domain Socket` 的支持。

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.net.StandardProtocolFamily;
import java.net.UnixDomainSocketAddress;

public class DockerSockTest {
    public static void main(String[] args) throws IOException {
        // 1. 明确告诉内核：我要创建一个 Unix 域套接字（而不是普通的 TCP 套接字）
        UnixDomainSocketAddress address = UnixDomainSocketAddress.of("/var/run/docker.sock");
        SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);

        // 2. 像连接服务器端口一样，连接这个套接字文件
        channel.connect(address);
        System.out.println("成功连接到底层 docker.sock 文件！");

        // 3. 纯手工拼接标准的 HTTP 请求原始报文（模拟 docker ps 的底层行为）
        String httpRequest = "GET /containers/json HTTP/1.1\r\n" +
                             "Host: docker.sock\r\n" +
                             "Connection: close\r\n\r\n"; // 两次回车代表请求头结束

        // 4. 将字符串转为字节，直接往内核管道里写数据
        ByteBuffer writeBuffer = ByteBuffer.wrap(httpRequest.getBytes());
        channel.write(writeBuffer);

        // 5. 循环读取内核从管道另一端吐回来的 HTTP 响应文本
        System.out.println("--- 收到来自 Docker 守护进程的原始 HTTP 响应 ---");
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        while (channel.read(readBuffer) != -1) {
            readBuffer.flip();
            // 直接将字节转换为文本打印在屏幕上
            System.out.print(new String(readBuffer.array(), 0, readBuffer.limit()));
            readBuffer.clear();
        }

        channel.close();
    }
}
```

测试结果：

```console
成功连接到底层 docker.sock 文件！
--- 收到来自 Docker 守护进程的原始 HTTP 响应 ---
HTTP/1.1 200 OK
Api-Version: 1.56
Content-Type: application/json
Docker-Experimental: false
Ostype: linux
Server: Docker/29.9.2 (linux)
Date: Mon, 26 Oct 2026 15:18:42 GMT
Content-Length: 1856
Connection: close

[{"Id":"919cacb14bff1838b3cb0c3c1332f53d0692c89018d5282571f497594171fe47","Names":["/reverent_meninsky"],"Image":"nginx","ImageID":"sha256:4c41e2d0a3a55b325ed572cd0200bdb72af1433c3daa2bca6ef8eea68ce504a2","ImageManifestDescriptor":{"mediaType":"application/vnd.oci.image.manifest.v1+json","digest":"sha256:59a9ae22b8933b536391e6c1708fa761297e922809c97675c5a56ba316be62af","size":2290,"annotations":{"com.docker.official-images.bashbrew.arch":"amd64","org.opencontainers.image.base.digest":"sha256:4876fdea29cac1a33a3195f1eb1952feb1fca7dd6dd00be3e2f1879cf20d34d2","org.opencontainers.image.base.name":"debian:trixie-slim","org.opencontainers.image.created":"2026-10-21T22:16:50Z","org.opencontainers.image.revision":"6fabb4c960a02a1e576d2789974c45f4cd2aa854","org.opencontainers.image.source":"https://github.com/nginx/docker-nginx.git#6fabb4c960a02a1e576d2789974c45f4cd2aa854:mainline/debian","org.opencontainers.image.url":"https://hub.docker.com/_/nginx","org.opencontainers.image.version":"1.31.8"},"platform":{"architecture":"amd64","os":"linux"}},"Command":"/docker-entrypoint.sh nginx -g 'daemon off;'","Created":1793025621,"Ports":[{"IP":"","PrivatePort":80,"Type":"tcp"}],"Labels":{"maintainer":"NGINX Docker Maintainers <docker-maint@nginx.com>"},"State":"running","Status":"Up 38 minutes","HostConfig":{"NetworkMode":"bridge"},"Health":{"Status":"none","FailingStreak":0},"NetworkSettings":{"Networks":{"bridge":{"IPAMConfig":null,"Links":null,"Aliases":null,"DriverOpts":null,"GwPriority":0,"NetworkID":"7990194027e5bd07a1a19a1e045adb037312e1a29c65d69153c5a94f29548d0b","EndpointID":"2377548cf9d62c084155f0873b3b56d46faf2e89dbe21da0a468049748cce234","Gateway":"172.17.0.1","IPAddress":"172.17.0.2","MacAddress":"c2:4e:9e:85:67:75","IPPrefixLen":16,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"DNSNames":null}}},"Mounts":[]}]
```

#### 使用 Python 编写程序进行读写测试

```python
import socket

# 1. 创建一个 Unix Domain Socket (AF_UNIX) 类型的套接字
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)

# 2. 建立连接（像连接 IP 端口一样连接这个本地文件路径）
s.connect("/var/run/docker.sock")

# 3. 纯手工发送符合标准 HTTP/1.1 规范的文本请求（两次回车代表请求结束）
s.sendall(b"GET /containers/json HTTP/1.1\r\nHost: localhost\r\n\r\n")

# 4. 接收并打印来自 Docker 守护进程返回的原始 HTTP 响应和 JSON 数据
print(s.recv(4096).decode("utf-8"))
```

## docker.sock 的本质

### 是一个 UDS

`docker.sock` 是一个 `Unix` 域套接字（`Unix Domain Socket`），简称 `UDS`。

在 Linux 操作系统中，一切皆文件。`/var/run/docker.sock` 在文件系统中表现为一个**特殊的套接字文件**。

- 不走网络协议栈
- 纯内存进程间通信（`IPC，Inter-Process Communication`，进程间通信）
- 监听进程 `dockerd`

`UDS` 的本质是 **Linux 内核在内存中开辟的一个高效率、带文件权限的 IPC 管道**。

它虽然在文件系统中表现为一个文件（`/var/run/docker.dock`），但不能像打开普通文本文件一样用 `cat` 或 `echo` 去读写它。这个文件在文件系统中只是一个挂载点（入口标识符），它的真实作用有两个：
1. 权限防火墙：利用 Linux 的文件权限机制。谁有这个文件的读写权，谁就能和对应的内核内存管道通信。
2. 进程寻址：让客户端进程能够通过一个路径找到服务端进程。

而 `docker.sock` 的权限如下：

```bash
ls -al /var/run/docker.sock
```

`owner` 为 `root`，`group` 为 `docker`，两者都有读写权限。

```console
srw-rw---- 1 root docker 0 Oct 26 22:37 /var/run/docker.sock
```

### 查看 UDS 信息

**如何根据 UDS 文件名，查找正在监听它的真实进程 `ID`？**

#### 方法 A：使用 ss

`ss` 是现代 Linux 系统中用来查看套接字状态的终极工具（替代 netstat）

```bash
sudo ss -xpl | grep docker.sock
```

返回结果：

```console
u_str LISTEN 0      4096      /run/docker.sock 1510334      * 0    users:(("dockerd",pid=669273,fd=5),("systemd",pid=1,fd=76))
```

#### 方法 B：使用 lsof

> `lsof` 意为 `List Open Files`

```bash
sudo lsof /var/run/docker.sock
```

返回结果：

```console
COMMAND    PID USER FD   TYPE             DEVICE SIZE/OFF    NODE NAME
systemd      1 root 76u  unix 0x0000000094dea39e      0t0 1510334 /run/docker.sock type=STREAM (LISTEN)
dockerd 669273 root  5u  unix 0x0000000094dea39e      0t0 1510334 /run/docker.sock type=STREAM (LISTEN)
```

#### 查看进程信息

```bash
ps -fp 669273
```

返回结果：

```console
UID          PID    PPID  C STIME TTY          TIME CMD
root      669273       1  0 22:37 ?        00:00:02 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

## 权限的重要性

在 `Linux` 内核眼里，没有权限高低的客户端工具，只有是否持有 `Socket` 文件读写权限的进程。如果被入侵，黑客可以利用简单的 `POST` 请求去实现容器创建或宿主机提权。