# MQTT 全链路深度学习 — 基础知识清单

按在 [[MQTT主文档]] 中首次出现的顺序排列。

---

## 第一篇：MQTT 协议规范

### OASIS（§1.1）

OASIS = Organization for the Advancement of Structured Information Standards（结构化信息标准促进组织）。一个非营利技术标准联盟，负责维护 MQTT、ODF（OpenDocument）、SAMLL 等标准。MQTT v3.1.1 于 2014 年 10 月成为 OASIS 标准。

### MQTT v3.1.1 vs v5.0（§1.1）

MQTT v3.1.1（2014）：本工程使用的版本。OASIS 标准的第一版，也是目前最广泛部署的版本。

MQTT v5.0（2019）：重大更新。新增了原因字符串（断开时告知原因）、会话/消息过期时间、主题别名（减少带宽）、请求/响应模式（在 PUBLISH 中嵌入 Response Topic）、用户属性（自定义键值对元数据）、共享订阅（多个订阅者负载均衡）、流控制等。

### 发布/订阅模型（§1.2）

一种消息通信范式。发布者（Publisher）把消息发送到一个命名通道（Topic），订阅者（Subscriber）向 Broker 声明自己对某个 Topic 感兴趣，Broker 负责把消息从发布者路由到所有匹配的订阅者。

与请求/响应模型的本质区别：
- 请求/响应：通信双方直接耦合。客户端必须知道服务器地址，服务器必须在线。
- 发布/订阅：通信双方完全解耦。发布者不知道订阅者是谁、有几个、在线与否。订阅者同理。

三种解耦维度：
1. 空间解耦：双方不需要知道对方的 IP/端口
2. 时间解耦：双方不需要同时在线（Broker 存储 QoS 1/2 的消息）[[QoS详解]]
3. 同步解耦：发布和消费是异步的

### Topic（主题）（§1.2）

MQTT 的消息寻址方式。UTF-8 编码的字符串，层级用 `/` 分隔。

设计约束：
- 至少 1 个字符
- 区分大小写
- 可以包含空格
- 不能包含 NULL 字符（U+0000）
- 前导 `/` 会创建一个空层级（不推荐）

Topic 不是预创建的——publish 到某个 topic 时 Broker 自动创建。也没有"删除 topic"的概念——没有订阅者时消息被丢弃。

### 变长编码（Remaining Length）（§1.3.1）

一种用最少的字节表示整数的编码方式。每个字节的低 7 位是数据位，最高位（bit 7）是延续标志：
- bit 7 = 1：表示后面还有一个字节
- bit 7 = 0：表示这是最后一个字节

举例：值 321
```
321 / 128 = 2（商），321 % 128 = 65（余数）
→ 第一个字节：65 | 0x80 = 0xC1（193，最高位置 1 表示还有后续）
→ 商 2 < 128：第二个字节：0x02（最高位为 0，表示结束）
→ 编码结果：C1 02
```

解析时：
```
读 0xC1 → 低7位 = 0x41 = 65, 最高位=1表示继续
读 0x02 → 低7位 = 0x02 = 2, 最高位=0表示结束
值 = 65 + 2 × 128 = 321
```

### DUP 标志（§1.3.1）

PUBLISH 报文固定头的 bit 3。当客户端/Broker 重发一条 QoS 1 或 QoS 2 的消息时，DUP 位置为 1。接收者用 DUP 判断是否是重复消息——但 MQTT 协议不强制接收者基于 DUP 做去重处理，这是应用层的责任。

### CONNECT 报文（§1.3.2）

客户端连接 Broker 时发送的第一个 MQTT 报文。Broker 只接受一次 CONNECT——如果已经连接后再次发送 CONNECT，Broker 必须断开连接。

重要字段：
- **Client Identifier（ClientId）**：每个客户端的唯一标识，最大 23 字符。Broker 用它区分不同客户端。如果 Clean Session=false，Broker 用 ClientId 查找之前的会话状态。
- **Protocol Level**：v3.1 = 3，v3.1.1 = 4，v5.0 = 5
- **Clean Session**：决定断开后是否保留订阅和未完成消息
- **Keep Alive**：心跳间隔（秒），0 表示不禁用心跳（Broker 不会因超时断开）

### SUBSCRIBE 报文（§1.3.3）

注意：SUBSCRIBE 报文本身使用 QoS=1。这意味着订阅请求会得到 SUBACK 确认。如果 SUBACK 丢失，客户端会重发 SUBSCRIBE（DUP=1）。

SUBSCRIBE 的 Payload 中，每个 Topic Filter 对应一个 Requested QoS——Broker 根据这个值和自身的最大允许 QoS，取较小值作为 Granted QoS（在 SUBACK 中返回）。

