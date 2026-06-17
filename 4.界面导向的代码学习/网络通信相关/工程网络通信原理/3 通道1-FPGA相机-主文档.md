# 通道1：FPGA 相机通道

## 涉及的文件

| 文件 | 内容 |
|------|------|
| `CameraTest(aiuse)/zksort/bus/mynetwork.h` | 类定义、BMP 位图结构体、采图模式宏 |
| `CameraTest(aiuse)/zksort/bus/mynetwork.cpp` | UDP 绑定、线程主循环、波形/图像包处理、BMP 保存 |
| `CameraTest(aiuse)/zksort/global/globalflow.cpp:10047-10048` | `myNetWork` 的创建和绑定入口 |

---

## 1. 这个通道是干什么的

FPGA 相机通道负责从 **FPGA 相机板** 接收三种数据：

| 数据类型 | 宏 | 说明 |
|---------|----|------|
| 波形数据 | `CAPTURE_SINGLE_WAVE`(0) | 单行 RGB+红外波形，用于灵敏度调试 |
| 图像数据 | `CAPTURE_IMAGE`(1) | 完整 BMP 图像（832 行），用于图像采集页显示 |
| 喷阀计数 | `CAPTURE_BLOW_COUNTS`(3) | 各喷阀的吹气次数统计 |

数据流方向是**收为主**：FPGA 相机板持续推送数据，上位机被动接收。

### 与通道2的本质区别

| | 通道1 (FPGA) | 通道2 (IPC) |
|------|-----------|----------|
| Socket | [[references/Socket编程基础|BSD socket]] | [[references/Qt网络类|Qt QUdpSocket]] |
| 驱动方式 | `while(isrunning)` 死循环 | `exec()` 事件循环 |
| 包到达通知 | [[references/Socket编程基础|select()]] 轮询 | [[references/Qt网络类|readyRead]] 信号 |
| 读取方式 | [[references/Socket编程基础|recvfrom()]] 阻塞读 | `readDatagram()` 非阻塞读 |
| 包头 | `FF 00 A5 5A C3 3C`（6字节） | `A5 5A + 12字节头`（14字节） |
| 退出方式 | `isrunning = false` | `quit()` + `wait()` |

---

## 2. 涉及哪些类和线程

