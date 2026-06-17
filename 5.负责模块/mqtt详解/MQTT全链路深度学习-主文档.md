# MQTT 全链路深度学习 — 从协议规范到嵌入式部署

## 文档说明

本文档覆盖 MQTT 从底层协议到本工程源码的全链路，目标读者是需要接手 MQTT 模块开发的工程师。

**配套文档**：[[MQTT全链路深度学习-基础知识]] — 按出现顺序排列的基础概念解释

---

# 第一篇：MQTT 协议规范

## 1.1 MQTT 是什么

MQTT = Message Queuing Telemetry Transport（消息队列遥测传输），1999 年由 IBM 的 Andy Stanford-Clark 和 Arcom 的 Arlen Nipper 为石油管道传感器监控设计。2013 年提交给 OASIS 标准化组织，当前最新版本是 MQTT v5.0（2019年），本工程使用的是 **MQTT v3.1.1**（2014年 OASIS 标准）。

核心设计约束：
- **极低带宽**：最小固定头仅 2 字节。石油管道卫星链路带宽极低（几百 bps），HTTP 的头部动辄几百字节无法使用。
- **低功耗**：协议设计允许设备大部分时间处于休眠状态，只在有数据时唤醒。
- **不可靠网络**：卫星/油田/工厂 WiFi 经常断线，协议内建心跳保活和遗嘱消息机制。

## 1.2 发布/订阅模型 vs 请求/响应模型

```
请求/响应（HTTP）：                  发布/订阅（MQTT）：

客户端 ───GET /api/data───→ 服务器     发布者 ───publish("temp", "25°C")───→ ┌────────┐
客户端 ←──200 OK {data}─── 服务器                                          │ Broker  │
                                                                           │         │
一对一，客户端必须知道服务器地址。           订阅者A ←──"25°C"── 订阅了"temp"  └────────┘
每次通信客户端主动发起。                   订阅者B ←──"25°C"── 订阅了"temp"
                                          订阅者C （不订阅"temp"，收不到）

                                          发布者不知道订阅者是谁，只跟 Broker 通信。
                                          一对多、多对多、多对一都天然支持。
```

**Broker（消息代理）** 是 MQTT 的核心。它不产生消息，只做三件事：
1. 接收客户端的 TCP 连接
2. 接收发布者发来的消息
3. 根据主题匹配规则转发给订阅者

**主题（Topic）** 是 MQTT 的寻址方式，用 UTF-8 字符串表示，层级用 `/` 分隔：

```
/ZKGDDEV47/cmd/switch          # 发往设备 ZKGDDEV47 的开关命令
/ZKGDDEV47/status/temperature  # 设备 ZKGDDEV47 上报的温度
/MqttServerClient              # 所有设备回包给云端的公共主题
```

本工程中：
- 设备订阅 `"/" + 机器编号`，如 `"/ZKGDDEV47"` → 接收云端下发的命令
- 设备 publish 到 `"MqttServerClient"` → 云端服务器订阅此主题接收回包

## 1.3 MQTT 报文格式（二进制层面）

MQTT 是二进制协议，所有报文由三部分组成：

```
┌──────────────┬──────────────┬──────────────┐
│  Fixed Header │ Variable Header │    Payload    │
│  (2-5 字节)    │  (部分报文有)     │  (部分报文有)   │
└──────────────┴──────────────┴──────────────┘
```

### 1.3.1 固定头（Fixed Header）— 所有报文必有

```
Byte 1:  ┌─────────────────────────┬───┬───┬───┬───┐
         │    Message Type (4bit)    │DUP│QoS│QoS│RETAIN│
         └─────────────────────────┴───┴───┴───┴───┘
         
Byte 2-5: Remaining Length（变长编码，1-4 字节）
```

**Message Type（4bit，0-15）** — 决定"这个报文是干什么的"：

| 值 | 报文类型 | 方向 | 作用 |
|----|---------|------|------|
| 1 | CONNECT | C→B | 客户端请求连接 |
| 2 | CONNACK | B→C | Broker 确认连接 |
| 3 | PUBLISH | 双向 | 发布消息 |
| 4 | PUBACK | 双向 | QoS 1 的确认 |
| 5 | PUBREC | 双向 | QoS 2 第一步确认 |
| 6 | PUBREL | 双向 | QoS 2 第二步释放 |
| 7 | PUBCOMP | 双向 | QoS 2 第三步完成 |
| 8 | SUBSCRIBE | C→B | 客户端订阅主题 |
| 9 | SUBACK | B→C | Broker 确认订阅 |
| 10 | UNSUBSCRIBE | C→B | 客户端取消订阅 |
| 11 | UNSUBACK | B→C | Broker 确认取消 |
| 12 | PINGREQ | C→B | 心跳请求 |
| 13 | PINGRESP | B→C | 心跳响应 |
| 14 | DISCONNECT | C→B | 客户端断开 |

**Flags（4bit）** — 仅 PUBLISH 报文使用：

- **DUP**（bit 3）：重发标志。1 = 这条消息是重发的（QoS 1/2 超时没收到确认）
- **QoS**（bit 2-1）：服务质量等级。0 = 最多一次，1 = 至少一次，2 = 恰好一次
- **RETAIN**（bit 0）：保留标志。1 = Broker 保留这条消息，新订阅者订阅时立即收到最后一条保留消息

**Remaining Length（变长编码）**：

不是简单的 2 字节定长，而是用"每个字节的最高位作为延续标志"的变长编码：

```
值范围          编码后字节数
0-127            1 字节
128-16383        2 字节
16384-2097151    3 字节
2097152-268435455 4 字节

编码规则：
do
    encodedByte = value % 128     // 低 7 位
    value = value / 128           // 移除已编码的低 7 位
    if (value > 0)
        encodedByte = encodedByte | 0x80   // 最高位置 1 表示还有后续字节
    output(encodedByte)
while (value > 0)
```

举例：值 = 321 → 321 % 128 = 65, 321 / 128 = 2 → 输出 0xC1（65|0x80）, 输出 0x02 → 两个字节

**设计原因**：MQTT 要适应极低带宽，每字节都要省。64 字节内的短消息只需 1 字节长度头，2KB 内的消息只需 2 字节，而不是浪费在 2 或 4 字节定长上。

### 1.3.2 CONNECT 报文详细结构

这是客户端连接 Broker 时发的第一个报文，结构最复杂：

```
Fixed Header:
  Byte 1: 0x10 (CONNECT, flags=0)
  Byte 2-?: Remaining Length

Variable Header:
  Protocol Name:    "MQTT" (4 字节，UTF-8 编码字符串前有 2 字节长度)
  Protocol Level:   4 (v3.1.1)
  Connect Flags:    1 字节
    bit 7: User Name Flag
    bit 6: Password Flag
    bit 5: Will Retain
    bit 4-3: Will QoS
    bit 2: Will Flag
    bit 1: Clean Session
    bit 0: Reserved
  Keep Alive:       2 字节（秒），最大值 65535

Payload（按顺序，只有 Connect Flags 对应位置位才出现）:
  Client Identifier (UTF-8 字符串，前 2 字节长度)
  Will Topic         (如果有遗嘱)
  Will Message       (如果有遗嘱)
  User Name          (如果有用户名)
  Password           (如果有密码)
```

