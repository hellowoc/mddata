# 通道1 FPGA相机 — 基础知识清单

按在 [[06-通道1-FPGA相机-主文档|主文档]] 中首次出现的顺序排列。

---

## 第 1 章出现的

### FPGA 相机板（§1）

FPGA = Field Programmable Gate Array（现场可编程门阵列）。一种可编程的硬件芯片。

在这个工程中，FPGA 相机板是一块硬件板卡，上面有一颗 FPGA 芯片和多颗 CMOS 图像传感器。FPGA 负责：
1. 控制相机曝光和读出
2. 对原始像素做硬件预处理（白平衡、伽马校正）
3. 将数据打包成 UDP 包，通过网线发送给上位机

相比 IPC AI 工控机（跑 Linux + GPU 推理），FPGA 是纯硬件逻辑，延迟极低（微秒级），但只能做固定的简单算法。

### 波形数据（§1）

"波形"指的是一行像素的 RGB 值曲线。色选机使用**线阵相机**——每次只拍一行像素（1024 或 2048 个像素点），物料在传送带上移动，连续拍摄行组成一幅图像。

波形调试是工程师用来调整灵敏度的核心工具——看某个颜色的波形峰值多高，决定触发喷阀的阈值。

### BMP 位图（§1）

BMP = Bitmap，Windows 标准位图格式。无压缩，文件体积大但解码简单。

主文档中提到的 `BITMAPFILEHEADER`（文件头）、`BITMAPINFOHEADER`（信息头）、`RGBQUAD`（调色板）都定义在 `mynetwork.h:55-88`。BMP 的像素数据从下往上、从左往右排列，24 位色按 BGR 顺序存储。

---

## 第 2 章出现的

### QThread::start()（§2）