```
┌── 主线程 ────────────────────────────────────────────┐
│                                                      │
│  NetWork (QWidget)                                   │
│  ├── udpBind("162.254.129.100", "8081")               │
│  │     ├── [[references/Socket编程基础|socket()]]      │
│  │     ├── [[references/Socket编程基础|setsockopt()]]   │
│  │     ├── [[references/Socket编程基础|bind()]]         │
│  │     └── 设置 addr_remote = 162.254.129.10:5000     │
│  ├── udpStart()                                      │
│  └── udpStop()                                       │
│                                                      │
└────┬─────────────────────────────────────────────────┘
     │ 拥有
┌────┴── MyNetWorkThread (独立线程) ───────────────────┐
│                                                      │
│  void run() {                                        │
│      while (isrunning) {                             │
│          switch (currenttype) {                      │
│          case CAPTURE_SINGLE_WAVE:                   │
│              发命令给FPGA → select()等数据             │
│              → recvfrom()收包 → 解析                  │
│              → emit SignalWaveRecvFinished()          │
│          case CAPTURE_IMAGE:                         │
│              ... 收到 imageHeight 行 → 保存BMP         │
│          }                                           │
│      }                                               │
│  }                                                   │
│                                                      │
│  信号：                                               │
│  ├── SignalWaveRecvFinished()  → 波形控件刷新         │
│  ├── SignalImageRecvFinished() → 图像控件显示          │
│  └── SignalBlowCountsRecvFinished() → 喷阀计数更新     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**线程关系**：
- `NetWork` 是 QWidget，在**主线程**创建（`globalflow.cpp:10047`）
- `MyNetWorkThread` 在 NetWork 构造函数中 `new`（主线程），但 `start()` 后 `run()` 在**独立线程**执行
- `run()` 是经典的 `while(isrunning)` 模式——线程的整个生命周期在死循环里度过
- 数据通过**信号**（`emit SignalWaveRecvFinished()`）从子线程传到主线程

### 还有一个抓图线程

`MyNetCptureThread`（`mynetwork.h` 中声明，`mynetwork.cpp` 中实现）是**另一个**独立的抓图线程。与 `MyNetWorkThread` 的区别：
- `MyNetWorkThread`：从 FPGA 收波形/图像（来自硬件相机板）
- `MyNetCptureThread`：网络图像抓取（可能来自其他网络源）

在 `globalflow.cpp:10051-10052` 创建但默认不启动（被注释掉了 `start()`）。

---

## 3. 包头格式

FPGA 相机通道的包头是 **6 个固定字节**：

```
FF 00 A5 5A C3 3C
│  │  └──┬───┘ └──┬───┘
│  │     帧头2     帧头3
│  帧头1
帧头0
```

与通道2 的 UDP 协议不同，FPGA 包头**不含任何可变字段**（没有 board 地址、命令字、序列号、CRC）。它纯粹是一个"魔术字"——6 字节固定模式，用来在字节流中标记"一帧数据的开始"。

**为什么这么简单**：FPGA 相机板通过串口命令来切换工作模式（发 `serialSendTransStartStop()` 告诉 FPGA "我要波形"），然后 FPGA 就开始推数据。不需要在 UDP 包里重复携带模式信息。

校验代码（`mynetwork.cpp:1163-1164`）：

```cpp
if ((wavePacketBuf[0] != 0xFF) || (wavePacketBuf[1] != 0x00)
    || (wavePacketBuf[2] != 0xA5) || (wavePacketBuf[3] != 0x5A)
    || (wavePacketBuf[4] != 0xC3) || (wavePacketBuf[5] != 0x3C))
{
    qDebug() << "wave head error!";
    continue;  // 包头不对，丢弃这一帧
}
```

---

## 4. 初始化流程

### Step 1：创建 NetWork 对象

`globalflow.cpp:10047-10048`：
```cpp
myNetWork = new NetWork;
myNetWork->udpBind(ipForInterface, "8081");
```

`NetWork` 继承自 `QWidget`。为什么网络对象是 QWidget？因为 NetWork 需要接收 FPGA 线程发来的信号，信号/槽机制要求 QObject 派生类。

### Step 2：创建 socket

**函数**：`NetWork::udpBind()`（`mynetwork.cpp:2352-2427`）

```cpp
// 1. 创建 BSD socket
sockfd = socket(AF_INET, SOCK_DGRAM, 0);
// AF_INET → IPv4
// SOCK_DGRAM → UDP（数据报）
// 0 → 自动选择协议

// 2. 扩展内核缓冲区（Linux）
system("echo 8388608 > /proc/sys/net/core/rmem_default");  // 默认 → 8MB
system("echo 33554432 > /proc/sys/net/core/rmem_max");      // 最大 → 32MB

// 3. SO_REUSEADDR — 允许端口重用
int val = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &val, sizeof(val));

// 4. SO_RCVBUF — 应用层接收缓冲区 8MB
int val = 8 * 1024 * 1024;
setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &val, sizeof(val));

// 5. 绑定本机地址
memset(&addr_local, 0, sizeof(addr_local));
addr_local.sin_family = AF_INET;
addr_local.sin_port = htons(8081);
addr_local.sin_addr.s_addr = inet_addr("162.254.129.100");
bind(sockfd, (struct sockaddr*)&addr_local, sizeof(addr_local));

