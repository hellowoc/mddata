# 通道2：IPC 命令通道

## 涉及的文件

| 文件 | 内容 |
|------|------|
| `CameraTest(aiuse)/zksort/bus/myudpthread.h` | 类定义、协议头 `udp_header`、命令字宏 |
| `CameraTest(aiuse)/zksort/bus/myudpthread.cpp` | 完整实现 |
| `CameraTest(aiuse)/zksort/global/globalflow.cpp:10044` | `g_Udp.initSocket()` 启动入口 |
| `CameraTest(aiuse)/zksort/global/globalparams.h:202` | `struIpcShareInfo` 结构体定义 |

---

## 1. 这个通道是干什么的

IPC 命令通道是上位机与 IPC AI 工控机之间**所有控制命令的收发通道**。

数据流方向是**双向的**：上位机发命令 → IPC 回响应。IPC 也会主动上报心跳。

典型场景：
- 用户点击"加载模型" → 上位机发 `CMD_UDP_IPC_REQ_MODEL_LOAD` → IPC 回复加载状态
- IPC 每 500ms 发一次心跳（`CMD_UDP_IPC_REQ_INFO`） → 上位机更新界面上的 CPU/内存/温度

---

## 2. 涉及哪些类和线程

```
┌── 主线程 ─────────────────────────────────────────┐
│                                                    │
│  [[references/全局单例|g_Udp]] (MyUdp)              │
│  ├── pushUdpCmdData()                              │
│  └── processTheDatagram()                          │
│                                                    │
└────┬───────────────────────────────────────────────┘
     │ 拥有
┌────┴── SendThread (独立线程) ──────────────────────┐
│                                                    │
│  ├── BackgroudObj (QUdpSocket)                     │
│  │   ├── bind(162.254.129.100:8082)                │
│  │   └── readyRead → readPendingDatagrams()        │
│  │                                                 │
│  └── FastNetObj (QUdpSocket)                       │
│      ├── bind(192.168.1.100:20000)                 │
│      └── readyRead → readPendingDatagrams()         │
│                                                    │
└────────────────────────────────────────────────────┘
```

**线程关系**：
- `MyUdp` 对象在**主线程**创建（`globalflow.cpp:10044`）
- `SendThread` 是**独立线程**，构造函数里调用 `start()` 自己启动自己，`run()` 里调 `exec()` 进入事件循环
- `BackgroudObj` 和 `FastNetObj` 在 **SendThread 线程**中创建（`SendThread::run()` 里 `new` 的），所以它们的 `QUdpSocket` 也属于 SendThread
- 主线程通过调用 `g_Udp.pushUdpCmdData()` → `SendThread::addPacket()` → `BackgroudObj::realSendData()` 来发送数据

---

## 3. 协议头结构

**定义位置**：`myudpthread.h:82-106`

一个 UDP 包 = **14 字节头** + **数据载荷**：

```
偏移  长度  字段名        含义
────────────────────────────────────
0     2     identify[2]   帧头，固定 0xA5 0x5A
2     2     verSion[2]    版本号，固定 0x00 0x02
4     1     board_h       板地址高字节 → 查 [[references/全局单例|g_Udp]] 的 m_viewIndexMap 得 view
5     1     board_l       板地址低字节 → unit = board_l - 1
6     2     len_h/len_l   数据载荷长度（不含头本身，但含 6 字节额外开销）
8     2     seq_h/seq_l   全局序列号（[[references/C标准库函数|uint16_t]]，递增到 65530 归零）
10    2     cmd_h/cmd_l   命令字（决定这个包是干什么的）
12    2     crc_h/crc_l   CRC 校验（前 12 字节的累加和）
```

**board_h/board_l 与 view/unit 的映射**：`board_h` 是硬件板卡的编号，`board_l` 是板上的单元编号。上位机通过 `m_viewIndexMap` 把硬件地址映射为逻辑的 view/unit：

```cpp
// myudpthread.cpp:26-33 — 发送时：view → board_h
m_viewIndexMap.insert(0, 0x02);  // view 0 → board_h = 0x02
m_viewIndexMap.insert(1, 0x04);  // view 1 → board_h = 0x04

// myudpthread.cpp:81 — 接收时：board_h → view
view = m_viewIndexMap.key(head.board_h);  // 0x02 → 0
unit = head.board_l - 1;
```