**Clean Session（bit 1）**：
- `true`（1）：断开时 Broker 丢弃所有未完成的消息和订阅。重新连接是"全新干净"的会话。
- `false`（0）：断开时 Broker 保留订阅和 QoS 1/2 的未完成消息。重新连接时自动恢复——对低功耗传感器很关键，设备休眠醒来后不用重新订阅。

本工程 `mqttsrv(id, false)` → clean_session = false，需要断线重连后自动恢复订阅。

**Keep Alive**：如果这段时间内没有任何报文交互，客户端必须发 PINGREQ，Broker 必须回 PINGRESP。如果 1.5 × Keep Alive 时间内没任何报文，Broker 认为客户端掉线，执行遗嘱。

本工程 keepalive = 50 秒。`test->loop()` 内部自动在到期前发 PINGREQ，上层不用管。

### 1.3.3 SUBSCRIBE 报文 + 主题过滤器

```
Fixed Header: 0x82 (SUBSCRIBE, QoS=1)

Variable Header:
  Packet Identifier: 2 字节（因为 SUBSCRIBE 的 QoS=1，需要 message id 用于 SUBACK 匹配）

Payload:
  Topic Filter 1: UTF-8 字符串（前 2 字节长度 + 实际字符串）
  Requested QoS 1: 1 字节
  Topic Filter 2: ...
  Requested QoS 2: ...
```

**通配符**：订阅时可以包含通配符，但不能在 publish 的 topic 中使用。

- `+`：匹配单层。`/home/+/temperature` 匹配 `/home/room1/temperature` 和 `/home/room2/temperature`，不匹配 `/home/room1/device/temperature`
- `#`：匹配所有剩余层级。必须是最后一个字符。`/home/#` 匹配 `/home` 下的所有主题，包括 `/home/room1/temperature/device`

本工程没有用通配符，直接订阅精确主题 `/ZKGDDEV47`。

### 1.3.4 PUBLISH 报文 — 最常用的报文

```
Fixed Header:
  Byte 1: 0x30 + DUP(1bit<<3) + QoS(2bit<<1) + RETAIN(1bit)
  Byte 2-?: Remaining Length

Variable Header:
  Topic Name: UTF-8 字符串（2 字节长度 + 实际字符串）
  Packet Identifier: 2 字节（只有 QoS > 0 时才会出现）

Payload:
  消息体（任意二进制数据，最大 256MB）
```

**注意**：QoS=0 的 PUBLISH 没有 Packet Identifier。因为不需要确认，不需要用 ID 匹配。

本工程 publish 时用 QoS=2：
```cpp
test->publish(NULL, "MqttServerClient", str.length(), str.c_str(), 2);
//                                                            ↑ QoS=2
```

### 1.3.5 QoS 的三级可靠性

这是 MQTT 区别于其他协议的核心机制。**QoS 是消息级别的，不是连接级别的**——同一条连接上可以同时有 QoS 0、1、2 的消息。

#### QoS 0：最多一次（At most once）

```
发送者 → PUBLISH → Broker → PUBLISH → 接收者
```
发了就不管了。Broker 不存储，不确认。丢就丢了。

**适用场景**：高频传感器数据（温度每秒上报一次，丢一次不影响）。

#### QoS 1：至少一次（At least once）

```
发送者 → PUBLISH(PacketId=123) → Broker → PUBLISH(PacketId=456) → 接收者
发送者 ← PUBACK(123) ──────── Broker ← PUBACK(456) ──────── 接收者

发送者发 PUBLISH 后缓存消息，等 PUBACK。
如果超时没收到 PUBACK → 重发（DUP 位置 1 的重发）。
Broker 收到 PUBACK 后删除缓存。
接收者可能收到重复消息（QoS 1 只保证"至少收到一次"）。
```

#### QoS 2：恰好一次（At most once）

最复杂的四步握手：

```
发送者 → PUBLISH(PacketId=123) ──────→ Broker
发送者 ← PUBREC(123) ───────────────── Broker    // 第一步确认："我收到了"
发送者 → PUBREL(123) ─────────────────→ Broker    // "可以转发给订阅者了"
发送者 ← PUBCOMP(123) ──────────────── Broker    // "转发出去了，这条消息可以忘了"

Broker 和每个订阅者之间也有同样的四步握手（用不同的 PacketId）。
```

**为什么需要 PUBREC → PUBREL → PUBCOMP 三步确认而不是 TCP 那样两次？**

TCP 的 ACK 确认的是"数据到了 TCP 缓冲区"。但 QoS 2 要求的是"消息确实被处理了"（Broker 已经把消息转发给了订阅者）。PUBREC = "我收到了，但我还没转发给订阅者"，PUBREL = "可以转发了"，PUBCOMP = "转发完了"。

**PacketId 的使用规则**：
- QoS=0：没有 PacketId
- QoS=1 和 QoS=2：有 PacketId
- PacketId 是一个会话级别的计数器，每个 PUBLISH 分一个新 ID
- 同一个 PacketId 在 PUBLISH→PUBACK 完成前不能被复用

本工程 publish 全部使用 QoS=2。这意味着：
1. `test->loop()` 内部自动处理 PUBLISH→PUBREC→PUBREL→PUBCOMP 四步
2. 如果网络断开，`loop()` 返回非零值，上层 `reconnect()` 后未完成的消息会被 mosquitto 自动重发
3. 代价是每个 publish 多了 3 次往返

## 1.4 遗嘱消息（Will Message）

CONNECT 时可以设置遗嘱消息。Broker 检测到客户端**异常断开**（Keep Alive 超时但没发 DISCONNECT）时，自动把遗嘱 PUBLISH 到指定主题。

本工程**没有设置遗嘱**（没有调用 `mosquitto_will_set()`）。这意味着设备掉线时云端不会自动知道，需要靠 Keep Alive 超时机制由云端 Broker 侧判断。

## 1.5 保留消息（Retained Message）

PUBLISH 时 `retain=true`，Broker 存储这条消息。之后有新订阅者订阅这个主题时，Broker 立即把最后一条保留消息发给它——新订阅者不用等发布者发。

本工程的 publish 全部 `retain=false`。

## 1.6 MQTT v3.1.1 vs v5.0

| 特性 | v3.1.1（本工程） | v5.0 |
|------|----------------|------|
| 原因字符串 | 无 | 断开/CONNACK 可带原因描述 |
| 会话过期 | 无 | 可设置会话过期时间（替代 Clean Session） |
| 消息过期 | 无 | 消息可设过期时间，过期后 Broker 丢弃 |
| 主题别名 | 无 | 用整数代替长主题名，减少带宽 |
| 请求/响应 | 无 | 内建响应主题和关联数据 |
| 用户属性 | 无 | 键值对元数据 |

