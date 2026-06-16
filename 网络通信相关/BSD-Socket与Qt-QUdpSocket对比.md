# BSD Socket 与 Qt QUdpSocket 对比

## 定位

| | BSD Socket | Qt QUdpSocket |
|------|-----------|-------------|
| 层级 | 系统调用层（C 标准 API） | 框架层（C++ Qt 封装） |
| 头文件 | `<sys/socket.h>` `<netinet/in.h>` `<arpa/inet.h>` | `<QUdpSocket>` `<QHostAddress>` |
| 依赖 | 仅 Linux 内核 | Qt Core 模块 |
| 跨平台 | Linux/Unix 原生，Windows 需 WinSock 适配 | Qt 支持的所有平台统一 API |

## 底层封装关系

```
Qt QUdpSocket
  └── QAbstractSocket (Qt 基类)
        └── QIODevice (统一读写接口)
              └── QObject (信号槽、事件循环)
                    └── 内部持有 BSD Socket fd
                          └── Linux 内核协议栈 (UDP/IP)
```

Qt QUdpSocket 底层也是 BSD Socket。它在 BSD Socket 的 fd 上包了一层 QObject 的事件循环机制——`readyRead` 信号的触发本质上是 Qt 内部事件分发器检测到 fd 可读后发射的。

## API 对比

### 创建

```cpp
// BSD Socket — 返回文件描述符（int）
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
// AF_INET: IPv4, SOCK_DGRAM: UDP, 0: 自动选择协议

// Qt — 返回 QObject 指针
QUdpSocket *socket = new QUdpSocket(parent);
// parent: 内存管理，parent 析构时自动 delete
```

### 绑定

```cpp
// BSD Socket — 填充 sockaddr_in 结构体
struct sockaddr_in addr_local;
memset(&addr_local, 0, sizeof(addr_local));
addr_local.sin_family = AF_INET;
addr_local.sin_port = htons(8081);                    // 端口，需手动字节序转换
addr_local.sin_addr.s_addr = inet_addr("162.254.129.100"); // IP，字符串→二进制
bind(sockfd, (struct sockaddr*)&addr_local, sizeof(addr_local));

// Qt — 直接传 QString
socket->bind(QHostAddress("162.254.129.100"), 8081);  // 自动处理字节序和格式
```

### 发送

```cpp
// BSD Socket — 原始字节指针
char buf[512];
sendto(sockfd, buf, len, 0, (struct sockaddr*)&addr_remote, sizeof(addr_remote));
//            ↑ fd  ↑ 数据指针  ↑ 长度

// Qt — QByteArray 自动管理
QByteArray data;
data.append("key=value");
socket->writeDatagram(data, QHostAddress("162.254.129.10"), 5001);
//                     ↑ QByteArray 内部自动取 data() 和 size()
```

### 接收通知

```cpp
// BSD Socket — 轮询：手动调用 select() 检查 fd 是否可读
fd_set fds;
FD_ZERO(&fds);
FD_SET(sockfd, &fds);
struct timeval tv = {0, 0};          // 超时 0 秒 → 非阻塞检查
int ret = select(sockfd + 1, &fds, NULL, NULL, &tv);
if (ret > 0) {
    // socket 有数据可读，可以调 recvfrom
}

// Qt — 事件驱动：连接信号
connect(socket, SIGNAL(readyRead()), this, SLOT(onDataReady()));
// socket 有数据到达时 Qt 自动发射信号 → 槽函数执行
```

### 实际读取

```cpp
// BSD Socket — 阻塞调用，必须在 select 确认可读后调用
char buf[4096];
socklen_t sin_size = sizeof(struct sockaddr_in);
int ret = recvfrom(sockfd, buf, sizeof(buf), 0,
                   (struct sockaddr*)&sender, &sin_size);
// 如果没有数据，recvfrom 会一直阻塞

// Qt — 非阻塞，读多少算多少
QByteArray datagram;
datagram.resize(socket->pendingDatagramSize());
QHostAddress sender;
quint16 senderPort;
socket->readDatagram(datagram.data(), datagram.size(), &sender, &senderPort);
// 在 readyRead 槽中调用，此时确定有数据
```

### 配置与清理

```cpp
// BSD Socket
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val));
setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &val, sizeof(val));
close(sockfd);

// Qt
// SO_REUSEADDR 和缓冲区大小由 Qt 默认管理
socket->deleteLater();
```

## 应用场景对比

### BSD Socket 适用场景

**高频数据流（每秒 > 100 包）**

场景：FPGA 相机每秒推送数千 UDP 包（图像数据），IPC 高速抓图同理

原因：每次 `readyRead` 信号发射，Qt 内部要做——锁事件队列、查找连接的槽、构造 QEvent 对象、QMetaObject 元对象分发、槽函数调用、返回事件循环。单次开销几微秒，但乘以每秒数千次 → 累积可观。BSD Socket 的 `select() + recvfrom()` 直接系统调用，没有中间层开销。

**极致延迟控制**

