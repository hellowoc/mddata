# select() 非阻塞机制详解 — 从阻塞根因到五种方案对比

## 文档主线

`recvfrom()` 默认是阻塞的——没数据时线程在内核中挂起。本工程用 `select()` 解决：在调 `recvfrom` 之前先问内核"fd 可读吗？"，有数据才收。这条主线串起五种方案（select + 阻塞 / O_NONBLOCK / SO_RCVTIMEO / Qt readyRead），以及本工程为什么在不同通道选了不同的方案。

这里select的可读指的是有没有数据，而不是权限上是否可读

---

## 第一章：阻塞的本质 — `recvfrom()` 在内核中到底做了什么

### 1.1 内核中的 socket 接收缓冲区

每个 socket 在内核中有一个 `struct sock` 结构体，维护了一个接收缓冲区——本质是一个链表，每个节点是 `sk_buff`：

```c
struct sock {
    struct sk_buff_head sk_receive_queue;  // 接收队列（双向链表头）
    wait_queue_head_t    sk_sleep;         // 等待队列头：等数据的进程挂在这里
    int                  sk_rcvbuf;        // 接收缓冲区总大小（字节）
};
```

数据到达流程：

```
网卡硬中断 → 驱动 → 内核协议栈 → IP层处理 → UDP层: skb加入 sk_receive_queue 队尾
                                                        ↓
                                                  唤醒 sk_sleep 上等待的进程
```

### 1.2 `recvfrom()` 的内核路径

```
用户态:
  retNum = recvfrom(sockfd, buf, 4096, 0, &addr, &addrlen);

═══════════  系统调用入口（syscall 指令） ═══════════

内核态:
  SYSCALL_DEFINE6(recvfrom, ...)
    → __sys_recvfrom()
      → sock_recvmsg()
        → sock->ops->recvmsg() = udp_recvmsg()
          → __skb_recv_udp()
            → __skb_recv_datagram()
              → __skb_try_recv_datagram()
                → __skb_try_recv_from_queue(&sk->sk_receive_queue)
                │  ↓ 遍历 sk_receive_queue 链表
                │  队列非空 → 取一个 skb，copy_to_user() 到用户态 buf → 返回字节数
                │  队列为空 → 返回 NULL
                │
                → 队列为空 → wait_for_packet()
                  → 进程状态设为 TASK_INTERRUPTIBLE（可中断睡眠）
                  → 进程加入 sk->sk_sleep 等待队列
                  → schedule() 让出 CPU
                  │
                  │  ┌─────────────────────────────┐
                  │  │  进程在这里挂起（阻塞）        │
                  │  │  CPU 分配给其他进程/线程       │
                  │  │  等待网卡中断唤醒...           │
                  │  └─────────────────────────────┘
                  │
                  │  网卡收到数据 → 中断 → skb 入队
                  │  → wake_up_interruptible(&sk->sk_sleep)
                  │  → 进程被唤醒 → 状态变回 TASK_RUNNING
                  │
                  → 重新检查 sk_receive_queue → 有数据了
                  → copy 数据到用户态 buf → 释放 skb → 返回
```

**阻塞发生在 `wait_for_packet()` → `schedule()`**。进程在这里交出 CPU，直到网卡中断触发唤醒。

---

## 第二章：`select()` — 在调 `recvfrom` 之前先问内核

### 2.1 数据结构

```c
// fd_set — 位掩码，每个 bit 代表一个 fd 是否在集合中
// Linux 64位：1024 bit ÷ 64 bit/long = 16 个 long = 128 字节
typedef struct {
    long fds_bits[16];
} fd_set;

// 三个操作宏
#define FD_ZERO(set)      memset(set, 0, sizeof(fd_set))
//   示例：FD_ZERO(&rds) → rds.fds_bits[0..15] = 0

#define FD_SET(fd, set)   ((set)->fds_bits[fd / 64] |= (1L << (fd % 64)))
//   示例：FD_SET(5, &rds) → rds.fds_bits[0] 的第 5 位置 1

#define FD_ISSET(fd, set) ((set)->fds_bits[fd / 64] &  (1L << (fd % 64)))
//   示例：FD_ISSET(5, &rds) → 返回非 0 表示 fd=5 可读

// 时间结构体 — 精确到微秒
struct timeval {
    long tv_sec;   // 秒
    long tv_usec;  // 微秒（1,000,000 微秒 = 1 秒）
};
```