### PUBLISH 的 Packet Identifier（§1.3.4）

2 字节无符号整数（1-65535），用于 QoS 1 和 QoS 2 的消息确认匹配。QoS=0 没有 Packet Identifier。

Packet Identifier 由发送者分配。同一个会话中，PUBLISH、PUBACK、PUBREC、PUBREL、PUBCOMP 共享同一个 ID 空间——PacketId=123 的 PUBLISH 和 PacketId=123 的 SUBSCRIBE 不会冲突。

### PUBACK/PUBREC/PUBREL/PUBCOMP（§1.3.5）

QoS 2 的确认链条：

```
发送者→Broker:   PUBLISH(PacketId=123, QoS=2)
发送者←Broker:   PUBREC(123)      ← "我收到了，但我还没有转发给订阅者"
发送者→Broker:   PUBREL(123)      ← "你现在可以安全转发了"
发送者←Broker:   PUBCOMP(123)     ← "我转发完了，这条消息在我这里清理掉了"
```

PUBREC 和 PUBREL 为什么必要：如果 Broker 收到 PUBLISH 后直接转发然后发 PUBCOMP，而此时网络断开——发送者不知道消息到底转发成功没有。PUBREC 确认"消息已持久化"，PUBREL 是"发送者确认知道 Broker 已持久化，请转发"，PUBCOMP 是"转发完毕"。

### Keep Alive 机制（§1.3.5）

Keep Alive 是客户端在 CONNECT 时声明的最大静默间隔。如果 Keep Alive 时间内没有任何 MQTT 报文交互，客户端必须发 PINGREQ，Broker 回 PINGRESP。

Broker 端的超时通常设置为 1.5 × Keep Alive——给网络延迟留缓冲。如果 1.5 × Keep Alive 内没收到任何报文，Broker 断开 TCP 连接并发布遗嘱消息（如果设置了遗嘱）。

### 遗嘱消息（Will Message）（§1.4）

CONNECT 时可选的附加信息，由三部分组成：
- Will Topic：遗嘱发布的主题
- Will Payload：遗嘱消息体
- Will QoS + Will Retain：遗嘱消息的 QoS 和保留标志

触发条件：Broker 检测到网络 I/O 错误或协议错误导致必须断开连接，且客户端没有正常发送 DISCONNECT。如果客户端正常发送 DISCONNECT → 断开，遗嘱不触发。

典型用法：设备在 CONNECT 时设置遗嘱 → topic = `/device/status`，payload = `"offline"`。当设备异常掉线，Broker 自动发布 `{"device":"xxx","status":"offline"}` → 云端立刻知道设备离线。

### 保留消息（Retained Message）（§1.5）

每条 PUBLISH 可以有 `retain` 标志。Broker 存储对应 topic 的最后一条 retain=1 的消息。新订阅者订阅该 topic 时，Broker 立即发送这条保留消息。

注意：
- 只有最后一条 retain=1 的消息被存储。新的 retain=1 消息覆盖旧的。
- 发送一条 Payload 为空的 retain=1 消息 → Broker 删除该 topic 的保留消息。
- retain 是 topic 级别的，不是 message 级别的。

---

## 第二篇：MQTT 在 Linux 网络栈

### BSD Socket API（§2.1）

POSIX 标准的网络编程接口。包含以下核心系统调用：
- `socket(AF_INET, SOCK_STREAM, 0)` → 创建 TCP socket（SOCK_STREAM = 面向连接）
- `connect(fd, &addr, len)` → 向远端发起 TCP 连接
- `send(fd, buf, len, flags)` → 发送数据到 socket
- `recv(fd, buf, len, flags)` → 从 socket 接收数据
- `close(fd)` → 关闭 socket

mosquitto 底层使用这些 API。详见 [[references/Socket编程基础]]。

### OpenSSL（§2.1）

开源的安全通信库，实现了 SSL/TLS 协议。由两个库组成：
- `libssl.so`：TLS 协议实现（握手、记录层）
- `libcrypto.so`：加密算法实现（AES、RSA、SHA256、随机数生成等）

mosquitto 使用 OpenSSL 进行 TLS 加密。链接时需要 `-lssl -lcrypto`。

### select()（§2.3）

I/O 多路复用系统调用。允许一个线程同时监控多个文件描述符（socket fd）是否可读/可写/出错。

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

`fd_set` 是一个位掩码——每个 bit 代表一个 fd 是否在被监控的集合中。`FD_ZERO` 清空集合，`FD_SET` 将 fd 加入集合，`FD_ISSET` 检查 fd 是否"就绪"。

