---
title: 用朴素的方式重新理解 Socket
date: 2024-04-02 02:16:59
tags: [java, socket]
---

用朴素的方式重新理解 `Socket`。

<!-- more -->

## Socket

一个简单的 `SocketServer`。

```java
public class SocketServer {

    private static final int PORT = 50218;

    public static void main(String[] args) throws IOException {
        try (ServerSocket server = new ServerSocket(PORT)) {
            System.out.println("服务端已启动 " + server);
            new Scanner(System.in).nextLine();
        }
    }
}
```

编译并运行程序。

```bash
$ javac SocketServer.java
$ java SocketServer
服务端已启动 ServerSocket[addr=0.0.0.0/0.0.0.0,localport=50218]
```

很简单的一段代码 `new ServerSocket(PORT)` 就完成了对指定端口的绑定和监听。

> 在很久以后，我才认识到，对指定端口的绑定和监听可以说是几乎完全独立于其他事情的一项工作。

**你可以使用 `ss` 查看 `socket` 的信息**。

```bash
$ ss -tlnp | grep 50218
LISTEN 0      50                               *:50218            *:*    users:(("java",pid=156420,fd=6))
```

返回结果表示当前 `Java` 进程（`PID` 为 `156420`）正在对所有本地 `IPv4/IPv6` 地址的 `50218` 端口进行网络监听（`Listen`），并已经准备好接收外部连接。


| 字段 | 含义 | 解释 |
| --- | --- | --- |
| `LISTEN` | 当前 `Socket` 的状态 | 监听状态。说明该服务正在运行，等待客户端发起连接。 |
| `0` | `Send-Q`（发送队列） | 在 `LISTEN` 状态下，表示当前已经完成三次握手、进入全连接队列（`Accept Queue`）等待进程处理的连接数。当前为 `0`，说明没有积压的连接。 |
| `50` | `Recv-Q`（接收队列） | 在 `LISTEN` 状态下，表示该端口能够容纳的最大全连接队列长度（`Backlog`）。此端口最多可同时积压 `50` 个等待 `accept()` 的连接。 |
| `*:50218` | 本地地址与端口 | `*` 代表 `0.0.0.0` 或 `[::]`（绑定了所有网卡接口）。外部网络和本地网络都可以通过 `50218` 端口访问该服务。 |
| `*:*`  | 远程地址与端口 | 由于处于监听（`LISTEN`）阶段，尚未与特定远程主机建立连接，因此显示为通配符。 |
| `users:(...)` | 进程详细信息 | `Java` 服务：打开该端口的进程是 `Java`。进程号 (`PID`)：`42178`。文件描述符 (`fd`)：`6`，系统分配的网络套接字句柄。 |

**你还可以使用 `ps` 查看该 `PID` 对应的完整启动参数**，比如它加载了哪个 `Jar` 包，或者指定了哪个主类。

```bash
$ ps -ef | grep 156420
root      156420  156331  0 03:18 pts/3    00:00:00 java SocketServer
root      157054  155682  0 03:20 pts/2    00:00:00 grep 156420
```

你现在就可以使用 `telnet` 和 `Server` 建立连接。

```bash
telnet 100.82.194.77 50218
```

可以看到在连接建立后，`Send-Q` 从 `0` 变成 `1`。

```bash
$ ss -tlnp | grep :50218
LISTEN 1      50                               *:50218            *:*    users:(("java",pid=156420,fd=6))
```

结合 `watch` 和 `ss` 可以持续查看 `Socket` 信息。

```bash
$ watch -n 1 "ss -ant | grep :50218"
Every 1.0s: ss -ant | grep :50218                                debian: Thu Oct 15 18:33:01 2026

LISTEN     1       50                                 *:50218                           *:*
ESTAB      0       0             [::ffff:100.82.194.77]:50218    [::ffff:100.123.104.120]:54077
```

你还可以编写程序和 `Server` 建立连接。

```java
public class SocketClient {

    private static final String HOST = "100.82.194.77";
    private static final int PORT = 50218;

    public static void main(String[] args) throws IOException {
        try (Socket socket = new Socket(HOST, PORT)) {
            System.out.println("客户端已建立连接 " + socket);
            new Scanner(System.in).nextLine();
        }
    }
}
```

编译并运行程序。成功建立的 `Socket` 使用 `localport` 为 `46016`。

```bash
$ javac SocketClient.java
$ java SocketClient
客户端已建立连接 Socket[addr=/100.82.194.77,port=50218,localport=47550]
```

可以看到 `Send-Q` 再次增加变成 `2`，显示 `2` 条建立的连接（`ESTAB`）。

```bash
$ watch -n 1 "ss -ant | grep :50218"
Every 1.0s: ss -ant | grep :50218                                debian: Thu Oct 15 20:10:00 2026

LISTEN     2       50                                 *:50218                           *:*
ESTAB      0       0             [::ffff:100.82.194.77]:50218    [::ffff:100.123.104.120]:54077
ESTAB      0       0             [::ffff:100.82.194.77]:50218    [::ffff:100.123.104.120]:47550
```

## Backlog

注意到 `Recv-Q` 的大小为 `50`，对应着 `ServerSocket` `backlog` 的默认值 `50`。