// 6. 设置远端地址（给 sendto 用的）
memset(&addr_remote, 0, sizeof(addr_remote));
addr_remote.sin_family = AF_INET;
inet_pton(AF_INET, "162.254.129.10", &addr_remote.sin_addr);
addr_remote.sin_port = htons(5000);
```

### Step 3：启动接收线程

**函数**：`NetWork::udpStart()`（`mynetwork.cpp:2430-2438`）

```cpp
void NetWork::udpStart()
{
    udpthread->setSocket(sockfd);           // 把 socket fd 传给线程
    udpthread->setAddr_remote(addr_remote); // 把远端地址传给线程

    if (udpthread->isrunning) return;       // 已经在跑，不重复启动
    udpthread->isrunning = true;            // 设标志
    udpthread->start();                     // 创建系统线程 → 调用 run()
}
```

---

## 5. 线程主循环

**函数**：`MyNetWorkThread::run()`（`mynetwork.cpp:1089`起）

### 整体骨架

```cpp
void MyNetWorkThread::run()
{
    // 初始化局部变量
    updateCameraInfo();                      // 更新图像传感器信息到 [[references/全局参数|struGsh]]

    while (isrunning)                        // 靠这个标志位活着
    {
        switch (currenttype)                 // currenttype 决定当前是哪种采图模式
        {
        case CAPTURE_SINGLE_WAVE:            // 单行波形
        case CAPTURE_BACKGROUND_WAVE:        // 背景波形
        case CAPTURE_IPC_EJECT:              // IPC 坏点
            // === 波形接收流程 ===
            break;

        case CAPTURE_IMAGE:                  // 单幅图像
        case CAPTURE_IAMGE_CONTINUOUS:       // 连续图像
            // === 图像接收流程 ===
            break;

        case CAPTURE_BLOW_COUNTS:            // 喷阀计数
            // === 计数接收流程 ===
            break;
        }
    }
}
```

### 波形接收流程（以 CAPTURE_SINGLE_WAVE 为例）

```cpp
case CAPTURE_SINGLE_WAVE:
    // 1. 清空缓冲区残留数据
    cleanUdpSocket();

    // 2. 更新相机参数
    updateCameraInfo();

    // 3. 通过串口通知 FPGA："我要波形数据"
    serialSendTransStartStop();

    // 4. 循环收包，直到收够一帧波形（waveBufSize 字节）
    while (isrunning)
    {
        // 4a. 用 select() 检查 socket 是否有数据可读
        if (checkUdpSocketStatues(0) < 0)
            break;

        // 4b. 阻塞接收
        retNum = recvfrom(sockfd,
                          (char*)&wavePacketBuf[curTotal],  // 接到波行缓冲区
                          waveBufSize - curTotal,            // 还剩多少空间
                          0,
                          (struct sockaddr*)&client,         // 发件人地址
                          &sin_size);

        curTotal += retNum;    // 累加已收字节

        if (curTotal == waveBufSize)   // 收满一帧 → 退出内层循环
            break;
    }

    // 5. 检查是否收满
    if (curTotal != waveBufSize)
    {
        myFlow.msleep(100);
        continue;               // 没收满，回到外层 while 重试
    }

    // 6. 校验包头 FF 00 A5 5A C3 3C
    if (包头不对)
    {
        qDebug() << "wave head error!";
        continue;               // 丢弃，重试
    }

    // 7. 从包缓冲拷贝到数据缓冲（跳过 6 字节包头）
    memcpy(waveDataBuf, &wavePacketBuf[6], dim * curPixels * td_dim);

    // 8. 解析波形 → 填入 [[references/全局参数|struGsh]].sRowRed/Green/Blue/Inf[]
    waveDataPacketed(true);

    // 9. 通知界面："波形数据好了"
    emit SignalWaveRecvFinished();

    // 10. 等 300ms，再进行下一次采集
    myFlow.msleep(300);
    break;
```

### 图像接收流程（以 CAPTURE_IMAGE 为例）

图像比波形大得多——不是一包能装下的，需要收 **imageHeight 行**的数据，每行 waveBufSize 字节。

```cpp
case CAPTURE_IMAGE:
    cleanUdpSocket();

    // 通知 FPGA 开始传输图像
    serialSendTransStartStop();

    // 计算需要收的行数（如果有图像拼接，行数翻倍）
    curImageRowCnt = myFlow.imgHeight;  // 默认 400 行

    while (isrunning)
    {
        // select() 检查可读
        checkUdpSocketStatues(1);   // 超时 1 秒

        // recvfrom() 收包
        retNum = recvfrom(sockfd, ...);
        curTotal += retNum;

        // 算一下已经收到了多少行
        imageRowCnt = curTotal / waveBufSize;

        // 收满指定行数 → 跳出
        if (curTotal >= waveBufSize * curImageRowCnt)
            break;
    }

    // 校验每行的包头 + 统计正确行数
    imageCorrectWaveCnt = imageCheckSum(imageRowCnt);

    // 保存为 BMP 文件
    saveBMPFile(path, width, height);

    // 通知界面
    emit SignalImageRecvFinished(imageCorrectWaveCnt, path);
    break;