详见 [[Socket编程基础#select]]。

**mosquitto 内部的处理**：`mosquitto_loop()` 内部调用 `select()` 监控 socket fd 是否可读。timeout 参数让 select 在没有数据时休眠指定毫秒，而不是空转烧 CPU。

### TCP 三次握手（§2.2）

```
1. SYN:      客户端 → 服务器  "我想连你"
2. SYN+ACK:  服务器 → 客户端  "可以，我准备好了"
3. ACK:      客户端 → 服务器  "收到，开始通信"
```

`mosquitto_connect()` 内部调用 `connect()` → 触发 TCP 三次握手 → 成功返回 0。

---

## 第三篇：TLS 加密

### TLS 记录层（§3.1）

TLS 在 TCP 之上插入了一层"记录层"（TLS Record Layer）。它把应用层数据（MQTT 报文）分成片段，加密后加上 TLS 记录头，通过 TCP 传输：

```
TLS 记录头（5 字节）：
  Byte 1: 内容类型
    - 20 = ChangeCipherSpec
    - 21 = Alert（警告/错误）
    - 22 = Handshake（握手消息）
    - 23 = Application Data（加密后的应用数据 = MQTT 报文）
  Byte 2-3: TLS 版本
  Byte 4-5: 加密后的数据长度
```

### 证书链验证（§3.2）

```
root.crt (CA 根证书)
  └── 签发 → 中间 CA 证书（可选）
        └── 签发 → 服务器证书 (cloud.chinaamd.cn)
```

`tls_set("./root.crt", NULL)` 加载根证书。TLS 握手时：
1. Broker 发送自己的证书链
2. mosquitto + OpenSSL 从根证书开始验证链上的每个证书签名
3. 如果 root.crt 确实是签发链上第一个证书的 CA → 验证通过

`tls_insecure_set(true)` 跳过了**域名验证**——即证书的 CN（Common Name）或 SAN（Subject Alternative Name）是否匹配连接的主机名。如果服务器证书是签发给 `cloud.chinaamd.cn` 的，但你通过 IP `192.168.1.100` 连接，严格模式下会失败。设为 true 后跳过这个检查。

### SSL_VERIFY_PEER vs SSL_VERIFY_NONE（§3.3）

`tls_opts_set(1, ...)` 第一个参数 = 1 = SSL_VERIFY_PEER：
- SSL_VERIFY_PEER（1）：验证服务器证书，验证失败立即中止连接
- SSL_VERIFY_NONE（0）：不验证证书（相当于明文传输，只是包了一层 TLS 格式）

### 证书文件的 PEM 格式（§3.3）

PEM = Privacy Enhanced Mail。一种 Base64 编码的 ASCII 格式：

```
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKlBFzqG......
-----END CERTIFICATE-----
```

`root.crt` 就是这种格式。可以用 `openssl x509 -in root.crt -text` 查看证书内容。

---

## 第四篇：mosquitto 库

### 动态链接库 .so（§4.1）

.so = Shared Object（Linux 共享库）。对应 Windows 的 .dll。

`libmosquitto.so.1` 的命名约定：
- `lib` 前缀（linker 自动添加）
- `mosquitto` 库名
- `.so` 共享库后缀
- `.1` 主版本号（ABI 兼容性——.1 版本的所有小版本互相兼容）

编译链接：`g++ ... -lmosquitto` → linker 搜索 `libmosquitto.so` → 它是指向 `libmosquitto.so.1` 的符号链接。

### 虚函数 + override（§4.2）

C++ 的多态机制。基类（mosquittopp）声明 `virtual void on_message(...)`，子类（mqttsrv）重写它。当 mosquitto C 库在 `loop()` 中收到 PUBLISH 报文时，通过回调函数指针调用到子类的 `on_message()`——这是运行时的动态绑定。

`override` 是 C++11 关键字（本工程编译为 C++98，可能被条件编译排除）。它告诉编译器"我正在重写一个基类虚函数"——如果基类没有这个虚函数，编译器会报错，防止拼写错误。

### 不透明指针（opaque pointer）（§4.2）

`struct mosquitto *m_mosq` 是一个不透明指针——mosquittopp 持有这个指针，但不知道 `struct mosquitto` 内部有什么成员。所有操作通过 C 函数完成（`mosquitto_connect(m_mosq, ...)` 等）。

这是经典的**封装**手法——隐藏实现细节。C 库可以改变内部结构而不破坏 ABI 兼容性。

### 模板方法模式（§4.2）

Gang of Four 设计模式之一。基类定义算法的骨架（connect → loop → on_message），把步骤中的某些环节延迟到子类实现（虚函数）。mosquittopp 定义的是 MQTT 通信循环的骨架，mqttsrv 实现了 on_message 这个环节。

---

## 第五篇：本工程 MQTT 架构

### QList<string>（§5.2）

Qt 的泛型容器。`QList<string>` 是 `std::string` 的动态数组。内部使用隐式共享（Copy-on-Write）——拷贝 QList 不会立即复制数据，只在第一次修改时才复制。

`recvMsgList` 在生产者和消费者之间传递 JSON 字符串。每次 `on_message` 回调 append 一条，`doParseMessage` 取出队头处理后 removeAt(0)。

### volatile（§5.2）

C/C++ 关键字。告诉编译器"这个变量可能被其他线程或硬件异步修改，不要对它做优化"。本工程的 `isrunning` 等标志位加 `volatile` 防止编译器把 `while(isrunning)` 优化成死循环。

注意：volatile 不保证原子性，不替代互斥锁。两个线程同时 `flag = flag + 1` 仍然有竞态条件。

### OAuth2（§5.2）

开放授权协议。本工程使用 **Client Credentials Grant** 模式（设备认证）：

```
设备 → POST /auth/mobile/token/device
         Authorization: Basic base64(client_id:client_secret)
         Body: mobile=DEVICE@ZKGDDEV47

设备 ← {
         "access_token": "eyJhbGciOi...",
         "token_type": "bearer",
         "expires_in": 604800   // 7 天
       }
```

参见 [[references/HTTP请求头配置汇总#Authorization]]。

---

## 第六篇：远程命令

### QRegExp（§6.6）

Qt 4 的正则表达式类（Qt 5 中改为 QRegularExpression）。

```cpp
QStringList vecList = strVec.split(QRegExp("::"));
// "a::b::c" → ["a", "b", "c"]
```

用 `"::"` 而非 `":"` 是因为参数值本身可能包含冒号（如时间戳 `2021-03-19 14:30:00`），双冒号减少误分割的概率。

### QVariant + Q_DECLARE_METATYPE（§6.6）

Qt 的类型擦除机制。`QVariant` 可以存储任意类型的数据。要让它支持自定义类型（如 `UpdateUnit`），需要：
1. `Q_DECLARE_METATYPE(UpdateUnit)` → 声明元类型
2. `QVariant::fromValue(unit)` → 包装
3. 接收端 `variant.value<UpdateUnit>()` → 解包

用于跨线程信号传参——信号参数必须是 Qt 元类型系统能识别的类型。

### MD5（§6.6）

消息摘要算法（Message Digest 5）。任意长度输入 → 128 位（16 字节）定长输出，通常用 32 字符十六进制字符串表示。

本工程中的用途：校验下载的固件文件完整性。云端下发 URL + 预期 MD5 值 → 设备下载后计算本地文件 MD5 → 对比 → 一致则文件完整。

---

## 第七篇：数据上传

### HTTP POST（§7.2-7.3）

HTTP 方法之一。向服务器提交数据。本工程的上传函数全部使用 POST，Content-Type 为 `application/json`。

### 本地缓存 + 补发（§7.3）

离线容错机制：
1. 网络不通/上传失败 → `cacheLocalRemoteInfo(url, json)` → 写入本地缓存文件
2. 程序下次启动 → `uploadLocalRemoteInfo()` → 读取缓存 → 逐个 HTTP POST 重发 → 成功则从缓存删除

这是"尽最大努力送达"的保障——不保证实时性，但保证最终一致性。

---

## 第八篇：嵌入式部署

### NTP（§8.3）

Network Time Protocol（网络时间协议）。用于从互联网时间服务器同步系统时钟。本工程使用 `ntpdate` 命令（一次性同步，非守护进程）：

```
bash
ntpdate 0.pool.ntp.org  # 从 NTP 服务器获取精确时间
hwclock -w               # 将系统时间写入硬件 RTC 芯片
```

TLS 证书有有效期——系统时间错误可能导致证书被判定为"未到生效期"或"已过期"，导致 TLS 连接失败。

### TCP 出站连接与 NAT（§8.3）

NAT = Network Address Translation（网络地址转换）。工厂内网设备通常只有内网 IP（如 192.168.1.x），通过路由器 NAT 上网。

MQTT 的优势：设备主动发起出站 TCP 连接到 Broker，NAT 路由器自动记录映射并转发回包。**不需要公网 IP、不需要端口映射、不需要 UPnP**。

这与 HTTP 服务器方案（需要配置端口映射让外网能访问设备的 80 端口）形成对比——MQTT 天然适合 NAT 后的嵌入式设备。

### mosquitto_pub / mosquitto_sub（§8.5）

mosquitto 命令行工具：
- `mosquitto_pub`：发布消息到指定 topic
- `mosquitto_sub`：订阅指定 topic 并打印收到的消息
- `-h host -p port`：指定 Broker 地址
- `-t topic`：指定主题
- `-m message`：消息内容
- `-v`：详细输出（打印 topic + payload）