### 2.2 `select()` 函数签名

```c
int select(int nfds,           // 监控范围：0 到 nfds-1，通常是 max_fd + 1
           fd_set *readfds,    // 入参：关心哪些 fd 可读；出参：哪些 fd 确实可读
           fd_set *writefds,   // 关心哪些 fd 可写（本工程传 NULL）
           fd_set *exceptfds,  // 关心哪些 fd 异常（本工程传 NULL）
           struct timeval *timeout);  // 最多等多久。NULL=永久等，{0,0}=不等立即返回

返回值：
  >0 ：有 N 个 fd 就绪
   0 ：超时到期，没有 fd 就绪
  -1 ：出错（fd 无效等）
```

### 2.3 `select()` 的内核执行流程

```c
int do_select(int n, fd_set *readfds, struct timespec *end_time)
{
    // 1. 把用户态 fd_set copy 到内核空间

    // 2. 遍历 0 到 n-1 的所有 fd
    for (int fd = 0; fd < n; fd++) {
        if (FD_ISSET(fd, readfds)) {
            // 检查 sk_receive_queue 是否非空
            if (!skb_queue_empty(&sk->sk_receive_queue)) {
                retval++;                     // 已经有数据了
                set_bit(fd, &result_bitmap);  // 标记这个 fd 就绪
            }
        }
    }

    // 3. 如果有 fd 已经就绪 → 直接返回，不等待
    if (retval > 0) goto out;

    // 4. 所有 fd 都没数据 → 把进程加入每个 fd 的等待队列
    for (int fd = 0; fd < n; fd++) {
        poll_table table;
        table.pt._qproc = __pollwait;
        table.pt._key  = POLLIN;  // 关心"可读"事件
        sock->ops->poll(fd, sock, &table);
        // poll 注册回调：当 fd 收到数据时，唤醒当前进程
    }

    // 5. 设定时器 = timeout，挂起
    int ret = schedule_timeout(end_time);
    //     ┌─────────────────────────────┐
    //     │  在这里等。两种唤醒条件：       │
    //     │  A. 网卡收到数据 → 回调触发    │
    //     │  B. 定时器到期 → 超时         │
    //     └─────────────────────────────┘

    // 6. 被唤醒 → 从等待队列移除 → 遍历 fd → 写入 result_bitmap
    // 7. 把 result_bitmap copy 回用户态 &rds
    return retval;
}
```

### 2.4 select 和 recvfrom 的关键区别

| | `recvfrom(fd)` | `select(fd+1, &rds, NULL, NULL, &timeout)` |
|------|------|------|
| 挂起方式 | 无限期等，直到数据到达 | 等 timeval 指定时间后超时返回 |
| 等什么 | 等一个 fd 有数据 | 可以同时等多达 1024 个 fd |
| 数据去向 | 把数据 copy 到用户态 | 不读数据，只告诉你"可读了" |
| 返回后做什么 | 数据已经在 buf 里 | 需要再调 recvfrom 实际读数据 |

---

## 第三章：本工程 select + recvfrom 配对实战

### 3.1 checkUdpSocketStatues() — 有时限的等待

```cpp
// mynetwork.cpp:617-638
int MyNetWorkThread::checkUdpSocketStatues(int seconds)
{
    // 1. 构造超时 = seconds 秒 + 500ms
    struct timeval timeout;
    timeout.tv_sec  = seconds;
    timeout.tv_usec = 500 * 1000;      // 500,000 微秒 = 500ms

    // 2. 把 sockfd 放入 fd_set
    fd_set rds;
    FD_ZERO(&rds);          // 清空位掩码
    FD_SET(sockfd, &rds);   // sockfd 对应 bit 置 1

    // 3. 调 select
    int ret;
    ret = select(sockfd + 1, &rds, NULL, NULL, &timeout);

    // 4. 三种返回值
    if (ret == -1) {
        // select 调用本身失败（sockfd 无效、网卡 down）
        emit SignalSocketFailed();
        myFlow.msleep(200);
        return -1;
    }
    if (ret == 0) {
        // 超时 — 等了 timeout 时间，sockfd 依然不可读
        return -1;
    }
    // ret > 0 — sockfd 可读，可以安全调 recvfrom
    return 1;
}
```

