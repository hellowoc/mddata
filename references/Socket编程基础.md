# Socket 编程基础

Linux 网络编程的核心概念。本工程中通道1（FPGA 相机）和通道3（高速抓图）使用了 BSD socket，通道2（IPC 命令）使用了 Qt 封装的 QUdpSocket。

---

## sockaddr_in — "地址+端口"结构体

**头文件**：`<netinet/in.h>`

**是什么**：一个 C 结构体，把 IP 地址和端口号打包在一起，传给 `bind()`、`sendto()` 等函数。

**为什么叫这个名字**：**sock**et **addr**ess **in**ternet → "互联网 socket 地址结构体"。

**结构**：

```cpp
struct sockaddr_in {
    short          sin_family;   // 协议族（AF_INET = IPv4）
    unsigned short sin_port;     // 端口号
    struct in_addr sin_addr;     // IP 地址（嵌套的结构体）
    char           sin_zero[8];  // 填充字节（不用管）
};
// sin_addr 内部：
//     unsigned long s_addr;     // 32 位 IP 地址的二进制形式
```

**本工程中的典型用法**：

```cpp
// mynetwork.cpp:2408-2411 — 填充 + bind
memset(&addr_local, 0, sizeof(addr_local));         // 先清零
addr_local.sin_family = AF_INET;                    // IPv4
addr_local.sin_port = htons(8081);                  // 端口 8081
addr_local.sin_addr.s_addr = inet_addr("162.254.129.100");  // IP
bind(sockfd, (struct sockaddr*)&addr_local, sizeof(addr_local));
```

---

## AF_INET

**是什么**：一个常量，值为 2。表示 **IPv4 协议族**（Address Family INET）。

```cpp
socket(AF_INET, SOCK_DGRAM, 0);  // AF_INET 告诉操作系统"我要用 IPv4"
addr_local.sin_family = AF_INET; // 填 sockaddr_in 时也必须一致
```

对比：`AF_INET6` = IPv6，`AF_UNIX` = Unix 本地 socket。

---

## htons(x) / ntohs(x) — 字节序转换

**头文件**：`<arpa/inet.h>`

**是什么**：把 16 位整数在"主机字节序"和"网络字节序"之间转换。

**为什么需要**：不同 CPU 架构对多字节整数的存储顺序不同：
- 小端（x86/ARM）：低位在前。如 `0x1234` 存为 `34 12`
- 网络字节序（大端）：高位在前。如 `0x1234` 存为 `12 34`

用 `htons(8081)` 把 8081 转成网络序，确保无论什么 CPU 都能正确解读。

```cpp
// htons = Host TO Network Short (16位)
addr_local.sin_port = htons(8081);

// inet_addr = 把字符串 IP 转成 32 位网络字节序整数
addr_local.sin_addr.s_addr = inet_addr("162.254.129.100");
```

---

## inet_addr(str) — 字符串 IP → 二进制

**头文件**：`<arpa/inet.h>`

**签名**：`in_addr_t inet_addr(const char *cp);`

**作用**：把 `"192.168.1.100"` 这样的字符串转为 `sockaddr_in.sin_addr.s_addr` 能用的 32 位二进制值。

```cpp
inet_addr("162.254.129.100")  →  0x6481FE0A（32位网络序）
```

---

## inet_pton(family, str, dst) — 新版 IP 字符串转换

**头文件**：`<arpa/inet.h>`

**签名**：`int inet_pton(int af, const char *src, void *dst);`

**作用**：和 `inet_addr` 一样，但支持 IPv4 和 IPv6。比 `inet_addr` 更现代。

```cpp
// mynetwork.cpp:2423
inet_pton(AF_INET, "162.254.129.10", &addr_remote.sin_addr);
// 把 "162.254.129.10" 写入 addr_remote.sin_addr
```

---

## socket() — 创建 socket

**头文件**：`<sys/socket.h>`

**签名**：`int socket(int domain, int type, int protocol);`

**作用**：创建一个 socket，返回**文件描述符**（一个整数）。

