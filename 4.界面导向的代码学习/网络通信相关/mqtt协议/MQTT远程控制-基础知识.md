# MQTT 远程控制 — 基础知识清单

按在 [[MQTT远程控制-主文档|主文档]] 中首次出现的顺序排列。

---

## 第 1 章出现的

### MQTT（§1）

MQTT = Message Queuing Telemetry Transport（消息队列遥测传输）。一种**极轻量级的发布/订阅消息协议**，1999 年由 IBM 为石油管道传感器设计。

核心设计理念：**设备不需要互相知道对方的存在，只需要知道"中间人"Broker 在哪里。** 发布者把消息发到 Broker，Broker 转发给所有订阅了该主题的设备。

与 UDP/TCP 的关系：MQTT 跑在 **TCP 之上**（所以保证可靠送达），UDP/TCP 是传输层，MQTT 是应用层。就像 HTTP 跑在 TCP 上一样，MQTT 也是一层应用协议。

| 对比 | MQTT | HTTP |
|------|------|------|
| 方向 | 双向（设备可主动上报） | 请求-响应（客户端发起） |
| 开销 | 最小固定头 2 字节 | 头动辄几百字节 |
| 连接 | 长连接，保持心跳 | 短连接，用完就断 |
| 适用 | IoT、嵌入式、低带宽 | Web、API |

### 发布/订阅模型（Publish/Subscribe）（§1）

```
传统 C/S 模型：              发布/订阅模型：
  设备A → 直接连 → 设备B      设备A → Broker ← 设备B
                                       ↑
                                    设备C
                                    
  必须知道对方的 IP 和端口      只需要知道 Broker 的地址
  一对一                        一对多、多对多都支持
  双方必须同时在线               可以离线——Broker 暂存消息
```

**类比**：微信公众号。你订阅了一个公众号（Topic），公众号发文章（Publish），你收到推送。公众号不知道你的手机号，你也不知道公众号的服务器 IP——微信（Broker）在中间转发。

### Broker（§1）

MQTT 的消息中转站。本工程的 Broker 是 `cloud.chinaamd.cn:1884`。

Broker 做的事：接收所有发布者的消息 → 查找谁订阅了这个主题 → 转发给所有订阅者。自己不产生消息，只做转发。

### 主题（Topic）（§1）

MQTT 的寻址方式，用斜杠分隔的字符串，类似文件路径：

```
/ZKGDDEV47/status/temperature
/ZKGDDEV47/cmd
/MqttServerClient
```

订阅者告诉 Broker"我对某个主题感兴趣"，之后所有发布到该主题的消息自动转发过来。本工程订阅的主题是 `"/" + 机器编号`，如 `"/ZKGDDEV47"`。

支持通配符：`+` 匹配单层，`#` 匹配多层。本工程没有用通配符。

### 微信小程序（§1）

微信小程序是一个运行在微信内的轻量应用，不需要安装。色选机厂商开发了一个小程序，让技术人员可以通过手机远程查看设备状态、调节参数、触发操作。

小程序通过云端的 MQTT 客户端连接 Broker，发布命令到设备的主题，订阅设备的状态回包主题。

---

## 第 2 章出现的

### mqttsrv（§2）

`mqttsrv` 继承自 `mosqpp::mosquittopp`——这是 libmosquitto 的 C++ 封装。它实现了所有 MQTT 协议细节（连接、订阅、发布、心跳），上层只需要实现回调函数。

```cpp
class mqttsrv : public mosqpp::mosquittopp
{
    void on_connect(int rc);                              // 连接成功/失败回调
    void on_disconnect();                                  // 断开回调
    void on_message(const struct mosquitto_message *msg);  // 收到消息回调
    void on_subscribe(int mid, int qos, const int *granted);// 订阅成功回调

    QList<string> recvMsgList;   // 接收队列
    QMutex m_paraseLock;         // 保护队列的锁
};
```

### mqttThread（§2）

QThread 子类，负责**网络连接层**的工作——TLS 握手、连接 Broker、死循环调 `loop()` 维持 MQTT 连接。

### mqttMsgParaseThread（§2）