```

---

## 6. 采图模式的切换

外部调用 `NetWork::setUdpRecvType(type)` 来切换模式：

```cpp
// 使用示例：
myNetWork->setUdpRecvType(CAPTURE_SINGLE_WAVE);      // 切换到波形模式
myNetWork->udpStart();                                // 启动线程
// ... 波形采集完成 ...
myNetWork->udpStop();                                 // 停止线程
```

`currenttype` 被 `run()` 的外层 `while(isrunning)` 循环的 `switch` 语句读取，所以：
- 在 `udpStop()` → `udpStart()` 之间修改 `currenttype` → 下次循环生效
- 不能在 `run()` 正在执行一个 case 时热切换（没有同步机制）

---

## 7. 为什么这个通道用 BSD socket 而不是 Qt

| 原因 | 解释 |
|------|------|
| **数据量** | 单帧图像 400 行 × 520 字节/行 ≈ 200KB，每秒可能多帧 |
| **信号开销** | Qt 的 `readyRead` 每收一个包发射一次信号。200KB 可能分几十个 UDP 包 → 几十次信号发射 → 开销大 |
| **select() 更高效** | 一次 `select()` 确认有数据，然后循环 `recvfrom()` 收完所有包，系统调用次数少 |
| **控制粒度** | `while(isrunning)` 可以随时 `break/continue`，控制流更直观 |

**Qt 事件循环的开销**：每次 `readyRead` 发射信号时，Qt 要做：锁事件队列 → 查找连接的槽 → 调用槽函数 → 返回事件循环。对于每秒几十次的交互式 UI 这不算什么，但对每秒成千上万包的相机数据流来说，累积开销就大了。

---

## 8. 完整的收发链路图

```
界面操作（点击"采集波形"按钮）
    │
    ↓
myNetWork->setUdpRecvType(CAPTURE_SINGLE_WAVE)    ← 主线程
myNetWork->udpStart()                              ← 主线程
    │
    ↓
MyNetWorkThread::start()                           ← 主线程调，子线程开始执行
    ═══════════════════ 线程边界 ═════════════════
    │
    ↓
run() → while(isrunning)                           ← 子线程
    │
    ↓
serialSendTransStartStop()                         ← 子线程：通过串口通知FPGA
    │
    ↓                                         串口 → FPGA 相机板
    │
    │ FPGA 开始推送 UDP 数据
    ↓
while (isrunning):
    checkUdpSocketStatues(0)  → select()          ← 子线程：检查 socket
    recvfrom(sockfd, ...)                          ← 子线程：收包
    ├── 检查包头 FF 00 A5 5A C3 3C
    ├── memcpy 到 waveDataBuf
    └── waveDataPacketed() → 填入 [[references/全局参数|struGsh]]
    ═══════════════════ 线程边界 ═════════════════
    │
    ↓
emit SignalWaveRecvFinished()                      ← 子线程 emit → 主线程槽
    │
    ↓
主线程槽函数：更新波形控件显示                       ← 主线程
```

---

## 9. 线程跨越点

| 环节 | 所在线程 | 说明 |
|------|---------|------|
| setUdpRecvType / udpStart | 主线程 | 设置参数、启动线程 |
| socket/bind/setsockopt | 主线程 | 初始化在 `udpBind()` 中完成 |
| select / recvfrom | 子线程 | 实际 I/O 在 `run()` 中进行 |
| memcpy / waveDataPacketed | 子线程 | 数据处理 |
| emit 信号 | 子线程 | Qt 自动跨线程调度到主线程 |
| 槽函数（UI 刷新） | 主线程 | 信号触发后在主线程执行 |

---

## 10. 采图模式速查

**定义位置**：`mynetwork.h:31-44`

| 宏 | 值 | 功能 | 触发场景 |
|------|-----|------|---------|
| `CAPTURE_SINGLE_WAVE` | 0 | 单行波形 | 波形显示页 |
| `CAPTURE_IMAGE` | 1 | 单幅图像（400行） | 图像采集页 |
| `CAPTURE_BLOW_COUNTS` | 3 | 喷阀吹气频率 | 喷阀调试页 |
| `CAPTURE_BACKGROUND_WAVE` | 4 | 背景波形 | 背景设置页 |
| `CAPTURE_IAMGE_CONTINUOUS` | 5 | 连续采图 | 连续图像显示 |
| `CAPTURE_MULTI_IMAGE` | 7 | 多张采集 | 多图对比 |
| `CAPTURE_IPC_EJECT` | 10 | IPC 坏点数 | IPC 状态页 |