### 3.2 run() 中的配对使用

```cpp
// mynetwork.cpp:1144-1156（简化）
while (isrunning) {
    // ★ 先 select：最多等 500ms
    if (checkUdpSocketStatues(0) < 0) {
        break;  // 超时或故障 → 退出，回到外层检查 isrunning
    }

    // ★ 再 recvfrom：select 保证有数据，不会阻塞
    retNum = recvfrom(sockfd, (char *)&wavePacketBuf[curTotal],
                      waveBufSize - curTotal, 0,
                      (struct sockaddr*)&client, &sin_size);
    curTotal += retNum;
    if (curTotal == waveBufSize) break;
}
```

`isrunning` 响应延时 ≤ 500ms（`checkUdpSocketStatues` 的单次超时上限）。

### 3.3 cleanUdpSocket() — timeout = 0 的极速清理

```cpp
// mynetwork.cpp:822-857
void MyNetWorkThread::cleanUdpSocket(int seconds)
{
    struct timeval timeout;
    fd_set rds;
    int ret = -1;

    while (1) {
        FD_ZERO(&rds);
        FD_SET(sockfd, &rds);
        timeout.tv_sec  = 0;      // 0 秒
        timeout.tv_usec = 0;      // 0 微秒 → select 不挂起

        ret = select(sockfd + 1, &rds, NULL, NULL, &timeout);

        if (ret == -1) { emit SignalSocketFailed(); return; }
        if (ret == 0)  { return; }  // 没数据 → 已清空 → 退出

        if (FD_ISSET(sockfd, &rds)) {
            ret = recv(sockfd, dirtydatabuf, sizeof(dirtydatabuf), 0);
            if (ret <= 0) return;
        }
        // 继续循环：可能还有更多脏数据
    }
}
```

`timeout = {0, 0}` 让 `select` 成为纯检测工具——只检查不等待。

---

## 第四章：方案 B — `fcntl(O_NONBLOCK)` 改为非阻塞 socket

### 用法

```c
#include <fcntl.h>

int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

// 之后 recvfrom 行为改变：
// 有数据 → 正常返回
// 没数据 → 立即返回 -1，errno = EAGAIN / EWOULDBLOCK
```

内核变化：`fcntl` 把 `sock->file->f_flags` 的 `O_NONBLOCK` 置 1。`recvfrom` 内核路径中 `__skb_try_recv_datagram()` 发现队列为空时，不调用 `wait_for_packet()`，直接返回 `-EAGAIN`。

### 使用模式

```c
while (isrunning) {
    ret = recvfrom(fd, buf, size, 0, ...);
    if (ret > 0) {
        // 处理数据
    } else if (ret == -1 && errno == EAGAIN) {
        msleep(10);  // 没数据，短暂休眠，让出 CPU
    }
}
```

### 对比 select 方案

| | select + 阻塞 recvfrom | O_NONBLOCK + 非阻塞 recvfrom |
|------|------|------|
| 没数据时的行为 | select 挂起等 timeout，CPU 让出 | recvfrom 立即返回 EAGAIN，循环重试 |
| CPU 消耗 | 挂起时 CPU 空闲 | 循环消耗 CPU（需配合 msleep） |
| 超时控制 | select 的 timeval 精确到微秒 | 需自建定时器或循环计数 |
| 多 fd | 一次 select 监控多个 | 逐个轮询 |
| 本工程 | ✓ 通道1、通道3 | ✗ |

---

## 第五章：方案 C — `setsockopt(SO_RCVTIMEO)` 设超时

### 用法

```c
struct timeval timeout = {3, 0};  // 3 秒超时
setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, &timeout, sizeof(timeout));

// 之后 recvfrom：
// 有数据 → 正常返回
// 超时没数据 → 返回 -1，errno = EAGAIN
```