**CRC 校验**：把包头前 12 个字节的每个字节累加，结果写入 `crc_h/crc_l`。不是标准 CRC16，而是简单的**累加和**。用于检测传输错误。

---

## 4. 发送路径

假设场景：**用户点击加载模型**，最终要发 `CMD_UDP_IPC_REQ_MODEL_LOAD`（`0x0004`）。

### Step 1：业务层调用

在 [[全局单例|myFlow]] 中：
```cpp
// globalflow.cpp 某处
g_Udp.pushUdpCmdData(CMD_UDP_IPC_REQ_MODEL_LOAD, true, viewId, unitId, dataLen, data);
```

**入参**：
| 参数 | 含义 | 本例的值 |
|------|------|---------|
| `nCmd` | 命令字 | `0x0004` |
| `needReturn` | 是否需要回包 | `true` |
| `viewId` | 目标视野编号 | 如 `0x09`（全部相机） |
| `unitId` | 目标单元编号 | 如 `0xff`（全部单元） |
| `dataLen` | 数据载荷长度 | 模型加载参数的长度 |
| `data` | 数据载荷指针 | 模型加载参数的二进制数据 |

### Step 2：组装协议头

**函数**：`MyUdp::pushUdpCmdData()`（`myudpthread.cpp:430-479`）

```cpp
// 1. 根据 viewId 查 board_h、根据 unitId 查 board_l
board = struCnfg.struLayerInfo[0].nViewBoardType[viewId];
nUnitAddr = struCnfg.struLayerInfo[0].stuViewInfo[viewId].nViewUnitId[unitId] + 1;

// 2. 填充 udp_header 各字段
udp_header head;
head.identify[0] = 0xa5;                              // 帧头
head.identify[1] = 0x5a;
head.verSion[0]   = 0x00;                             // 版本
head.verSion[1]   = 0x02;
head.board_h = board;                                 // 板地址
head.board_l = nUnitAddr;
head.len_h = (dataLen + 6) / 256;                     // 包长（高字节）
head.len_l = (dataLen + 6) % 256;                     // 包长（低字节）
head.seq_h = udpPacketSeq / 256;                      // 全局序列号
head.seq_l = udpPacketSeq % 256;
if (udpPacketSeq++ > 65530) udpPacketSeq = 0;         // 序列号回绕
head.cmd_h = nCmd / 256;                              // 命令字
head.cmd_l = nCmd % 256;

// 3. 计算 CRC = 前 12 字节累加和
sum = identify[0] + identify[1] + verSion[0] + verSion[1]
    + board_h + board_l + len_h + len_l + seq_h + seq_l + cmd_h + cmd_l;
head.crc_h = sum / 256;
head.crc_l = sum % 256;

// 4. 交给发送线程
udpSender->addPacket(head, data, viewId, unitId, dataLen);
```

### Step 3：拼包 + 路由

**函数**：`SendThread::addPacket()`（`myudpthread.cpp:781-812`）

```cpp
// 1. 拼完整包（14字节头 + 数据载荷）
QByteArray send_buf;
send_buf.resize(headSize + dataLen);
memcpy(send_buf.data(), &header, headSize);          // 拷贝头
memcpy(send_buf.data() + headSize, buf, dataLen);     // 拷贝数据

// 2. 根据命令字选择走哪个 socket
if (cmd == CMD_UDP_HT_GET_CAM_IMAGE_INFO || cmd == CMD_UDP_HT_GET_FAST_CAPTURE)
{
    // 高速抓图命令 → m_MySocket2（FastNetObj, eth1, 192.168.1.x）
    peerIP = 从配置拼出 192.168.1.x;
    sendLen = m_MySocket2->realSendData(send_buf, peerIP);
}
else
{
    // 普通 IPC 命令 → m_MySocket（BackgroudObj, eth0, 162.254.129.10）
    sendLen = m_MySocket->realSendData(send_buf, peerIP);
}
```

**路由逻辑**：高速抓图命令（`0x5006`/`0x5007`）走 FastNetObj → eth1 网卡；其余所有 IPC 命令走 BackgroudObj → eth0 网卡。

### Step 4：发送到网络

**函数**：`BackgroudObj::realSendData()`（`myudpthread.cpp:925-957`）

```cpp
strPeerIP = "162.254.129.10";  // 硬编码覆盖，无论传入什么 IP

QMutexLocker lock(&m_SendLock);  // 加锁（多线程安全）
udpSocket->writeDatagram(sendBuf,
    QHostAddress(strPeerIP),             // → 162.254.129.10
    m_pAttach->remotePort);              // → 端口 5001
```