本工程使用 v3.1.1，因为 mosquitto 1.4.14 不完全支持 v5.0。

---

# 第二篇：MQTT 在 Linux 网络栈中的位置

## 2.1 完整协议栈

```
┌──────────────────────────────────────────┐
│              应用层                        │
│  ┌────────────────────────────────────┐  │
│  │ 本工程业务逻辑                        │  │
│  │ mqttsrv / mqttThread / mqttMsgParaseThread │
│  │ JSON 解析、手机命令分发、数据上报      │  │
│  ├────────────────────────────────────┤  │
│  │ mosquittopp (C++ wrapper)          │  │  ← 本工程源码目录: 3rdparty/mqtt/
│  │ 虚函数回调：on_message/on_connect   │  │
│  ├────────────────────────────────────┤  │
│  │ libmosquitto (C 库)                │  │
│  │ 协议序列化/反序列化                  │  │
│  │ MQTT 报文打包/解包                  │  │
│  │ PUBLISH/PUBACK/PUBREC/PUBREL/PUBCOMP│  │
│  │ SUBSCRIBE/SUBACK/PINGREQ/PINGRESP  │  │
│  │ Keep Alive 定时器                   │  │
│  │ select() 轮询 socket               │  │
│  ├────────────────────────────────────┤  │
│  │ OpenSSL (libssl.so)                │  │  ← TLS 加密
│  │ TLSv1 握手 + 对称加密传输           │  │
│  ├────────────────────────────────────┤  │
│  │ BSD Socket API                     │  │  ← 系统调用
│  │ socket() / connect() / send() / recv() │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│            传输层 (TCP)                    │
│  端口: 1883 (明文) / 8883 (TLS)           │
│  本工程: 1884 (TLS)                       │
│  保证可靠、有序、面向连接的字节流          │
├──────────────────────────────────────────┤
│            网络层 (IP)                     │
│  cloud.chinaamd.cn → DNS → IP 地址        │
├──────────────────────────────────────────┤
│          数据链路层 / 物理层               │
│  eth0 以太网 / wlan0 WiFi / 4G 模块      │
└──────────────────────────────────────────┘
```

## 2.2 TCP 连接的生命周期

```
客户端 (工控机)                              Broker (cloud.chinaamd.cn:1884)
    │                                               │
    │──TCP SYN──────────────────────────────────→│  socket()
    │←─TCP SYN+ACK───────────────────────────────│  bind(1884)
    │──TCP ACK──────────────────────────────────→│  listen()
    │                                               │  accept()
    │  三次握手完成，TCP 连接建立                     │
    │                                               │
    │──TLS ClientHello──────────────────────────→│  TLS 握手
    │←─TLS ServerHello + 证书────────────────────│  (mosquitto 库 + OpenSSL 自动处理)
    │──密钥交换 + 加密通道建立──────────────────→│
    │                                               │
    │──MQTT CONNECT─────────────────────────────→│  clientId="ZKGDDEV47"
    │←─MQTT CONNACK(rc=0)───────────────────────│  keepalive=50
    │                                               │
    │──MQTT SUBSCRIBE("/ZKGDDEV47")─────────────→│
    │←─MQTT SUBACK──────────────────────────────│
    │                                               │
    │  ............ 正常运行，收发消息 .............. │
    │                                               │
    │──MQTT PINGREQ─────────────────────────────→│  每 50 秒一次（loop() 自动）
    │←─MQTT PINGRESP────────────────────────────│
    │                                               │
    │  ............ 设备关机 ........................ │
    │                                               │
    │──MQTT DISCONNECT──────────────────────────→│
    │──TCP FIN──────────────────────────────────→│
```

## 2.3 mosquitto_loop() 内部到底做了什么

`test->loop()` 是本工程中最关键的调用——mqttThread 在死循环中每 100ms 调一次。它的内部流程：

```
int mosquitto_loop(struct mosquitto *mosq, int timeout, int max_packets)
{
    // 1. 检查是否有数据需要通过 socket 发送
    //    (之前 publish/subscribe 调用的结果排在发送队列中)
    if (有数据待发送) {
        // 尝试 write() 到 socket。如果 socket 缓冲区满，下次 loop 再写
        packet_write(mosq);
    }

    // 2. 用 select() 检查 socket 是否有数据可读
    fd_set rfd;
    FD_ZERO(&rfd);
    FD_SET(mosq->sock, &rfd);
    struct timeval tv = {timeout/1000, (timeout%1000)*1000};
    int rc = select(mosq->sock + 1, &rfd, NULL, NULL, &tv);

    if (rc > 0) {
        // 3. 有数据到达，read() 从 socket 读到内部缓冲区
        //    然后解析 MQTT 报文：
        //    固定头 → 剩余长度 → 可变头 → Payload
        packet_read(mosq);
    }

    // 4. 处理收到的完整报文：
    switch (packet_type) {
    case CONNACK:    → 调 on_connect() 回调
    case PUBLISH:    → 调 on_message() 回调
    case PUBACK:     → 标记对应的 PUBLISH 已完成，从重发队列移除
    case PUBREC:     → 自动回 PUBREL
    case PUBCOMP:    → 标记对应的 PUBLISH 已完成
    case SUBACK:     → 调 on_subscribe() 回调
    case PINGRESP:   → 更新心跳计时器
    }

    // 5. 检查 QoS 1/2 消息的重发超时：
    //    如果某条 PUBLISH 发出去超过 message_retry 秒（默认 20 秒）
    //    还没收到 PUBACK/PUBCOMP → 重发（DUP 位置 1）

    // 6. 检查 Keep Alive：如果距上次发送任何报文已经接近 keepalive 秒，
    //    自动发 PINGREQ

    // 7. 返回：
    //    MOSQ_ERR_SUCCESS = 一切正常
    //    MOSQ_ERR_CONN_LOST = 连接断开（select 返回错误/TCP RST）
    //    MOSQ_ERR_NO_CONN = 还没 connect
    //    其他错误码 = 协议错误
}
```

**关键理解**：
1. `loop()` 内部调用 `select()`，与之前讲的 mynetwork.cpp 中 `MyNetWorkThread::run()` 用的是同一个系统调用。区别是 mosquitto 帮你封装好了——你不需要自己管理 fd_set 和 sockaddr_in。
2. `loop()` 不是只做"收"，而是"收发 + 保活 + 重发"四合一的维护函数。
3. 本工程用 `loop(timeout=-1, max_packets=1)`，timeout=-1 表示使用默认 1000ms 超时。但外层循环有 `msleep(100)`，实际调用频率约 10Hz。
4. `on_message()` 等回调就是在 `loop()` 内部被调用的——**回调的执行线程是 mqttThread**，不是主线程也不是 mqttMsgParaseThread。

---

# 第三篇：TLS 加密 — mosquitto + OpenSSL 集成

## 3.1 TLS 在 MQTT 中的作用