内核变化：超时值写入 `sock->sk_rcvtimeo`。`recvfrom` 的 `wait_for_packet()` 中调用 `schedule_timeout(sk->sk_rcvtimeo)` 替代无限等待。

### 本工程为什么禁用

```cpp
// mynetwork.cpp:2473-2485 — 整个函数体被 #if 0 禁用了
void NetWork::setUdpRecvTimeOut(int t_s, int t_ms)
{
#if 0
    struct timeval time_out = {t_s, t_ms * 1000};
    setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, ...);
#endif
}
```

原因：`SO_RCVTIMEO` 只能设**一个固定值**。但波形接收要快（500ms）、图像接收可以久（10s）、清脏数据要 0。三个场景三个超时，一个固定值不够。`select()` 每次调用可以传入不同的 `timeout`。

---

## 第六章：方案 D — Qt QUdpSocket::readyRead 信号

### 本工程通道2 的做法

```cpp
// myudpthread.cpp — BackgroudObj

udpSocket = new QUdpSocket(this);
udpSocket->bind(QHostAddress("162.254.129.100"), 8082);

connect(udpSocket, SIGNAL(readyRead()), this, SLOT(readPendingDatagrams()));

void BackgroudObj::readPendingDatagrams()
{
    while (udpSocket->hasPendingDatagrams()) {
        QByteArray datagram;
        datagram.resize(udpSocket->pendingDatagramSize());
        QHostAddress sender;
        quint16 senderPort;
        udpSocket->readDatagram(datagram.data(), datagram.size(), &sender, &senderPort);
        // 处理数据...
    }
}
```

Qt 事件循环底层也用 `select()`/`poll()` 监控 socket fd。区别是 Qt 把"fd 可读"封装成了 `readyRead` 信号。

### select 方案 vs readyRead 方案

| | BSD select + recvfrom | Qt readyRead |
|------|------|------|
| 驱动方式 | 轮询：手动调 select 问"有数据吗" | 事件驱动：有数据才通知 |
| 所在线程 | while(isrunning) 死循环 | exec() 事件循环 |
| 中间开销 | select → recvfrom 直接系统调用 | 每次 readyRead：锁事件队列 → QEvent → QMetaObject 分发 → 槽调用 |
| 适用 | 每秒数千包高频数据 | 每秒几条的低频命令 |
| 本工程 | 通道1 FPGA 相机、通道3 高速抓图 | 通道2 IPC 命令 |

---

## 第七章：五种方案完整对比

| | 核心 API | 阻塞行为 | 超时控制 | 本工程 |
|------|------|------|------|------|
| select + 阻塞 | `select()` + `recvfrom()` | select 挂起等 timeout | 每次不同 timeval | ✓ 通道1、3 |
| O_NONBLOCK | `fcntl()` + `recvfrom()` | 立即返回 EAGAIN | 自建循环+msleep | ✗ |
| SO_RCVTIMEO | `setsockopt()` + `recvfrom()` | 挂起等固定超时 | 单值写死 | 尝试过，#if 0 禁用 |
| Qt readyRead | `QUdpSocket` + `connect()` | 事件循环代劳 | Qt 内部 select | ✓ 通道2 |

**本工程选 select 的核心原因**：高频数据场景下直接系统调用无 Qt 信号开销，且每次可传不同 `timeout` 精确控制等待时间。

---

## 附录：本工程相关源文件索引

| 文件 | 函数/行号 | 内容 |
|------|---------|------|
| `CameraTest(aiuse)/zksort/bus/mynetwork.cpp:617-638` | `checkUdpSocketStatues()` | select + timeout 检查 |
| `CameraTest(aiuse)/zksort/bus/mynetwork.cpp:822-857` | `cleanUdpSocket()` | timeout={0,0} 极速清理 |
| `CameraTest(aiuse)/zksort/bus/mynetwork.cpp:1089-` | `MyNetWorkThread::run()` | select + recvfrom 配对 |
| `CameraTest(aiuse)/zksort/bus/mynetwork.cpp:2473-2485` | `setUdpRecvTimeOut()` | SO_RCVTIMEO 方案（已禁用） |
| `CameraTest(aiuse)/zksort/bus/myudpthread.cpp` | `BackgroudObj` | Qt readyRead 方案 |
