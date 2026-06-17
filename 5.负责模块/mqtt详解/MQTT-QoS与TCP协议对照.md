# MQTT QoS 与 TCP 协议对照

## 一、层次关系

```
┌──────────────────────────────────────────┐
│  MQTT QoS 2 (应用层可靠)                   │
│  保证：消息从"发布者"可靠到达"订阅者"       │
│  中间经过 Broker 转发，TCP 管不到这一层      │
├──────────────────────────────────────────┤
│  TCP (传输层可靠)                          │
│  保证：字节从"主机A的内存"可靠到达"主机B的内存"│
│  只管两台机器之间的传输，不管消息内容         │
├──────────────────────────────────────────┤
│  IP (网络层，不可靠)                        │
│  只负责把包从源 IP 路由到目标 IP，可能丢包    │
└──────────────────────────────────────────┘
```

**MQTT 跑在 TCP 之上。TCP 已经保证了 MQTT 报文在"工控机↔Broker"这一段不丢、不乱序。** 那为什么 MQTT 还需要 QoS？因为 MQTT 的"端到端"是**发布者 → Broker → 订阅者**，跨越了两段 TCP 连接。MQTT QoS 弥补的是 Broker 转发环节的可靠性，不是 TCP 连接本身。

---

## 二、TCP 三次握手 — 建立连接

### 为什么是三次

双方都要确认"我能收到你的消息，你也能收到我的消息"。

```
步骤1: 客户端 → 服务器: SYN(seq=x)
       "我想连你，我的起始序号是 x"

步骤2: 客户端 ← 服务器: SYN(seq=y) + ACK(ack=x+1)
       "我收到了你的 SYN（ack=x+1 就是确认），我的起始序号是 y"

步骤3: 客户端 → 服务器: ACK(ack=y+1)
       "我收到了你的 SYN+ACK"
```

为什么不是两次？两次只能确认"客户端→服务器的链路是通的"。服务器不知道客户端能不能收到自己的消息。

为什么不是四次？步骤 2 把服务器的 SYN 和对客户端 SYN 的 ACK 合并到一个包里了——"捎带确认"（piggybacking）。

### 关键数据结构

```c
// TCP 连接在内核中维护的状态（每端各有一套）
struct tcp_sock {
    uint32_t snd_nxt;    // 下一个要发送的字节序号
    uint32_t snd_una;    // 最早的未被确认的字节序号
    uint32_t rcv_nxt;    // 期望收到的下一个字节序号
    uint32_t snd_wnd;    // 发送窗口大小（对方还能收多少）
    uint32_t rcv_wnd;    // 接收窗口大小（我还能收多少）
    // ... 重传队列、拥塞控制参数等
};
```

**sequence number（序号）** 是理解 TCP 的关键——TCP 把数据看作**字节流**，每个字节有一个序号。ACK 的含义是"序号 N 之前的所有字节我都收到了，期待下一个字节是 N"。

---

## 三、TCP 数据传输 — 确认与重传

这和 MQTT QoS 1 的思路完全一样：

```
MQTT QoS 1:                          TCP:
发送 PUBLISH(PacketId=123)           发送 seq=1000~1999
等 PUBACK(123)                       等 ACK=2000
超时 → 重发 PUBLISH(DUP=1, 123)      超时 → 重发 seq=1000~1999
```

区别在于 TCP 不是每个包都单独确认——**TCP 是累积确认**：

```
发送方连续发了 4 个段：
  seq=1000~1999 (len=1000)
  seq=2000~2999 (len=1000)
  seq=3000~3999 (len=1000)
  seq=4000~4999 (len=1000)

接收方回了：
  ACK=3000   ← 这意味着 1000~2999 全收到了，期待 3000 开始的数据

如果 seq=3000~3999 丢了，接收方回：
  ACK=3000   ← 重复确认，"我还是在等 3000"

发送方收到 3 次重复 ACK=3000 → 立即重发 seq=3000~3999（快速重传）
```

这和 MQTT QoS 1 的"一个 PacketId 一个 PUBACK"不同——TCP 的 ACK 覆盖了一个**范围**。

---

## 四、TCP 四次挥手 — 断开连接

### 为什么比握手多一次

因为 TCP 是**全双工**的——双方都可以独立地"我不再发数据了"，但另一方可能还想继续发。

```
步骤1: 主动关闭方 → 被动方: FIN(seq=m)
       "我不再发数据了（但我还能收）"

步骤2: 主动关闭方 ← 被动方: ACK(ack=m+1)
       "知道了，你那边关了"
       ★ 此时：主动方→被动方 的方向已关闭
       但 被动方→主动方 的方向还开着，被动方可能还有数据要发

步骤3: 主动关闭方 ← 被动方: FIN(seq=n)
       "我的数据也发完了，我也关了"

步骤4: 主动关闭方 → 被动方: ACK(ack=n+1)
       "知道了"
       主动方等 2MSL（Maximum Segment Lifetime × 2，约 2 分钟）后彻底关闭
```

**为什么 FIN 和 ACK 不合并（像握手那样）？** 握手时，服务器收到 SYN 后**立刻**就可以回 SYN+ACK。挥手时，被动方收到 FIN 后**可能还有数据没发完**。它必须先 ACK 表示"收到你的关闭请求"，然后继续发剩余数据，发完后才能发 FIN。这两个动作时间上可能间隔很久，不能合并。

---

## 五、两端状态机对照