```cpp
sockfd = socket(AF_INET, SOCK_DGRAM, 0);
//          ↑        ↑           ↑
//      IPv4      UDP协议    自动选择
//               SOCK_STREAM = TCP
```

**socket 是什么**：socket 是操作系统提供的一个**抽象句柄**。你拿着这个整数，就可以用 `bind/sendto/recvfrom` 等函数操作网络连接。在 Linux 中，socket 和文件操作使用同一套接口——`read/write/close` 也可以用于 socket。

---

## bind() — 绑定地址

**头文件**：`<sys/socket.h>`

**签名**：`int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`

**作用**：把 socket 和某个 IP+端口绑定。意思是"这个地址归我了，发到这里的包都给我"。

```cpp
bind(sockfd, (struct sockaddr*)&addr_local, sizeof(addr_local));
// 绑定到 162.254.129.100:8081
```

**类比**：去邮局登记"162.254.129.100 的 8081 号信箱是我的，以后寄到这个信箱的信我来取"。

---

## select() — 检查 socket 是否有数据可读

**头文件**：`<sys/select.h>` 或 `<sys/time.h>`

**签名**：`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);`

**作用**：监听一个或多个文件描述符，等待其中任何一个"就绪"（可读/可写/出错）。可以设置超时。

**为什么用它**：`recvfrom()` 是阻塞的——如果没数据就一直等。先用 `select()` 确认有数据，再用 `recvfrom()` 读取，避免线程卡死。

**本工程中的用法**：

```cpp
// mynetwork.cpp:1146 — checkUdpSocketStatues(0) 内部调用了 select()
fd_set fds;
FD_ZERO(&fds);                // 清空集合
FD_SET(sockfd, &fds);         // 把 sockfd 加入监控集合

struct timeval tv;
tv.tv_sec = 0;                // 超时 0 秒
tv.tv_usec = 0;               // 超时 0 微秒 → 相当于"看一眼有没有数据，没有马上返回"

int ret = select(sockfd + 1, &fds, NULL, NULL, &tv);
// ret > 0 → socket 可读
// ret = 0 → 超时，没有数据
// ret < 0 → 出错
```

---

## recvfrom() — 阻塞接收 UDP 数据

**头文件**：`<sys/socket.h>`

**签名**：`ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);`

**作用**：从 socket 读取数据。**阻塞调用**——没数据到就一直等。

```cpp
// mynetwork.cpp:1150
retNum = recvfrom(sockfd,                          // 从哪个 socket 读
                  (char*)&wavePacketBuf[curTotal],  // 读到哪个缓冲区
                  waveBufSize - curTotal,            // 最多读多少字节
                  0,                                 // flags（默认0）
                  (struct sockaddr*)&client,         // [出参] 发送方的地址
                  &sin_size);                        // 地址结构体大小
```

**返回值**：实际读到的字节数。`-1` 表示出错。

---

## sendto() — 发送 UDP 数据

**头文件**：`<sys/socket.h>`

**签名**：`ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);`

**作用**：通过 socket 发送数据到指定地址。

```cpp
sendto(sockfd, buf, len, 0, 
       (struct sockaddr*)&addr_remote,    // 目标地址
       sizeof(addr_remote));
```

---

## setsockopt() — 设置 socket 选项

**头文件**：`<sys/socket.h>`

**作用**：调整 socket 的底层行为。

```cpp
// SO_REUSEADDR — 允许端口重用
int val = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val));

// SO_RCVBUF — 设置接收缓冲区大小（8MB）
int val = 8 * 1024 * 1024;
setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &val, sizeof(val));
```

**SO_REUSEADDR 为什么重要**：程序关闭后，端口可能进入 `TIME_WAIT` 状态（等 2 分钟才能重新使用）。加了 `SO_REUSEADDR` 后可以立即重用端口——这在调试时反复启停程序非常关键。

---

## 完整 Socket 生命周期

```
1. socket()    创建 socket，拿到文件描述符 sockfd
2. setsockopt() 设置选项（重用地址、增大缓冲）
3. bind()      绑定本机 IP+端口（"我占这个地址了"）
4. sendto()    发送数据到远端
5. recvfrom()  从远端接收数据
6. close()     关闭 socket
```