场景：需要自己控制 recvfrom 的时机和超时策略

原因：`select()` 的超时参数精确到微秒（`struct timeval {秒, 微秒}`），`recvfrom()` 阻塞深度由调用者决定。Qt 的信号机制延迟取决于事件循环当前负载——无法精确控制。

**无需 Qt 事件循环的线程**

场景：纯 C 线程，不跑 exec()

原因：MyNetWorkThread、MyFastNetWorkThread 的 `run()` 内是纯粹的 while(isrunning) 死循环，没有 exec()。QUdpSocket 的 readyRead 依赖事件循环才能触发信号——没有 exec() 信号永远不会来。

### Qt QUdpSocket 适用场景

**低频命令交互（每秒 < 10 包）**

场景：IPC AI 工控机命令收发、状态查询

原因：信号开销在低频场景下可忽略。Qt 的 readyRead 信号 + 槽函数开发效率远高于手写 select 轮询——不需要手动管理 fd_set、不需要填 sockaddr_in、不需要关心字节序。

**Qt 线程中需要异步响应**

场景：SendThread 的 run() 中调了 exec()

原因：readyRead 信号被 exec() 的事件循环捕获并分发给槽函数。多个 socket（BackgroudObj + FastNetObj）共用一个事件循环，Qt 自动管理——不需要手写 select 监听多个 fd。

**需要跨平台兼容**

场景：代码需要在 Windows 和 Linux 上编译

原因：QUdpSocket 在 Windows 底层用 WinSock，Linux 底层用 BSD Socket——但上层 API 完全一致。BSD Socket 原始代码在 Windows 上需要 `WSAStartup/WSA Cleanup`、`closesocket` 替代 `close`、`SOCKET` 替代 `int fd` 等适配。

## 本工程中的典型代码

### BSD Socket 典型代码（mynetwork.cpp:2352-2427）

```cpp
void NetWork::udpBind(QString ip, QString port)
{
    // 1. 创建 socket
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    
    // 2. 改内核缓冲区（Linux 特有）
    system("echo 8388608 > /proc/sys/net/core/rmem_default");
    system("echo 33554432 > /proc/sys/net/core/rmem_max");
    
    // 3. 设置 socket 选项
    int val = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val));  // 端口重用
    val = 8 * 1024 * 1024;
    setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &val, sizeof(val));     // 8MB 接收缓冲
    
    // 4. 绑定本机地址
    struct sockaddr_in addr_local;
    memset(&addr_local, 0, sizeof(addr_local));
    addr_local.sin_family = AF_INET;
    addr_local.sin_port = htons(8081);
    addr_local.sin_addr.s_addr = inet_addr("162.254.129.100");
    bind(sockfd, (struct sockaddr*)&addr_local, sizeof(addr_local));
    
    // 5. 设置远端地址（后续 sendto 用）
    struct sockaddr_in addr_remote;
    memset(&addr_remote, 0, sizeof(addr_remote));
    addr_remote.sin_family = AF_INET;
    inet_pton(AF_INET, "162.254.129.10", &addr_remote.sin_addr);
    addr_remote.sin_port = htons(5000);
}
```

### Qt QUdpSocket 典型代码（myudpthread.cpp:911-1007）

```cpp
// 构造函数：创建 + 绑定
BackgroudObj::BackgroudObj(QObject *parent, int listenPort, SendThread* pAttach)
{
    udpSocket = new QUdpSocket(this);
    m_bIsBind = udpSocket->bind(QHostAddress("162.254.129.100"), listenPort);  // 8082
}

// run() 中连接信号
connect(m_MySocket->udpSocket, SIGNAL(readyRead()),
        m_MySocket, SLOT(readPendingDatagrams()));

// 槽函数：收到数据
void BackgroudObj::readPendingDatagrams()
{
    while (udpSocket->hasPendingDatagrams()) {
        QByteArray datagram;
        datagram.resize(udpSocket->pendingDatagramSize());
        QHostAddress sender;
        quint16 senderPort;
        udpSocket->readDatagram(datagram.data(), datagram.size(), &sender, &senderPort);
        // 校验 + 分发
        m_pAttach->m_parent->processTheDatagram(datagram, ipOffset);
    }
}

// 发送
qint64 BackgroudObj::realSendData(const QByteArray &sendBuf, ...)
{
    return udpSocket->writeDatagram(sendBuf, QHostAddress("162.254.129.10"), 5001);
}
```

## 选型决策流程

```
                                是否需要 UDP 通信？
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                   每秒 > 100 包？               每秒 < 10 包？
                        │                           │
                    BSD Socket                  Qt QUdpSocket
                    + while 死循环               + exec 事件循环
                        │                           │
                ┌───────┴────────┐          ┌───────┴────────┐
                │                │          │                │
           需要跨平台？      纯 Linux     需要跨平台？       纯 Linux
                │                │          │                │
           加 ifdef        直接用          直接用         直接用
           WinSock 适配     BSD API       Qt API          Qt API
```