标准的 MQTT 端口 1883 是明文的——任何抓包工具（tcpdump/Wireshark）可以直接看到 JSON 内容。本工程在端口 1884 上使用 TLSv1 加密，整个 TCP 连接被 OpenSSL 包裹。

```
没有 TLS：                    有 TLS：
┌──────────┐                 ┌──────────┐
│ MQTT 明文 │                 │ MQTT 明文 │
├──────────┤                 ├──────────┤
│ TCP 段   │ ← 抓包可见       │ TLS 记录层 │ ← 加密的，抓包看不到
├──────────┤                 ├──────────┤
│ IP 包    │                 │ TCP 段   │
└──────────┘                 ├──────────┤
                              │ IP 包    │
                              └──────────┘
```

## 3.2 TLS 握手过程

```
客户端 (工控机)                              Broker (cloud.chinaamd.cn:1884)
    │                                               │
    │ 1. ClientHello                                 │
    │    TLS 版本: TLSv1                              │
    │    支持的加密套件列表                           │
    │    随机数 client_random                        │
    │──→                                            │
    │                                               │
    │ 2. ServerHello + Certificate + ServerHelloDone │
    │    选定 TLSv1 + 加密套件                       │
    │    随机数 server_random                        │
    │    服务器证书 (含公钥)                         │
    │←──                                            │
    │                                               │
    │ 3. 客户端验证证书：                            │
    │    - 用 root.crt (CA 根证书) 验证服务器证书签名 │
    │    - tls_insecure_set(true) 跳过了域名验证     │
    │    验证通过 → 生成 pre_master_secret           │
    │    用服务器公钥加密 pre_master_secret          │
    │                                               │
    │ 4. ClientKeyExchange + ChangeCipherSpec       │
    │    加密的 pre_master_secret                    │
    │    "之后都用协商好的密钥加密"                  │
    │──→                                            │
    │                                               │
    │ 5. ChangeCipherSpec + Finished                 │
    │←──                                            │
    │                                               │
    │ ===== 加密通道建立，后续通信用对称加密 =========│
    │  对称密钥 = f(client_random, server_random,     │
    │                pre_master_secret)              │
```

## 3.3 本工程的 TLS 配置逐行分析

```cpp
// mqttsrv.cpp:93-95

// 1. 设置 TLS 选项：证书验证级别=1(SSL_VERIFY_PEER)，协议=TLSv1
test->tls_opts_set(1, "tlsv1", NULL);
//                 ↑    ↑        ↑
//     cert_reqs=1   TLSv1    ciphers=NULL(用默认)

// SSL_VERIFY_PEER(1)：要求验证服务器证书
// SSL_VERIFY_NONE(0)：不验证（明文+加密都没意义）

// 2. 跳过证书的域名验证
test->tls_insecure_set(true);
// 设为 true 的原因：设备通过 IP 直连或内网域名连 Broker，
// 证书上的 CN (Common Name) 通常是一个公网域名 (cloud.chinaamd.cn)，
// 严格验证会导致连接失败。
// 
// 安全风险：理论上可以被中间人攻击——但物理隔离的工厂内网风险极低。
// 证书本身的签名验证仍然有效（确保服务器是持有合法证书的）。

// 3. 加载 CA 根证书
rc = test->tls_set("./root.crt", NULL);
//                  ↑ CA 证书文件   ↑ capath=NULL(不用目录)
// root.crt 是 CA 的根证书，用于验证 Broker 发来的服务器证书
// 这个文件预置在设备文件系统中
```

**root.crt 的位置**：`./root.crt` 表示当前工作目录。设备启动时的工作目录通常是 `/app/` 或可执行文件所在目录。这个文件需要在设备制造时预置或通过 OTA 推送。

---

# 第四篇：mosquitto 库源码分析

## 4.1 库的组成

```
3rdparty/mqtt/
├── libmosquitto.so.1        # C 核心库动态链接文件
├── libmosquittopp.so.1      # C++ wrapper 动态链接文件
└── libmosquittopp.so        # 符号链接 → libmosquittopp.so.1
```

头文件：
- `bus/mosquitto.h` — C API，约 1500 行，140+ 个函数/类型声明
- `bus/mosquittopp.h` — C++ wrapper，约 105 行，`mosqpp::mosquittopp` 类

**注意**：本工程没有编译 mosquitto 源码，而是直接链接预编译的 `.so` 文件。源码在 mosquitto 项目本身（https://github.com/eclipse/mosquitto）。

## 4.2 mosquittopp — C++ wrapper 的设计

```cpp
namespace mosqpp {

class mosquittopp {
private:
    struct mosquitto *m_mosq;  // 持有 C 库的不透明指针

public:
    mosquittopp(const char *id=NULL, bool clean_session=true);
    virtual ~mosquittopp();

    // --- 虚函数回调（子类重写） ---
    virtual void on_connect(int rc) { return; }
    virtual void on_disconnect(int rc) { return; }
    virtual void on_publish(int mid) { return; }
    virtual void on_message(const struct mosquitto_message *message) { return; }
    virtual void on_subscribe(int mid, int qos_count, const int *granted_qos) { return; }
    virtual void on_unsubscribe(int mid) { return; }
    virtual void on_log(int level, const char *str) { return; }
    virtual void on_error() { return; }

    // --- 核心方法（透传 C 库函数） ---
    int connect(const char *host, int port=1883, int keepalive=60);
    int reconnect();
    int publish(int *mid, const char *topic, int payloadlen, const void *payload, int qos=0, bool retain=false);
    int subscribe(int *mid, const char *sub, int qos=0);
    int loop(int timeout=-1, int max_packets=1);
    int tls_set(const char *cafile, ...);
    int tls_opts_set(int cert_reqs, ...);
    int tls_insecure_set(bool value);
    // ... 其他方法
};
}
```

**设计模式**：模板方法（Template Method）。mosquittopp 定义了事件循环的骨架（connect → loop → on_message），子类重写虚函数来处理业务逻辑。

**本工程的继承关系**：
```cpp
class mqttsrv : public mosqpp::mosquittopp
{
    // 重写了 on_message —— 收到消息后加锁放入 recvMsgList
    void on_message(const struct mosquitto_message *message) override;

    // 其他回调都是空函数体（在头文件内联定义: on_connect/on_disconnect/on_publish）
    // 只打印一行 qDebug() 日志
};
```

## 4.3 loop() 的两种使用模式

**模式 1：单线程轮询（本工程使用的）**
```cpp
while (1) {
    rc = test->loop();      // 在主循环中反复调用
    if (rc) { /* 处理断线 */ }
    msleep(100);
}
```
优点：自己控制循环节奏，可以插入其他逻辑（如检查标志位）。缺点：需要自行管理重连。

**模式 2：独立线程（mosquitto 内置）**
```cpp
test->loop_start();  // 启动独立线程，自动调用 loop()
// 线程安全，可以在其他线程中 publish
// 退出时: test->disconnect(); test->loop_stop();
```
本工程没用这个模式——因为本工程已经自己管理 mqttThread，不需要 mosquitto 再开一个线程。