见 [[references/Qt核心类型#QThread]]。

### isrunning 标志位（§2）

`MyNetWorkThread` 的 `bool` 成员（`mynetwork.h:157`）。是模式一死循环线程的"生命开关"。

`udpStart()` 设为 `true` → `run()` 开始循环。`udpStop()` 设为 `false` → `run()` 退出循环 → 线程结束。

### emit（§2）

Qt 关键词，发射信号。`emit SignalWaveRecvFinished()` 向所有连接了此信号的槽发送通知。Qt 自动处理跨线程调度——子线程 emit，主线程的槽函数被调用。

### 信号/槽跨线程机制（§2）

子线程 emit 信号时：
1. Qt 检查信号的接收者（槽所在对象）属于哪个线程
2. 如果和发送者同一线程 → 直接调用槽函数（同步）
3. 如果不同线程 → 将调用封装为"事件"，放入接收者线程的事件队列，等接收者线程的事件循环处理（异步）

本工程中，`MyNetWorkThread` emit 信号 → 主线程 Widget 的槽函数 → Qt 自动走异步路径。

---

## 第 3 章出现的

### 魔术字/魔数（Magic Number）（§3）

协议中约定的固定字节序列，用于帧同步。选择一个不太常见的字节组合，降低误判概率。

`FF 00 A5 5A C3 3C` 的设计思路：
- `FF 00`：来自相机传感器的像素数据通常是 0~255 之间的值，`FF 00` 相邻出现概率低
- `A5 5A`：二进制 `10100101 01011010`，明显的 0/1 交替模式
- `C3 3C`：与前两个互补的模式

### 串口命令（§3）

上位机通过**串口**（COM1/COM2）向 FPGA 发送命令，告诉 FPGA 切换工作模式。UDP 通道只负责**数据接收**，不负责控制。

`serialSendTransStartStop()` 内部调用 `mySerial.pushCom1CmdData()` 或 `pushCom2CmdData()`，发送串口命令帧给 FPGA。

---

## 第 4 章出现的

### ipForInterface（§4-Step1）

函数 `udpBind(ipForInterface, "8081")` 的第一个参数。从配置文件读取的 eth0 IP 地址（通常为 `"162.254.129.100"`）。

但这个参数在函数内部实际上**被忽略了**——`udpBind()` 的第一行就把 IP 硬编码为了 `"162.254.129.100"`：

```cpp
void NetWork::udpBind(QString ip, QString port)
{
    // ...
    addr_local.sin_addr.s_addr = inet_addr("162.254.129.100");  // 硬编码
    // 参数 ip 没有被实际用于 bind
}
```

### socket()（§4-Step2）

见 [[references/Socket编程基础#socket() — 创建 socket]]。

### AF_INET（§4-Step2）

见 [[references/Socket编程基础#AF_INET]]。

### SOCK_DGRAM（§4-Step2）

`SOCK_DGRAM` = **数据报 socket**（UDP）。与 `SOCK_STREAM`（流式 socket，TCP）相对。

DGRAM 来自 datagram（数据报）——UDP 以独立的数据报为单位收发，每个包有边界。TCP 是字节流，没有包的边界概念。

### 内核缓冲区（§4-Step2）

Linux 内核为每个 socket 维护接收/发送缓冲区。当数据到达网卡时，内核先把数据放入缓冲区，等待应用程序调用 `recvfrom()` 取走。如果应用程序取走速度慢于到达速度，缓冲区满了 → 新数据被丢弃（丢包）。

```bash
echo 8388608 > /proc/sys/net/core/rmem_default  # 默认接收缓冲 8MB
echo 33554432 > /proc/sys/net/core/rmem_max     # 最大接收缓冲 32MB
```

`/proc/sys/` 是 Linux 的**伪文件系统**，里面的文件不是真正的磁盘文件，而是内核参数的接口。`echo 值 > 文件` 就是在修改内核参数。

### SO_REUSEADDR（§4-Step2）

见 [[references/Socket编程基础#setsockopt() — 设置 socket 选项]]。

### SO_RCVBUF（§4-Step2）

`SO_RCVBUF` = Socket Option Receive Buffer。设置 socket 的**应用层**接收缓冲区大小。这里设为 8MB（`8 * 1024 * 1024` 字节）。

### sockaddr_in（§4-Step2）

见 [[references/Socket编程基础#sockaddr_in — "地址+端口"结构体]]。

### memset()（§4-Step2）

见 [[references/C标准库函数#memset(ptr, value, n)]]。

### htons()（§4-Step2）

见 [[references/Socket编程基础#htons(x) / ntohs(x) — 字节序转换]]。

### inet_addr()（§4-Step2）

见 [[references/Socket编程基础#inet_addr(str) — 字符串 IP → 二进制]]。

### bind()（§4-Step2）

见 [[references/Socket编程基础#bind() — 绑定地址]]。

### inet_pton()（§4-Step2）

见 [[references/Socket编程基础#inet_pton(family, str, dst) — 新版 IP 字符串转换]]。

### addr_remote（§4-Step2）

`NetWork` 的成员变量（`mynetwork.h` 中声明），类型 `struct sockaddr_in`。存储远端地址 `162.254.129.10:5000`。在 `udpSendTo()` 中作为发送目标。

### setSocket() / setAddr_remote()（§4-Step3）

`MyNetWorkThread` 的方法。把主线程创建的 socket 文件描述符和远端地址传给子线程：

```cpp
void setSocket(int fd)          { sockfd = fd; }
void setAddr_remote(struct sockaddr_in addr) { addr_remote = addr; }
```

### start()（§4-Step3）

见 [[references/Qt核心类型#QThread]]。这里与通道2 不同——通道2 的 SendThread 在**构造函数**中调 `start()`，通道1 的 MyNetWorkThread 由外部显式调用 `udpthread->start()`。

---

## 第 5 章出现的

### updateCameraInfo()（§5）

更新 `[[references/全局参数|struGsh]]` 中与相机相关的字段（像素数、维度、当前视野/层/单元）。在每次采图循环开始时调用，确保配置是最新的。

### cleanUdpSocket()（§5）

清空 socket 缓冲区中的残留数据。调用前可能有之前未读的数据残留，先清掉避免干扰本次采集。

内部做法是以 `MSG_DONTWAIT` 标志反复调用 `recvfrom()` 直到没有数据。

### serialSendTransStartStop()（§5）

通过串口向 FPGA 发送"开始/停止传输"命令。这个函数内部调用 `mySerial.pushCom1CmdData()` 或 `pushCom2CmdData()`，传递一个包含板卡类型、目标地址、启动/停止标志的串口帧。

### select()（§5）

见 [[references/Socket编程基础#select() — 检查 socket 是否有数据可读]]。

### checkUdpSocketStatues(0)（§5）

`MyNetWorkThread` 的方法（`mynetwork.cpp`）。内部封装了 `select()` 调用。参数 `0` 表示超时 0 秒——看一眼有没有数据，没有马上返回（非阻塞检查）。

### recvfrom()（§5）

见 [[references/Socket编程基础#recvfrom() — 阻塞接收 UDP 数据]]。

### wavePacketBuf / waveDataBuf（§5）

`MyNetWorkThread` 的成员数组：
- `wavePacketBuf[]`：原始接收缓冲区，直接接收 UDP 包字节（**含** 6 字节包头）
- `waveDataBuf[]`：解析后的数据缓冲区，只存波形数据（**不含**包头）

`memcpy(waveDataBuf, &wavePacketBuf[6], ...)` 跳过前 6 字节包头。

### waveBufSize（§5）

每行波形的字节数。在 `updateCameraInfo()` 中根据相机参数计算：
```
waveBufSize = dim × curPixels × td_dim × 每像素字节数 + 6(包头)
```

### curTotal（§5）

已接收的字节数。每次 `recvfrom()` 后累加，与 `waveBufSize` 比较判断是否收完一行。

### waveDataPacketed()（§5）

`MyNetWorkThread` 的方法（`mynetwork.cpp`）。将 `waveDataBuf` 中的原始波形数据解析为 RGB+红外四个通道，填入 `[[references/全局参数|struGsh]].sRowRed/Green/Blue/Inf[1024]`。

界面上的波形控件（`mywavewidget`）读取这些数组来绘制曲线。

### dim / curPixels / td_dim（§5）

相机参数：
- `dim`：颜色维度（1=灰度，3=RGB）
- `curPixels`：每行像素数（1024 或 2048）
- `td_dim`：是否包含红外通道（1=不含，2=含红外）

### SignalWaveRecvFinished()（§5）

`MyNetWorkThread` 的信号（`mynetwork.h:168`）。无参数。通知主线程"一行波形数据已就绪"，波形控件读取 `struGsh.sRowRed[]` 等数组并重绘。

### myFlow.msleep(300)（§5）

调用 `[[references/全局单例|myFlow]].msleep(300)`，让当前线程休眠 300 毫秒。为什么是 300ms？给 FPGA 板一些时间准备下一帧数据，也给界面时间刷新波形。

---

## 第 6 章出现的

### setUdpRecvType(type)（§6）

`NetWork` 的方法。设置 `MyNetWorkThread` 的 `currenttype` 成员变量，决定 `run()` 中 `switch` 走哪个 case。

---

## 第 7 章出现的

### Qt 元对象系统（§7）

Qt 在标准 C++ 基础上增加的特性：信号/槽、运行时类型信息、动态属性。由 moc（Meta-Object Compiler）在编译前处理 `Q_OBJECT` 宏所在的头文件，生成额外的 C++ 代码。

信号/槽的跨线程调度的开销来自元对象系统——每次 emit 要做：查找连接的槽 → 确定槽在哪个线程 → 如果是跨线程则构造事件 → 放入目标线程事件队列 → 等待事件循环处理。

### QWidget 作为网络对象（§1/§4）

`NetWork` 继承自 `QWidget` 而不是 `QObject`，因为：
1. QWidget 本身就是 QObject 的派生类，支持信号/槽
2. 没有实际的"界面"——`NetWork` 是一个隐藏的 QWidget，只用来管理 UDP 通信和接收信号

---

## 第 8 章出现的

### 跨线程 emit 的异步性（§8）

子线程 `emit SignalWaveRecvFinished()` 后：
- emit 本身**不阻塞**，不等待槽执行完
- 信号事件在主线程的事件队列里排队
- 主线程处理完当前事件（如鼠标点击处理完）后，才从队列里取出这个信号并执行槽

这意味着波形数据可能发出几毫秒后界面才刷新。对 300ms 的采集间隔来说，几毫秒延迟完全无感知。