```
MQTT QoS 2 发送方:                    TCP 连接状态机:
                                      
IDLE                                  CLOSED
  │ send PUBLISH                        │ 主动: send SYN
  ▼                                     ▼
WAIT_FOR_PUBREC                       SYN_SENT
  │ recv PUBREC                         │ recv SYN+ACK, send ACK
  ▼                                     ▼
WAIT_FOR_PUBCOMP                      ESTABLISHED  ← 数据传输阶段
  │ recv PUBCOMP                        │
  ▼                                     │ 主动: send FIN
IDLE                                    ▼
                                      FIN_WAIT_1
                                        │ recv ACK
                                        ▼
                                      FIN_WAIT_2
                                        │ recv FIN, send ACK
                                        ▼
                                      TIME_WAIT (等 2MSL)
                                        │
                                        ▼
                                      CLOSED
```

**核心对应关系**：

| MQTT QoS 要素 | TCP 对应物 |
|--------------|-----------|
| PacketId（2字节） | Sequence Number（4字节） |
| PUBACK = "收到消息123" | ACK = "字节 N 之前的都收到了" |
| 重发 PUBLISH(DUP=1) | 超时重传 / 3次重复ACK 快速重传 |
| PUBREC = "我承担转发责任" | TCP 没有这一层——TCP 不关心"应用是否处理了数据" |
| PUBREL = "你可以处理了" | TCP 没有对应物——MQTT 在 TCP 之上的额外保障 |
| PUBCOMP = "处理完毕" | TCP 没有对应物 |

**PUBREC/PUBREL/PUBCOMP 是 MQTT 独有的——因为 MQTT 的 Broker 不是终点，是中间人。TCP 只管两台机器之间的传输，"收到"就是终点。**

---

## 六、从本工程代码看 TCP vs UDP 的选择

### 为什么不用 TCP 而全用 UDP + 自定义可靠性

1. **FPGA 是纯硬件**，没有 TCP/IP 协议栈，只能发原始 UDP 包。选 TCP 意味着要在 FPGA 上实现完整的 TCP 状态机——在硬件逻辑里不现实。

2. **工业相机每秒数千帧**，TCP 的累积确认 + 拥塞控制会引入不可预测的延迟。丢一帧没关系（下一帧马上到），但延迟一帧就可能导致喷阀错过物料。

3. **应用层自定义协议可以精确控制重传策略**：UDP + 序列号匹配：重传 3 次，每次等 500ms。TCP 默认重传：15 次，RTO 由内核自适应。前者在"快速失败→让用户知道→人工介入"上更优。

### MQTT 为什么用 TCP

MQTT 是设备→云端的控制通道，包量少（一秒几条），可靠性第一，延迟第二——正是 TCP 适合的场景。

---

## 七、用 BSD Socket 写 TCP 通信模块

基于本工程已有的代码风格：

```cpp
// ====== TCP 客户端（类似 MQTT 连接 Broker） ======

// 1. 创建 socket
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
//                          ↑ TCP = SOCK_STREAM
//                          UDP = SOCK_DGRAM

// 2. 连接服务器（TCP 三次握手在这里完成）
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(8900);
inet_pton(AF_INET, "cloud.chinaamd.cn", &server_addr.sin_addr);

int ret = connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr));
// connect() 内部完成：SYN → 等 SYN+ACK → 回 ACK
// 成功返回 0 时，三次握手已完成，连接进入 ESTABLISHED 状态

// 3. 发送数据
const char *data = "GET /api/status HTTP/1.1\r\n...";
int sent = send(sockfd, data, strlen(data), 0);
// TCP 保证：sent 字节的数据会按序、完整地到达服务器
// 不像 UDP 的 sendto，不需要指定目标地址（连接已建立）

// 4. 接收数据
char buf[4096];
int n = recv(sockfd, buf, sizeof(buf), 0);
// 阻塞直到有数据到达
// TCP 保证：收到的字节和发送顺序完全一致，没有丢字节

// 5. 关闭连接（四次挥手）
close(sockfd);
// close() 内部：发 FIN → 等 ACK → 等对方的 FIN → 回 ACK
```

### 和本工程 UDP 代码的关键区别

| | UDP (mynetwork.cpp) | TCP (mqttsrv 底层) |
|------|-----|-----|
| socket 类型 | `SOCK_DGRAM` | `SOCK_STREAM` |
| 连接 | 无连接，每次 sendto 指定地址 | `connect()` 建立连接，之后 send 不需要指定地址 |
| 接收 | `recvfrom()` 每个包有边界 | `recv()` 字节流，没有包边界 |
| 可靠性 | 丢包不重传 | 自动重传丢的字节 |
| 数据边界 | 有（一个 sendto = 一个包） | 没有（多次 send 的数据可能合并到一个 recv 中） |

**TCP 字节流"没有边界"是最大的坑**——你可能 `send("hello")` 然后 `send("world")`，对方一次 `recv` 收到 `"helloworld"`。所以 TCP 应用程序自己要在字节流之上定义消息边界——MQTT 协议就是干这个的（用变长编码的 Remaining Length 来划分消息边界）。

---

## 八、MQTT QoS 2 在 TCP 层面的实际传输

```
应用层（MQTT）              传输层（TCP）              网络层（IP）

PUBLISH(123) →
                              send() → 数据放入 TCP 发送缓冲区
                              TCP 把数据分成若干 TCP 段
                              每个段：seq=N, len=M
                              ─→ TCP段 ─→ IP包 ─→ 以太网帧 ─→
                                                       
                              ←── TCP段(ACK=N+M) ────────
                              ACK 确认字节全部到达
                              recv() 从缓冲区取数据
← PUBREC(123)                 
                              send() → PUBREC 字节放进 TCP 缓冲区...
                              （同上，省略）

... PUBREL 和 PUBCOMP 同理，每条 MQTT 报文都被 TCP 保证可靠传输 ...
```

**TCP 保证了每一段 TCP 连接上的 MQTT 报文可靠到达。MQTT QoS 保证了报文经过 Broker 转发后，仍然可靠到达最终订阅者。**