## 4.4 线程安全注意事项

mosquitto.h 第 109-117 行的注释明确说明：

> libmosquitto provides thread safe operation, with the exception of mosquitto_lib_init which is not thread safe.

> If your application uses threads you must use mosquitto_threaded_set to tell the library this is the case.

本工程中，`publish()` 在 **mqttMsgParaseThread** 中被调用（`onCmdReturn()`），而 `loop()` 在 **mqttThread** 中被调用——两个不同的线程。mosquitto 库内部使用互斥锁保护数据结构，允许这种跨线程调用。但本工程没有调用 `mosquitto_threaded_set()`——这可能是个潜在的线程安全隐患。

---

# 第五篇：本工程 MQTT 架构全览

## 5.1 三件套 + 启动入口

```
globalflow.cpp:10057-10066

myHttpFileClient = new MyHttpFileClient;           // HTTP 客户端（Token/上传）
myMqttThread = new mqttThread;                      // MQTT 连接线程
myMqttMsgParaseThread = new mqttMsgParaseThread;     // MQTT 消息解析线程

if (ipForInterface == "162.254.129.100") {          // 只有 eth0 有正确 IP 才启动
    myMqttThread->start();
    myMqttMsgParaseThread->start();
}
```

全局指针（`mqttsrv.cpp:6-8`）：
```cpp
mqttThread *myMqttThread;                  // 全局可访问
mqttMsgParaseThread *myMqttMsgParaseThread; // 全局可访问
```

## 5.2 三件套的职责划分

```
┌── mqttThread（QThread, while死循环模式）────────────┐
│ 职责：网络连接层 — 只管 TCP/TLS/MQTT 的连接维护      │
│                                                      │
│ run():                                               │
│   1. ping 外网 (最多15次, 每次等20秒)                │
│   2. mosqpp::lib_init()                              │
│   3. TLS 配置 + tls_set(./root.crt)                  │
│   4. test->connect(cloud.chinaamd.cn, 1884, 50)     │
│   5. test->subscribe( "/ZKGDDEV47" )               │
│   6. ntpdate 网络授时                                │
│   7. while(1) { test->loop(); msleep(100); }        │
│      断线自动 reconnect + 重新 subscribe             │
│                                                      │
│ 成员：                                               │
│   mqttsrv *test  — MQTT 客户端对象                  │
│   int networkConnectIsOk                            │
│   QString topic — "/ZKGDDEV47"                      │
└──────────────────────────────────────────────────────┘
          │
          │ recvMsgList (QList<string> + QMutex)
          │
┌──┴─ mqttMsgParaseThread（QThread, for(;;)死循环）───┐
│ 职责：业务逻辑层 — 解析 JSON、分发命令、数据上报     │
│                                                      │
│ run():                                               │
│   1. sleep(10) 等 mqttThread 先连上                  │
│   2. Token 认证 (最多重试10次)                       │
│   3. 初始数据上报 (uploadLocalRemoteInfo →           │
│      upLoadSchemeInfo → uploadPara →                 │
│      uploadMachinePara → uploadLogFile)             │
│   4. for(;;) {                                       │
│        - Token 过期续期                              │
│        - 轮询 20+ 个标志位 → 执行对应上传           │
│        - 轮询 recvMsgList → doParseMessage()        │
│        - msleep(200)                                 │
│      }                                               │
│                                                      │
│ 20+ 个上传标志位（int 型）：                         │
│   n_uploadPartsStatusFlag, n_uploadAlarmFlag,        │
│   n_uploadRealTimeParaFlag, n_uploadImageFlag,       │
│   n_uploadWaveFlag, n_uploadBlowCountsFlag,          │
│   n_uploadStatisticsFlag, n_uploadLogFileFlag ...    │
│                                                      │
│ 信号（跨线程 → 主线程）：                            │
│   cmd_OpenSwitch, cmd_Feed, cmd_updatezksort_ask     │
│   StartRemoteControl, EndRemoteControl               │
└──────────────────────────────────────────────────────┘
          │
          │ HTTP (HTTPS)
          │
┌──┴── MyHttpFileClient ──────────────────────────────┐
│ 职责：HTTP/HTTPS 网络 I/O                            │
│                                                      │
│ - requestTokenUpdate() — OAuth2 设备认证             │
│ - upLoadData(url, json) — 通用 HTTP POST            │
│ - requestOssUploadSign() — 申请阿里云 OSS 签名       │
│ - upLoadOssFile(path) — 上传文件到阿里云 OSS         │
│ - httpDownloadManager — 断点续传下载                │
│ - g_connectName — 设备编号 "ZKGDDEV47"               │
│ - g_token — OAuth2 Bearer Token                      │
└──────────────────────────────────────────────────────┘
```

## 5.3 数据流：从手机到设备，再回传到手机

```
手机微信小程序
  │ MQTT Publish
  │ Topic: "MqttServerClient" (发给云端)
  │ Payload: JSON {cmd:[{cmdCode:"CMD_Feed", content:"50:20:...", uuid:"xxx"}]}
  ↓
cloud.chinaamd.cn:1884 (MQTT Broker)
  │ 云端服务器订阅 "MqttServerClient"，收到 JSON
  │ 云端解析 cmdCode，发现是对设备 ZKGDDEV47 的命令
  │ 云端 Publish 到 "/ZKGDDEV47"
  ↓
mqttThread::loop() → 收到 PUBLISH → on_message()
  │ 校验 topic 匹配 "/ZKGDDEV47"
  │ strRcv = (char*)message->payload  ← JSON 字符串
  │ QMutexLocker lock(&m_paraseLock)
  │ recvMsgList.append(strRcv)
  │
  │ ===== 以上在 mqttThread =====
  │
mqttMsgParaseThread::run() 主循环
  │ for(;;) {
  │   while (!recvMsgList.isEmpty()) {
  │     msg = recvMsgList[0]
  │     doParseMessage()
  │     recvMsgList.removeAt(0)
  │   }
  │   msleep(200)
  │ }
  │
  │ ===== 以上在 mqttMsgParaseThread =====
  │
doParseMessage() 内部：
  1. JSON 解析：utfRec = "{\"cmd\":" + msg + "}"
     Json::Reader → root → root["cmd"][i]
  2. 提取字段：cmdCode, content, uuid
  3. 特殊命令先处理：CMD_LockStatusUpload, CMD_ConnectVPN
  4. ★ 先回 ACK：onCmdReturn(uuid, cmd, RECEIVE=13)
  5. 状态安全检查（正在清灰？正在采图？首次运行？特殊页面？）
  6. 按 cmdCode 分发到具体函数（onCmdFeed/onOpenSwitch/...）
  7. 执行成功 → onCmdReturn(uuid, cmd, SUCCESS=0)
     ├── Json 组装 {uuid, devId, cmdCode, result, type:"push"}
     └── test->publish(NULL, "MqttServerClient", ..., 2)
         └── QoS=2 → Broker → 云端服务器 → 微信小程序
```

## 5.4 线程跨越点汇总