**关键细节**：`strPeerIP` 在函数体内被硬编码覆盖为 `"162.254.129.10"`。这意味着无论调用方传什么 IP 参数，普通 IPC 命令永远发给 `162.254.129.10:5001`。

---

## 5. 接收路径

### Step 1：初始化 — 绑定监听端口

**时机**：程序启动时，`myFlow.initAll()` → `g_Udp.initSocket()`（`globalflow.cpp:10044`）

`initSocket()` 只做了 `new SendThread(this)`。SendThread 的构造函数里调用 `start()`，然后 `while(!m_bInit)` 忙等待直到子线程初始化完成。

### Step 2：子线程创建 socket 对象

**函数**：`SendThread::run()`（`myudpthread.cpp:715-737`）

```cpp
// 1. 创建 IPC 命令 socket（eth0）
m_MySocket = new BackgroudObj(NULL, 8082, this);
connect(m_MySocket->udpSocket, SIGNAL(readyRead()), 
        m_MySocket, SLOT(readPendingDatagrams()));

// 2. 创建快速命令 socket（eth1）
m_MySocket2 = new FastNetObj(NULL, 20000, this);
connect(m_MySocket2->udpSocket, SIGNAL(readyRead()), 
        m_MySocket2, SLOT(readPendingDatagrams()));

m_bInit = true;   // 通知主线程"我初始化好了"
exec();           // 进入 Qt 事件循环
```

**BackgroudObj 构造函数**（`myudpthread.cpp:911-917`）中调用 `bind()`：

```cpp
udpSocket = new QUdpSocket(this);
m_bIsBind = udpSocket->bind(QHostAddress("162.254.129.100"), 8082);
```

### Step 3：收到数据

**函数**：`BackgroudObj::readPendingDatagrams()`（`myudpthread.cpp:974-1007`）

```cpp
while (udpSocket->hasPendingDatagrams())     // 可能有多个包排队
{
    QByteArray datagram;
    datagram.resize(udpSocket->pendingDatagramSize());  // 按实际大小分配

    QHostAddress sender;       // 发件人 IP（出参）
    quint16 senderPort;        // 发件人端口（出参）
    udpSocket->readDatagram(datagram.data(), datagram.size(),
                            &sender, &senderPort);      // 读出完整包

    // 校验帧头
    if (datagram.at(0) != 0xa5 || datagram.at(1) != 0x5a || datagram.size() > 4096)
        continue;  // 不是合法的 IPC 包，丢弃

    // 计算 ipOffset（从发送方 IP 最后一段推算）
    quint8 ipOffset = sender.toString().right(3).toInt() - IPC_IP_ADDR_BASE;

    // 交给主对象解析
    m_pAttach->m_parent->processTheDatagram(datagram, ipOffset);
}
```

**`m_pAttach->m_parent`**：`m_pAttach` 是 `SendThread*`，`m_parent` 是 `SendThread` 持有的 `MyUdp*`（即 `[[references/全局单例|g_Udp]]`）。所以这行调用的是 `g_Udp.processTheDatagram()`。

### Step 4：解析回包

**函数**：`MyUdp::processTheDatagram()`（`myudpthread.cpp:56-149`）

```cpp
// 1. 取出包头
udp_header head;
memcpy(&head, datagram.data(), headSize);   // 14 字节

// 2. 反向查 view/unit
view = m_viewIndexMap.key(head.board_h);    // 例：0x02 → 0
unit = head.board_l - 1;                    // 例：1 → 0

// 3. 计算数据载荷长度
nPacketLen = head.len_h * 256 + head.len_l - 6;

// 4. 取出数据载荷
unsigned char* buf = (unsigned char*)malloc(nPacketLen);
memcpy(buf, datagram.data() + headSize, nPacketLen);

// 5. 通知发送线程"这个序列号的包已收到"
udpSender->RecvPacket(head);

// 6. 提取命令字
nCmd = (head.cmd_h * 256 + head.cmd_l) & 0x7fff;

// 7. 按命令字分发
switch (nCmd)
{
case 0x0001:  // CMD_UDP_IPC_REQ_INFO — IPC 心跳
    struIpcShare.struIpcInfo[view][unit].aliveNum  = buf[0]<<8 | buf[1];
    struIpcShare.struIpcInfo[view][unit].version   = 拷贝 buf[2..33]，32字节;
    struIpcShare.struIpcInfo[view][unit].modelStat = buf[68];
    struIpcShare.struIpcInfo[view][unit].cpuUsed   = buf[81];
    struIpcShare.struIpcInfo[view][unit].memUsed   = buf[82];
    struIpcShare.struIpcInfo[view][unit].chipTemp  = buf[83];
    // ... 共解析 30+ 个字段
    break;

case 0x0002:  // CMD_UDP_IPC_REQ_MODEL_ABLE — 模型列表
    // 解析模型名称和类别数
    break;

case 0x00C0:  // CMD_CAMERA_AI_MISC_STATUS — IPC 综合状态
    // 解析固件版本、模型上传状态
    break;
}

free(buf);  // 释放临时缓冲区
```