```java
public ServerSocket(int port) throws IOException {
    this(port, 50, null);
}
```

实测可知，同时最多可建立 `backlog + 1` 条连接。

```java
public class BulkSocketClient {

    private static final String HOST = "100.82.194.77";
    private static final int PORT = 50218;
    private static final int DEFAULT_BACKLOG = 50;

    public static void main(String[] args) throws IOException {
        List<Socket> list = new ArrayList<>();
        for (int i = 0; i < DEFAULT_BACKLOG + 1; i++) {
            Socket socket = new Socket(HOST, PORT);
            list.add(socket);
        }
        new Scanner(System.in).nextLine();
    }
}
```

可以看到 `Send-Q` 变成 `51`，显示 `51` 条建立的连接（`ESTAB`）。

```bash
$ ss -ant | grep :50218
LISTEN    51     50                               *:50218                        *:*
ESTAB     0      0           [::ffff:100.82.194.77]:50218 [::ffff:100.123.104.120]:44686
ESTAB     0      0           [::ffff:100.82.194.77]:50218 [::ffff:100.123.104.120]:48672
……
ESTAB     0      0           [::ffff:100.82.194.77]:50218 [::ffff:100.123.104.120]:44786
ESTAB     0      0           [::ffff:100.82.194.77]:50218 [::ffff:100.123.104.120]:47318
```

修改 `backlog` 为 `10` 进行测试。

```java
public class BacklogSocketServer {

    private static final int PORT = 50218;
    private static final int BACKLOG = 10;

    public static void main(String[] args) throws IOException {
        try (ServerSocket server = new ServerSocket(PORT, BACKLOG)) {
            System.out.println("服务端已启动 " + server);
            new Scanner(System.in).nextLine();
        }
    }
}
```

可以看到 `Recv-Q` 的大小变成 `10`，

```bash
$ ss -ant | grep :50218
LISTEN    0      10                               *:50218                       *:*
```

和先前一样尝试建立 `51` 条连接，可以看到 `Send-Q` 最终数量是 `11`，显示 `11` 条建立的连接（`ESTAB`）。

```bash
$ watch -n 1 "ss -ant | grep :50218"
Every 1.0s: ss -ant | grep :50218                                debian: Fri Oct 16 01:20:10 2026

LISTEN     11      10                                  *:50218                            *:*
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:46850
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:46838
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:47896
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:46892
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:47918
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:46864
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:46880
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:47932
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:47910
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:46848
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:46874
```

如果网络通畅，过程中可以观察到除了率先建立的 `11` 条连接，还有很多连接处于 `SYN-RECV` 状态。

<div style="width:90%;margin:auto">{% asset_img "Snipaste_2026-10-15_10-18-10.png" 客户端连接请求大于 backlog %}</div>


## BindAddr

当服务器拥有多个 `IP`，而 `Java` 进程又对所有本地 `IPv4/IPv6` 地址的 `50218` 端口进行网络监听时，客户端可以通过不同 `IP` 与服务端建立连接。

```bash
$ watch -n 1 "ss -ant | grep :50218"
Every 1.0s: ss -ant | grep :50218                                debian: Fri Oct 16 01:29:13 2026

LISTEN     2       50                                  *:50218                            *:*
ESTAB      0       0              [::ffff:100.82.194.77]:50218     [::ffff:100.123.104.120]:57609
ESTAB      0       0              [::ffff:172.28.67.181]:50218     [::ffff:115.196.202.124]:54449
```

可以对特定 `IP` 地址进行网络监听。

```java
public class BindAddrSocketServer {

    private static final int PORT = 50218;
    private static final int DEFAULT_BACKLOG = 50;
    private static final String IP = "172.28.67.181";

    public static void main(String[] args) throws IOException {
        InetAddress bindAddr = InetAddress.getByName(IP);
        try (ServerSocket server = new ServerSocket(PORT, DEFAULT_BACKLOG, bindAddr)) {
            System.out.println("服务端已启动 " + server);
            new Scanner(System.in).nextLine();
        }
    }
}
```

编译并运行程序。`Java` 进程对 `172.28.67.181` 的 `50218` 进行网络监听。

```bash
$ javac BindAddrSocketServer.java
$ java BindAddrSocketServer
服务端已启动 ServerSocket[addr=/172.28.67.181,localport=50218]
```

```bash
$ watch -n 1 "ss -ant | grep :50218"
Every 1.0s: ss -ant | grep :50218                                debian: Fri Oct 16 02:18:19 2026

LISTEN  0        50              [::ffff:172.28.67.181]:50218                           *:*
```

执行命令 `telnet 172.28.67.181 50218`，成功建立连接。

```bash
$ watch -n 1 "ss -ant | grep :50218"
Every 1.0s: ss -ant | grep :50218                                debian: Fri Oct 16 02:18:50 2026

LISTEN     1       50             [::ffff:172.28.67.181]:50218                            *:*
ESTAB      0       0              [::ffff:172.28.67.181]:50218     [::ffff:115.196.202.124]:49814
```

使用服务器的其他 `IP` 尝试建立连接，失败。

```bash
$ telnet 100.82.194.77 50218
Connecting To 100.82.194.77...Could not open connection to the host, on port 50218: Connect failed
```