| 操作 | 执行线程 | 备注 |
|------|---------|------|
| test->loop() 收包 | mqttThread | mosquitto 内部 select+recv |
| on_message() 回调 | mqttThread | 只做 topic 校验 + 加锁入队 |
| recvMsgList 写入 | mqttThread | QMutexLocker 保护 |
| recvMsgList 读取 | mqttMsgParaseThread | QMutexLocker 保护 |
| doParseMessage() | mqttMsgParaseThread | JSON 解析 + 命令分发 |
| 业务函数(onCmdFeed等) | mqttMsgParaseThread | 读写全局结构体 + 串口发送 |
| onCmdReturn() publish | mqttMsgParaseThread | 跨线程调 test->publish() |
| emit StartRemoteControl | mqttMsgParaseThread | Qt AutoConnection → 主线程 |
| DealStartRemoteControl() | 主线程 | 弹出锁屏确认对话框 |
| upload 函数 (HTTP) | mqttMsgParaseThread | 调 myHttpFileClient 的同步 HTTP |
| n_uploadXxxFlag 写 | 任意线程 | 其他线程设 1 |
| n_uploadXxxFlag 读 | mqttMsgParaseThread | run() 主循环检查 |

---

# 第六篇：20+ 远程命令逐条分析

## 6.1 命令表总览

| cmdCode | 处理函数 | 操作类型 | 硬件路径 | 回传次数 |
|---------|---------|---------|---------|---------|
| CMD_OpenSwitch | onOpenSwitch | 控制 | myFlow.onOff() | 2 (ACK + result) |
| CMD_OpenSwitchs | onOpenSwitchs | 控制 | myFlow.resetFeederRG() | 2 |
| CMD_Feed | onCmdFeed | 控制 | myFlow.resetFeederRG() | 2 |
| CMD_MemSet | onMemSet | 控制 | 串口发送控制板参数 | 2 |
| CMD_ShutDown | onShutDown | 控制 | system("shutdown -h now") | 2 (先回成功再关机) |
| CMD_SaveCfg | onSaveCfg | 控制 | g_Runtime().save() | 2 |
| CMD_GlobalParaSet | onGlobalParaSet | 配置 | 写全局结构体 + 串口发送 | 2 |
| CMD_ModelParamsSet | onModelParamsSet | 配置 | 写模型参数 | 2 |
| CMD_CaptureImage | onCaptureImage | 数据采集 | UDP 通道采图 | 2 |
| CMD_CaptureWave | onCaptureImage | 数据采集 | UDP 通道采波形 | 2 |
| CMD_CaptureAllImage | onCaptureAllImage | 数据采集 | 全通道扫描采图 | 2 |
| CMD_RemoteControl | onRemotecontrol | 控制 | emit StartRemoteControl | 仅在确认后回传 |
| CMD_PushVPNCert | onVPNPushCert | 配置 | 写 /sdcard/cfg/zk.ovpn | 2 |
| CMD_ConnectVPN | onVPNConnect | 控制 | system("openvpn ...") | 2 |
| CMD_Shell_Cmd | onShellCmd | 控制 | g_Runtime().mySystem() | 2 |
| CMD_UpdateZKSort | onUpdateZKSort | OTA | emit cmd_updatezksort_ask | 通过信号触发 |
| CMD_UpdateIpcModel | onUpdateZKSort | OTA | emit cmd_updatezksort_ask | 通过信号触发 |
| CMD_ParaUpload | onParaUpload | 数据上报 | 只设标志位(空函数体) | 1 (异步上报) |
| CMD_RealTimeParaUpload | uploadRealTimePara | 数据上报 | HTTP POST | 1 |
| CMD_LogfileUpload | onLogfileUpload | 数据上报 | HTTP 上传日志文件 | 2 |
| CMD_CnffileUpload | onCnffileUpload | 数据上报 | tar 打包 → OSS 上传 | 2 |
| CMD_WipeSpeedUpload | onWipeSpeedUpload | 数据上报 | HTTP POST | 2 |
| CMD_BlowCountsUpload | onBlowCountsUpload | 数据上报 | UDP → HTTP | 2 |
| CMD_VavleAudioUpload | onVavleAudioUpload | 数据上报 | OSS 上传 .wav | 2 |
| CMD_IpcInfoUpload | onIpcInfoUpload | 数据上报 | emit cmd_uploadIpcInfo_ask | 通过信号触发 |
| CMD_ModelParamsUpload | onModelParamsUpload | 数据上报 | HTTP POST | 2 |
| CMD_LockStatusUpload | doParseMessage 内联 | 状态查询 | 只查结构体 | 1 |

## 6.2 CMD_Feed（供料调节）— 从云端到振动器的完整路径

```
手机 App 设供料值 "50:20:30:80:50:20:30:80:50:20"

→ JSON: {"cmdCode":"CMD_Feed","content":"50:20:30:80:50:20:30:80:50:20","uuid":"xxx"}
→ mqttMsgParaseThread::doParseMessage()
  → onCmdFeed("50:20:30:80:50:20:30:80:50:20")
    1. "50:20:30:..." → split(":") → ["50","20","30",...]
    2. 校验 len == struCnfg.nChuteTotal (通道数，如 10)
    3. for i=0..9:
         struCnfp.struGroupCtrl[struGsh.ctrlboardIndex].nFeederValueMain[i] = vecList[i].toInt()
       → nFeederValueMain[0]=50, [1]=20, [2]=30, ...
    4. myFlow.resetFeederRG()
         → 从 nFeederValueMain[] 读取新值
         → 组装串口命令帧（CMD_FEED_SET 命令字 + 板卡地址 + 10 个通道的值）
         → mySerial.pushCom1CmdData(帧数据, 帧长度)
         → 串口发送线程 → COM1 → PLC/控制板
         → PLC 改变 PWM 占空比/DAC 输出 → 振动器振幅改变
    5. return SUCCESS
  → uploadPara()  ← 上传最新参数到云端
  → onCmdReturn(uuid, "CMD_Feed", 0) ← 成功回传
    → test->publish(NULL, "MqttServerClient", JSON, QoS=2)
    → Broker → 云端 → 手机显示"供料调节成功"
```

## 6.3 CMD_GlobalParaSet — 最复杂的参数设置

该命令接收一个 JSON 字符串，包含视角、算法的层级结构：

```json
{
  "sortSerial": 0,
  "viewData": [{
    "view": 0,
    "algs": [
      {"id": 0, "value": 25, "state": 1},   // 灰度A：灵敏度25，使能开启
      {"id": 3, "value": 120, "state": 1},  // 灰度D：灵敏度120
      {"id": 6, "value": 100, "state": 1}   // 异色A：灵敏度100
    ]
  },{
    "view": 1,
    "algs": [{"id": 1, "value": 27, "state": 1}]
  }]
}
```

