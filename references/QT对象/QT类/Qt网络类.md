# Qt 网络类

本工程中通道2（IPC 命令）使用了 Qt 封装的网络类，而非 BSD socket。

---

## QUdpSocket

**头文件**：`<QUdpSocket>`

**是什么**：Qt 对 UDP 的封装。底层也是 BSD socket，但用 Qt 的信号/槽和事件循环来驱动。

**对比 BSD socket**：`BSD socket` 的 `select()+recvfrom()` 是阻塞的 C API，`QUdpSocket` 是异步的信号驱动。

**本工程中的典型用法**：

```cpp
// 创建（myudpthread.cpp:915）
udpSocket = new QUdpSocket(this);

// 绑定（myudpthread.cpp:917）
m_bIsBind = udpSocket->bind(QHostAddress("162.254.129.100"), 8082);
// 绑定成功返回 true。失败原因可能是：端口已被占用、IP 不属于本机

// 连接信号（myudpthread.cpp:718）
connect(udpSocket, SIGNAL(readyRead()), this, SLOT(readPendingDatagrams()));
// 有数据到达时 Qt 自动发射 readyRead 信号

// 发送（myudpthread.cpp:956）
udpSocket->writeDatagram(sendBuf,
    QHostAddress("162.254.129.10"),   // 目标 IP
    5001);                              // 目标端口

// 接收（myudpthread.cpp:990-991）
QHostAddress sender;      // 发送方的 IP（出参）
quint16 senderPort;       // 发送方的端口（出参）
udpSocket->readDatagram(datagram.data(), datagram.size(), &sender, &senderPort);
```

**关键方法**：
| 方法 | 作用 |
|------|------|
| `bind(QHostAddress, port)` | 绑定本机 IP+端口，返回 bool |
| `writeDatagram(data, host, port)` | 发送数据报 |
| `readDatagram(data, maxSize, &sender, &port)` | 读取数据报，同时获取发送方地址 |
| `hasPendingDatagrams()` | 是否有待读取的数据报 |
| `pendingDatagramSize()` | 下一个数据报的字节数 |

---

## QHostAddress

**头文件**：`<QHostAddress>`

**是什么**：Qt 对 IP 地址的封装。类似于 BSD 的 `sockaddr_in`，但使用更简单。

**本工程中的典型用法**：

```cpp
// 从字符串构造
QHostAddress addr("162.254.129.100");

// 传给 bind
udpSocket->bind(QHostAddress("162.254.129.100"), 8082);

// 传给 writeDatagram
udpSocket->writeDatagram(sendBuf, QHostAddress(strPeerIP), remotePort);

// 作为出参接收发送方地址
QHostAddress sender;
udpSocket->readDatagram(..., &sender, &senderPort);
QString ip = sender.toString();              // → "162.254.129.10"
int lastPart = sender.toString().right(3).toInt();  // 取最后一段
```

---

## QSettings

**头文件**：`<QSettings>`

**是什么**：Qt 的配置文件读写类。可以读写 INI 格式的配置文件。

**本工程中的典型用法**：

```cpp
// machineinfowidget.cpp:71 — 读取 IP 配置
QSettings setting(CFG_APPSet, QSettings::IniFormat);
// CFG_APPSet 是配置文件路径的宏，一般在 constant.h 中定义

setting.beginGroup("eth0");                // 进入 [eth0] 节
QString ipStr0 = setting.value("ipStr").toString();  // 读取 ipStr=...
setting.endGroup();                        // 退出 [eth0] 节

setting.beginGroup("eth1");                // 进入 [eth1] 节
QString ipStr1 = setting.value("ipStr").toString();
setting.endGroup();
```

配置文件大概长这样：
```ini
[eth0]
ipStr=162.254.129.100
[eth1]
ipStr=192.168.1.100
```

---

## readyRead 信号 — Qt 特色的异步接收

**是什么**：QUdpSocket 的 **readyRead()** 信号。当有 UDP 数据到达时，Qt 的事件循环自动发射此信号。

**工作流程**：

```
UDP 包到达 162.254.129.100:8082
    ↓
Linux 内核将数据放入 QUdpSocket 的接收缓冲区
    ↓
Qt 事件循环检测到 socket 可读
    ↓
发射 readyRead() 信号
    ↓
连接的槽函数 readPendingDatagrams() 被调用
    ↓
while (udpSocket->hasPendingDatagrams())
    循环读取所有排队的数据报
```

**为什么用 while 循环读**：可能同时到达多个包，每个包触发一次信号，但槽被调用时可能又来了新包。用 `while` 确保一次性清空缓冲区。