QThread 子类，负责**业务逻辑层**——解析 JSON、分发命令、执行操作、标志位驱动的数据上传。

### recvMsgList（§2）

`QList<string>` 类型，被 `QMutex` 保护。mqttThread 往里写（`on_message` 回调），mqttMsgParaseThread 往外读（轮询取出）。生产者-消费者模型。

### QMutex（§2）

见 [[Qt核心类型#QMutex + QMutexLocker]]。

---

## 第 3 章出现的

### #ifdef Q_OS_UNIX（§3）

见 `08-系统状态界面-基础知识`。MQTT 依赖的 libmosquitto 是 Linux 库，Windows 上没有，所以整块编译跳过。

### ipForInterface（§3）

`myFlow` 的成员变量。eth0 的 IP 地址。只有 eth0 网卡正确配置了 IP 时才启动 MQTT——确保网络是通的。

---

## 第 4 章出现的

### ntp.ntsc.ac.cn（§4-Step1）

中国科学院国家授时中心 NTP 服务器。ping 它测试互联网连通性——能 ping 通说明设备能上网，MQTT 才有意义。

### TLS/SSL（§4-Step2）

TLS = Transport Layer Security（传输层安全）。在 TCP 之上加一层加密，第三方即使截获了数据包也无法解密。

```cpp
test->tls_opts_set(1, "tlsv1", NULL);  // 使用 TLSv1 版本
test->tls_insecure_set(true);          // 不验证服务器域名
test->tls_set("./root.crt", NULL);     // 加载 CA 根证书
```

**验证流程**：
```
1. 客户端加载 root.crt（CA 根证书）
2. 连接 Broker 时，Broker 发来自己的证书
3. 客户端用 CA 根证书验证 Broker 的证书是否合法
4. 验证通过 → 建立加密通道 → 所有后续通信都加密
```

`tls_insecure_set(true)` 跳过了域名验证——允许 IP 地址直连的证书。工业设备通常没有 DNS 域名，只能用 IP 连接。

### mosquitto（libmosquitto）（§4-Step2）

开源的 MQTT 协议库，用 C 语言编写。提供了 `mosquittopp`（C++ wrapper），本工程用的是 v1.4.14 版本。

### lib_init()（§4-Step2）

mosquitto 库的全局初始化函数。分配内部数据结构、初始化网络。程序生命周期内调一次。

### keepalive（§4-Step3）

MQTT 连接的心跳间隔，单位秒。本工程设为 50 秒——Broker 如果 50 秒内没收到任何消息，会认为设备掉线。`test->loop()` 内部会自动在 keepalive 时间到期前发 PINGREQ。

---

## 第 5 章出现的

### mosquitto_message（§5）

mosquitto 库定义的结构体：

```cpp
struct mosquitto_message {
    int mid;                   // 消息 ID
    char *topic;               // 主题名称
    void *payload;             // 消息内容（二进制）
    int payloadlen;            // 消息内容长度
    int qos;                   // QoS 等级
    bool retain;               // 是否保留消息
};
```

本工程只用了 `topic`（检查是否匹配）和 `payload`（取出 JSON 字符串）。

### QoS（§5）

Quality of Service，MQTT 的可靠性等级：

| QoS | 含义 | 本工程 |
|-----|------|--------|
| 0 | 最多一次（发了不管） | 不用 |
| 1 | 至少一次（确认收到） | 不用 |
| 2 | 恰好一次（四步握手） | **publish 用 QoS 2** |

`test->publish(NULL, topic, len, data, 2)` 最后一个参数是 QoS=2——确保回包一定被云端收到。

### QMutexLocker（§5）

见 [[Qt核心类型#QMutex + QMutexLocker]]。

---

## 第 6 章出现的

### JSON（§6）

JSON = JavaScript Object Notation。一种数据交换格式，用花括号、方括号、逗号、冒号表示层级结构。

```json
{
  "cmdCode": "CMD_OpenSwitch",
  "content": "1:0:1:1:0",
  "uuid": "0a2de935-b381-fd09-f68b-b071ba65c275"
}
```

本工程用 `Json::Reader` 解析 JSON → `Json::Value` 树形结构 → 用 `[]` 访问字段。

### jsoncpp（§6）

本工程使用的 C++ JSON 库（`3rdparty/jsoncpp/`）。提供了 `Json::Reader`（解析）、`Json::Value`（树形数据结构）、`Json::FastWriter`（序列化）。

### cmdCode（§6）

命令码。字符串形式，如 `"CMD_OpenSwitch"`、`"CMD_Feed"`。与 UDP 通信中 16 位的 `CMD_UDP_IPC_REQ_INFO` 作用相同——标识"这是一个什么操作"。MQTT 用的是字符串，UDP 用的是数字，因为 MQTT 的数据量不敏感（走 TCP + 云端），用字符串更方便调试。

### uuid（§6）

Universally Unique Identifier。每个请求的唯一 ID，32 位十六进制字符串加连字符。小程序生成 uuid 塞进请求，设备回包时带上同样的 uuid，小程序用 uuid 把"请求"和"回包"匹配起来——"我刚发的那条命令，设备回我结果了，就是这个"。

与 UDP 通信中 `seq_h/seq_l`（序列号）的作用相同——请求-应答匹配。

### RECEIVE（§6）

错误码 13，表示"指令已收到，尚未执行"。不是最终结果，是一个 ACK——"我收到你的命令了，马上处理"。

### 状态检查（§6）

doParseMessage 在执行业务函数之前，先检查 6 种不安全的场景：

```
正在清灰？      → 喷涂高压气体，不能分心执行远程命令
正在采图？      → 占用网络带宽，不宜被打断
首次运行未完成？→ 配置还没加载完，不能接受远程操控
当前在特殊页面？→ 方案管理/背景设置/图像采集等 8 种页面不允许远程干预
```

**设计哲学**：宁可拒绝执行，也不能在危险状态下执行远程命令。

---

## 第 7 章出现的

### MqttServerClient（§7）

云端服务器在 Broker 上订阅的主题。所有设备的回包都发布到这个主题，云端集中收。与设备订阅的主题（`"/机器编号"`）分离——设备收命令走自己的主题，回包走公共主题。

### publish（§7）

mosquitto 的发布函数：

```cpp
int publish(
    int *mid,           // [出参] 消息 ID，NULL 表示不关心
    const char *topic,  // 目标主题
    int payloadlen,     // 消息长度
    const void *payload,// 消息内容
    int qos,            // 0/1/2
    bool retain         // 是否保留（新订阅者是否收到最后一条消息）
);
```

本工程 publish 回包使用 QoS=2——确保云端一定能收到。

---

## 第 8 章出现的

### 标志位机制（§8）

**生产者-消费者模型的异步模式**：

```
任何代码 → n_uploadRealTimeParaFlag = 1   （生产者，写入标志位）

mqttMsgParaseThread 主循环                  （消费者，检查标志位）
  → if (n_uploadRealTimeParaFlag) {
        uploadRealTimePara();               // 执行实际的上传操作
        n_uploadRealTimeParaFlag = 0;       // 执行完清零
    }
```

与信号/槽的区别：信号/槽是**通知**（emit 到槽），标志位是**轮询**（主循环每秒检查一次）。标志位更适合"不着急"的场景——数据上报晚几秒没问题，不需要信号的开销。

### Token（OAuth2）（§8）

`myHttpFileClient->g_token`：OAuth2 设备认证令牌。设备首次启动时向云端申请 Token，之后所有 HTTP 请求都带上 Token 证明身份。Token 有过期时间（本工程中每周续期一次），过期后需重新申请。

---

## 第 11 章出现的

### 跨线程信号（§11）

`emit StartRemoteControl(phonenum)` 在 mqttMsgParaseThread 中发射，接收槽在**主线程**。Qt 用 `AutoConnection` 自动处理跨线程调度。

### HTTPS 上传（§11）

数据上传不通过 MQTT——因为图片/日志等大文件通过 MQTT 传输效率低。数据上传走 HTTPS（HTTP + TLS）到云端的 REST API。MQTT 只用来接收命令和回传小数据包。

两者的分工：

```
MQTT：  实时命令 + 状态码回传    （小数据，低延迟）
HTTPS： 文件上传下载 + 数据上报   （大文件，可断点续传）
```