onGlobalParaSet 内部处理流程：
1. JSON 解析 → 遍历 sortSerial → view → algs
2. 根据 `arithid` 分发到不同的算法参数结构体：
   - ARITH_GREY_A~D → struGreyColor[n].nSensLow/nSensUp
   - ARITH_DISCOLOR_A/B, ARITH_CROSS → 值域缩放 (value/100 * MAX_SENS)
   - ARITH_SVM_A~D → struSvm[n].nSens
   - ARITH_BIG_SMALL → struBigSmall[0].nAreaLow/nAreaUp
   - ARITH_LONG_SHORT → struLongShort[0].nLengthLow/nLengthUp
   - ARITH_CIRCLE_LONG → struCircleLong[0].nCiloLow/nCilosUp
3. 根据 `nMatCopySetMode` 判断是否需要镜像到后视（整机相同/前后相同）
4. 调用 `myFlow.materialParamsCopySendAssemble()` → 串口发送给 FPGA
5. 调用 `myFlow.materialArithEnableCopy()` → 更新算法使能标志

## 6.4 CMD_ShutDown — 远程关机

```cpp
void onShutDown() {
    if (struGsh.bSortStart == 1) {
        myFlow.onOff();  // 先停止色选
    }
    #ifndef Q_OS_WIN
    struGsh.bFlagPowerOff = 1;
    system("shutdown -h now");  // Linux 立即关机
    #endif
}
```

注意 `doParseMessage()` 中的特殊处理——先回 ACK = SUCCESS，再调 onShutDown()：
```cpp
// 关机命令收到时先回传成功标识
onCmdReturn(uuid, tmp, SUCCESS);  // ← 先告诉手机"收到了"
onShutDown();                       // ← 然后关机
```
如果先关机再回传，publish 永远发不出去——手机不知道关机命令是否执行了。

## 6.5 CMD_RemoteControl — 远程锁屏控制

```cpp
int onRemotecontrol(int cmd_value, int time, QString phonenum)
{
    if (cmd == 1) {               // 开启远程控制
        if (time < 0)             // 控制时间为 0 → 参数错误
            errorcode = CMD_PARA_ERROR;
        else if (当前已在远程控制页 && phone != userphonenum)  // 被他人占用
            errorcode = OCCUPY_ERROR;
        else {
            struGsh.remoControlmode = 2;  // 等待确认状态
            struGsh.remoControltime = time;
            userphonenum = phonenum;
            emit StartRemoteControl(phonenum);  // → 主线程弹出锁屏确认框
        }
    }
    else if (cmd == 0) {          // 断开远程控制
        emit EndRemoteControl();
    }
}
```

**双向确认流程**：
1. 手机发 CMD_RemoteControl, cmd=1, 控制时长 time=N
2. 设备端弹出锁屏确认对话框 → 现场操作员点"同意"或"拒绝"
3. 如果同意 → `n_uploadRemoteControlFlag = 1` → 主循环检测后 `uploadRemoteControl()` → HTTP 上传确认结果
4. 如果拒绝 → `struGsh.remoteControlstatus = USERREJECT` → onCmdReturn 回传错误码
5. 控制时间到期 → `EndRemoteControl` 自动断开

## 6.6 CMD_UpdateZKSort — 上位机 OTA 升级

```cpp
void onUpdateZKSort(string cmd_value)
{
    // cmd_value = "https://xxx.com/zksort.bin::md5hash::v2.1.0::1"
    //                                        ↑ URL      ↑ MD5   ↑ 版本  ↑ force

    QStringList vecList = strVec.split(QRegExp("::"));
    // vecList[0] = URL, [1] = MD5, [2] = 版本号, [3] = force

    UpdateUnit unit;
    unit.url = vecList.at(0);
    unit.md5 = vecList.at(1);
    unit.ver = vecList.at(2);
    unit.force = vecList.at(3).toInt();

    // MD5 相同 → 已是最新版本，不升级
    if (g_Runtime().getFileMd5("/media/mmcblk0p1/3-app.bin") == unit.md5) {
        onCmdReturn(m_uuid, m_tmp, VERSIONSAME);
        return;
    }

    // 发射信号 → 主线程弹出升级确认对话框
    QVariant m_unit = QVariant::fromValue(unit);
    emit cmd_updatezksort_ask(m_unit);
}
```

**升级流程**（信号触发后的操作在另一个文件中，如 `machineinfowidget.cpp`）：
1. 手机下发升级 URL + MD5 + 版本号
2. 本地 MD5 比对 → 相同则跳过
3. 弹框让现场操作员确认
4. 用户同意 → httpDownloadManager 断点续传下载 `/media/mmcblk0p1/3-app.bin`
5. 下载完成 → MD5 校验 → 替换可执行文件 → 重启

---

# 第七篇：标志位驱动的异步上传机制

## 7.1 为什么用标志位而不是信号槽

信号槽是"即时通知"——emit 之后槽立即（或排队）执行。但数据上传不需要即时性：
- 每秒上报一次实时参数？晚几秒无所谓
- 日志文件每天 23:59:30 上传一次？更不需要实时
- 喷阀测试完成后上传吹气次数？等 200ms 没问题

用标志位的好处：
- 生产者只需要 `flag = 1`，一行代码，不需要关心 connect 关系
- 消费者（主循环）统一调度，可以控制上传频率（避免瞬间大量 HTTP 请求）
- 去耦合——任何代码都可以设标志位，不需要持有 `mqttMsgParaseThread` 的指针

## 7.2 主循环的调度逻辑

```cpp
for (;;) {
    sleep(1);  // 每秒一轮
    reportFlag = Normal_Send;  // 默认标记为"主动上报"

    // Token 过期检查
    if (g_token_expires < struGsh.nCounter - m_nextWeek) {
        requestTokenUpdate(URL_TOKEN_REQUEST);  // 续期
    }

    // 依次检查 20+ 个标志位
    if (n_uploadPartsStatusFlag)      uploadIpcPartsStatusInfo();
    if (n_uploadAlarmFlag)            uploadParaAlarm();
    if (n_uploadRealTimeParaFlag)     uploadRealTimePara();
    if (n_uploadStatisticsFlag)       uploadStatistics();
    if (n_uploadBlowCountsFlag)       uploadBlowCounts();
    if (n_uploadLogFileFlag)          uploadLogFile(today);
    if (n_uploadRemoteControlFlag)    uploadRemoteControl();
    // ... 共 20+ 个

    // 处理收到的 MQTT 消息
    while (!recvMsgList.isEmpty()) {
        msg = recvMsgList[0];
        doParseMessage();
        recvMsgList.removeAt(0);
    }
    msleep(200);
}
```

**关键设计**：
- `sleep(1)` 在循环开头——给其他线程 1 秒时间设置标志位
- `reportFlag` 在每次循环开始时重置为 Normal_Send，处理消息时改为 Request_Send
- 上传函数内部先清零对应的标志位（`n_uploadXxxFlag = 0`），再执行 HTTP。如果 HTTP 失败，标志位已清零——不会死循环重试。依赖外部的重试机制（如 `cacheLocalRemoteInfo` 缓存后补发）

## 7.3 上传函数的容错设计