**数据最终去向**：`processTheDatagram()` 将解析结果写入 `[[references/全局参数|struIpcShare]]`。UI 的定时器每秒读取 `struIpcShare` 并刷新显示。

**`RecvPacket(head)`**：从 `SendThread` 的发送队列中移除匹配的包（按序列号 `seq` 匹配）。如果该包之前已重发多次，`RecvPacket` 标记它已收到，不再重发。

---

## 6. 完整的收发链路图

```
界面操作（点击按钮）
    │
    ↓
myFlow.xxx()                            ← 主线程
    │
    ↓
g_Udp.pushUdpCmdData(CMD_xxx, ...)      ← 主线程：组装 14 字节头 + CRC
    │
    ↓
udpSender->addPacket(head, data, ...)   ← 主线程调子线程方法（无锁）
    │
    ↓
BackgroudObj::realSendData()            ← SendThread 中执行
    │
    ↓
writeDatagram() → 网卡 eth0             ← 系统调用
    ═══════════════════ 以太网 → IPC AI 工控机 ══════════════
    │
    │ IPC 处理后回复
    ↓
UDP 包到达 162.254.129.100:8082         ← Linux 内核
    │
    ↓
QUdpSocket::readyRead 信号              ← Qt 事件循环
    │
    ↓
BackgroudObj::readPendingDatagrams()    ← SendThread 中执行
    │
    ↓
processTheDatagram()                    ← SendThread 中执行
    │
    ├── 写入 [[references/全局参数|struIpcShare]]（全局共享数据）
    │
    ↓
UI 定时器(1秒) 读取 struIpcShare        ← 主线程
    │
    ↓
界面刷新：版本、CPU、内存、温度...        ← 主线程
```

---

## 7. 线程跨越点

| 环节 | 所在线程 | 说明 |
|------|---------|------|
| UI 操作 → pushUdpCmdData | 主线程 | 组装包头，无 I/O |
| addPacket → realSendData | SendThread | 跨线程方法调用 |
| writeDatagram | SendThread | 系统调用，不阻塞 |
| readyRead → readPendingDatagrams | SendThread | Qt 信号驱动 |
| processTheDatagram | SendThread | 解析 + 写入全局数据 |
| UI 定时器读取 | 主线程 | SendThread 写、主线程读，时间差可接受 |

---

## 8. 命令字速查

**完整定义**：`myudpthread.h:15-49`

| 命令字 | 宏名 | 功能 |
|--------|------|------|
| `0x0001` | `CMD_UDP_IPC_REQ_INFO` | 心跳查询 IPC 状态 |
| `0x0002` | `CMD_UDP_IPC_REQ_MODEL_ABLE` | 查询可用模型列表 |
| `0x0003` | `CMD_UDP_IPC_REQ_MODEL_INFO` | 查询模型详情 |
| `0x0004` | `CMD_UDP_IPC_REQ_MODEL_LOAD` | 加载模型 |
| `0x0008` | `CMD_UDP_IPC_REQ_START_SORT` | 启动/停止分选 |
| `0x0009` | `CMD_UDP_IPC_REQ_SHUTDOWN` | 关闭/重启 IPC |
| `0x00C0` | `CMD_CAMERA_AI_MISC_STATUS` | IPC 心跳 + 综合状态 |
| `0x00C1` | `CMD_SCREEN_GET_FILE` | 从 IPC 下载文件 |
| `0x5006` | `CMD_UDP_HT_GET_FAST_CAPTURE` | 启动高速抓图 |
| `0x5101` | `CMD_UDP_HT_GET_CAM_IMAGE_INFO_ANS` | 相机图像信息回包 |