```cpp
void uploadPara() {
    // 1. 构造大 JSON（几百行，包含全部算法参数）
    // 2. 检查网络：
    if (!struGsh.bFlagMqttConnect) {
        // 网络不通 → 缓存到本地，等网络恢复后补发
        cacheLocalRemoteInfo(URL_STATUSINFO_UPLOAD, str);
        return;
    }
    // 3. HTTP POST
    if (!myHttpFileClient->upLoadData(URL_STATUSINFO_UPLOAD, str)) {
        // 发送失败 → 缓存（reportFlag+2 = 主动/请求补发）
        cacheLocalRemoteInfo(URL_STATUSINFO_UPLOAD, str);
    }
}
```

**本地缓存机制**（`cacheLocalRemoteInfo` → `uploadLocalRemoteInfo`）：
- 网络不通或上传失败时，把 JSON + URL 写入本地文件
- 程序下次启动时，`uploadLocalRemoteInfo()` 读取缓存文件 → 逐个重发 → 删除已成功的
- 保证"设备离线期间产生的数据，联网后不丢失"

---

# 第八篇：嵌入式部署要点

## 8.1 文件依赖

设备上需要的文件：
```
/app/zksort                      # 主可执行文件（编译时 -lmosquitto -lmosquittopp）
/app/root.crt                    # CA 根证书（TLS 验证 Broker）
/lib/libmosquitto.so.1           # mosquitto C 库
/lib/libmosquittopp.so.1         # mosquitto C++ 库
/lib/libssl.so.1.0.0             # OpenSSL（mosquitto 的 TLS 依赖）
/lib/libcrypto.so.1.0.0          # OpenSSL 加密库

/sdcard/cfg/CFG_APPSet.ini       # 配置文件，包含 connectName 和 testIp
/sdcard/cfg/zk.ovpn              # VPN 配置文件（CMD_PushVPNCert 写入）
```

## 8.2 编译链接

zksort.pro 中需要添加：
```qmake
LIBS += -L$$PWD/../3rdparty/mqtt -lmosquitto -lmosquittopp
LIBS += -lssl -lcrypto
```

## 8.3 网络要求

- **出站方向**：工控机 → cloud.chinaamd.cn:1884（MQTT TLS），cloud.chinaamd.cn:8900（HTTP API）
- **不需要入站**：不需要公网 IP，不需要端口映射。MQTT 是客户端主动连 Broker，出站连接不受 NAT 影响。
- **DNS**：需要能解析 cloud.chinaamd.cn
- **NTP**：需要能访问 ntp.ntsc.ac.cn 或 0.pool.ntp.org（时间同步，TLS 证书验证需要正确时间）

## 8.4 安全考虑

| 层面 | 措施 | 缺口 |
|------|------|------|
| 传输 | TLSv1 加密 | `tls_insecure_set(true)` 跳过了域名验证 |
| 认证 | OAuth2 Bearer Token | Token 写入配置文件明文存储 |
| 授权 | connectName 作为设备标识 | 没有每个命令的独立授权 |
| 指令 | CMD_Shell_Cmd 可执行任意 shell | 风险极高——需确保云端已做权限控制 |

## 8.5 mosquitto Broker 侧的配置（如需自建测试环境）

如果要在本地搭建测试 Broker（如 mosquitto 命令行工具）：

```bash
# 安装 mosquitto (Ubuntu)
sudo apt-get install mosquitto mosquitto-clients

# 启动 Broker（前台，便于调试）
mosquitto -v -p 1883

# 订阅设备回包主题（模拟云端）
mosquitto_sub -h localhost -p 1883 -t "MqttServerClient" -v

# 向设备发命令（模拟手机）
mosquitto_pub -h localhost -p 1883 -t "/ZKGDDEV47" -m '{"cmd":[{"cmdCode":"CMD_Feed","content":"50:20:30:80:50","uuid":"test-001"}]}'
```

本工程需要修改 `testIp` 为 `localhost` 并关掉 TLS（注释掉 `tls_opts_set`/`tls_set`）才能在本地测试。

## 8.6 关键调试手段

1. **mosquitto_sub 监控**：在另一台机器上订阅 `#` 通配符主题，查看所有 MQTT 消息流
2. **tcpdump 抓包**：`tcpdump -i eth0 port 1884 -w mqtt.pcap` → Wireshark 分析 TLS 握手和 MQTT 报文
3. **日志**：本工程用 log4qt 和 qDebug() 输出关键节点——`on_connect`、`on_message`、`doParseMessage` 都有日志
4. **Broker 日志**：mosquitto Broker 的 `-v` 参数打印所有连接/断开/订阅/发布事件

---

# 附录

## A. 错误码速查

| 码 | 宏 | 含义 |
|----|-----|------|
| 0 | SUCCESS | 成功 |
| -1 | SYSTEM_ERROR | 系统错误（如首次运行未完成） |
| 2 | UNKNOWN_CMD | 未知命令 |
| 3 | CMD_PARA_ERROR | 参数格式错误 |
| 6 | MEM_SETING | 正在切换方案中 |
| 8 | FEED_ING | 正在供料中 |
| 9 | NOT_MAIN_WND | 当前页面不允许远程操作 |
| 10 | CLEAN_ING | 正在清灰中 |
| 12 | ALARM_PRESSUER | 气压报警，不允许下料 |
| 13 | RECEIVE | 指令已收到（ACK，非最终结果） |
| 15 | VERSIONSAME | 固件版本相同，不升级 |
| 16 | USERREJECT | 现场操作员拒绝远程控制 |
| 20 | M_TIMEOUT | 操作超时 |
| 22 | NETWORK_ERR | 网络异常 |
| 35 | LOCKED_ERROR | 当前未锁屏（远程控制需要先锁屏） |
| 36 | OCCUPY_ERROR | 已被其他手机远程控制 |
| 37 | REMOTE_CANCEL | 远程取消 |
| 38 | NOT_USER_WINDOW | 不在用户权限窗口 |

## B. 源文件索引

| 文件 | 行数 | 内容 |
|------|------|------|
| CameraTest(aiuse)/zksort/bus/mqttsrv.h | 289 | 三个类定义 + ErrorCode 枚举 |
| CameraTest(aiuse)/zksort/bus/mqttsrv.cpp | ~3700 | 全部实现 |
| CameraTest(aiuse)/zksort/bus/mosquitto.h | 1536 | libmosquitto C API |
| CameraTest(aiuse)/zksort/bus/mosquittopp.h | 107 | mosquittopp C++ wrapper |
| CameraTest(aiuse)/zksort/global/globalflow.cpp:10057 | MQTT 线程创建入口 |
| CameraTest(aiuse)/zksort/bus/myhttpfileclient.h/cpp | HTTP 客户端 |
| CameraTest(aiuse)/3rdparty/mqtt/libmosquitto.so.1 | 预编译 C 库 |
| CameraTest(aiuse)/3rdparty/mqtt/libmosquittopp.so.1 | 预编译 C++ wrapper |
