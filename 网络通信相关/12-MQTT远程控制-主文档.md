# MQTT 远程控制 — 完整链路

## 涉及的文件

| 文件 | 内容 |
|------|------|
| `CameraTest(aiuse)/zksort/bus/mqttsrv.h` | 三个类定义、ErrorCode 枚举、40+ 命令处理函数声明 |
| `CameraTest(aiuse)/zksort/bus/mqttsrv.cpp` | 完整实现（约 3700 行） |
| `CameraTest(aiuse)/zksort/bus/mosquitto.h` | libmosquitto 1.4.14 头文件 |
| `CameraTest(aiuse)/zksort/bus/myhttpfileclient.h` | HTTP 文件传输客户端（OAuth2 认证） |
| `CameraTest(aiuse)/zksort/global/globalflow.cpp:10057-10066` | MQTT 线程创建入口 |

---

## 1. 远程控制是什么

色选机部署在工厂，技术人员通过**手机微信小程序**远程监控和操控设备。通信不走 HTTP 轮询，而是用 MQTT——一种**发布/订阅**式的轻量级消息协议。

```
┌────────────────┐         ┌────────────────┐         ┌────────────────┐
│  微信小程序      │  MQTT   │  cloud.chinaamd │  MQTT   │  你的工控机     │
│  (技术人员手机)  │←───────→│  .cn:1884       │←───────→│  (色选机)      │
│                │         │  (MQTT Broker)  │         │                │
└────────────────┘         └────────────────┘         └────────────────┘
```

**小程序和控制设备不直接通信，而是通过中间人 Broker 转发。**

类比：微信群里发消息，你发到群里（Broker），群里的所有人（订阅了这个群的设备）都收到。不需要设备和手机直接连。

---

## 2. 三个类 + 两个线程

```
┌── mqttThread（独立线程）───────────────────────────┐
│                                                    │
│  mqttsrv *test（MQTT 客户端对象）                   │
│  ├── on_message() → 放入 recvMsgList               │
│  ├── recvMsgList（QList<string> + QMutex 保护）    │
│  └── sendMsgList（待发送队列）                      │
│                                                    │
│  run():                                            │
│    1. ping 检查网络                                │
│    2. TLS 连接 cloud.chinaamd.cn:1884              │
│    3. subscribe 订阅主题 "/机器编号"                │
│    4. while(1) test->loop() 死循环收发              │
│                                                    │
└────┬───────────────────────────────────────────────┘
     │ recvMsgList（加锁传递）
     │
┌────┴── mqttMsgParaseThread（独立线程）─────────────┐
│                                                    │
│  20+ 个 n_uploadXxxFlag 标志位                     │
│                                                    │
│  run():                                            │
│    1. OAuth2 Token 认证                            │
│    2. 初始数据上报                                  │
│    3. for(;;) 死循环:                              │
│       ├── 检查 20+ 个上传标志位 → 执行对应的上传    │
│       └── 轮询 recvMsgList → doParseMessage()      │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## 3. 启动流程

`globalflow.cpp:10057-10066`：

```cpp
myHttpFileClient = new MyHttpFileClient;           // HTTP 客户端（用于 Token 认证）
myMqttThread = new mqttThread;                      // MQTT 连接线程
myMqttMsgParaseThread = new mqttMsgParaseThread;     // MQTT 消息解析线程

if (ipForInterface == "162.254.129.100") {          // 只有 eth0 工作才启动
    myMqttThread->start();
    myMqttMsgParaseThread->start();
}
```

**只 Linux 编译**：所有 MQTT 相关代码都包在 `#ifdef Q_OS_UNIX` 里，Windows 编译时跳过。

---

## 4. 连接流程（mqttThread::run）

`mqttsrv.cpp:54-175`

### Step 1：网络检查

```cpp
for (int i = 0; i < 15; i++) {
    执行 "ping -w 5 ntp.ntsc.ac.cn"
    if (输出中有 "ttl") → 网络通，继续
    else → sleep(20) 重试
}
// 15 次 × 20 秒 = 最多等 5 分钟
```

### Step 2：TLS 加密配置

```cpp
mosqpp::lib_init();                    // 初始化 mosquitto 库
test->tls_opts_set(1, "tlsv1", NULL); // 用 TLSv1 加密
test->tls_insecure_set(true);         // 跳过证书域名验证
test->tls_set("./root.crt", NULL);    // 加载 CA 根证书
```

### Step 3：连接 Broker

```cpp
rc = test->connect("cloud.chinaamd.cn", 1884, 50);
//                 ↑ Broker 地址          ↑ 端口 ↑ keepalive 秒数
if (rc != 成功) {
    while (1) {      // 死等重连
        test->connect(...);
        sleep(1);
    }
}
```

### Step 4：订阅主题

```cpp
topic = "/" + myHttpFileClient->g_connectName;   // 如 "/ZKGDDEV47"
rc = test->subscribe(NULL, topic);                // 订阅后，发到这个主题的消息都能收到
if (rc != 0) 重试最多 3 次;
```

### Step 5：同步时间 + 进入死循环

```cpp
system("ntpdate 0.pool.ntp.org");   // 网络授时同步系统时间
system("hwclock -w");                // 写入硬件 RTC

while (1) {
    rc = test->loop();               // ★ 核心：死循环调 mosquitto 的 loop()
    if (rc != 0) {                   // loop 出错 = 连接断开
        sleep(5);
        test->reconnect();            // 自动重连
        test->subscribe(NULL, topic); // 重新订阅
    }
    msleep(100);
}
```

`test->loop()` 是 mosquitto 的**核心函数**——处理接收到的消息、发送待发的消息、维持心跳 keepalive。必须反复调用，每条消息正是在 `loop()` 内部触发 `on_message()` 回调。

---

## 5. 接收消息（mcqttsrv::on_message）

`mqttsrv.cpp:19-33`

```cpp
void mqttsrv::on_message(const struct mosquitto_message *message)
{
    // 1. 检查主题是否匹配
    string topic = "/" + g_connectName;
    bool res = false;
    mosqpp::topic_matches_sub(topic, message->topic, &res);

    if (res) {
        // 2. 取出消息体（JSON 字符串）
        strRcv = (char*)message->payload;

        // 3. 加锁放入接收队列
        QMutexLocker lock(&m_paraseLock);
        recvMsgList.append(strRcv);
    }
}
```

**这里只是接收并缓存，不做任何业务处理。** 队列是线程安全的——mqttThread 往里放，mqttMsgParaseThread 往外取，靠 `QMutex` 保护。

---

## 6. 解析消息（doParseMessage）

`mqttsrv.cpp:347-356` — 从队列取消息：

```cpp
while (!myMqttThread->test->recvMsgList.isEmpty()) {
    msg = myMqttThread->test->recvMsgList[0];  // 取队头
    doParseMessage();                             // 解析 + 执行
    QMutexLocker lock(&...m_paraseLock);
    myMqttThread->test->recvMsgList.removeAt(0); // 删除已处理的
}
```

### 消息格式（JSON）

```json
[
  {
    "cmdCode": "CMD_OpenSwitch",
    "content": "1:0:1:...",
    "devCode": "ZKGDDEV47",
    "uuid": "0a2de935-b381-..."
  },
  {
    "cmdCode": "CMD_Feed",
    "content": "50:20:30:...",
    "uuid": "ea53e47e-c3b5-..."
  }
]
```

**一次可能携带多条命令**（一个 JSON 数组），每条有自己的 cmdCode 和 uuid。

### 解析步骤

`mqttsrv.cpp:362-461`：

```
1. 用 jsoncpp 解析 JSON → 取出 "cmd" 数组
2. 遍历数组，对每条命令：
   ├── 提取 cmdCode、content、uuid
   │
   ├── 特殊命令先处理：
   │   ├── CMD_LockStatusUpload → 检查锁屏状态
   │   └── CMD_ConnectVPN → 连接/断开 VPN
   │
   ├── ★ 先回 ACK："指令已收到" → onCmdReturn(uuid, cmd, RECEIVE)
   │
   ├── 状态检查（不满足则拒绝执行）：
   │   ├── 正在清灰？     → 回 CLEAN_ING
   │   ├── 正在采图？     → 回 NOT_MAIN_WND
   │   ├── 首次运行未完成？→ 回 SYSTEM_ERROR
   │   └── 当前在特殊页面？→ 回 NOT_MAIN_WND
   │       （方案管理页、背景设置页、图像采集页...这 8 种页面不允许远程操控）
   │
   └── 按 cmdCode 分发：
       ├── CMD_OpenSwitch    → onOpenSwitch(value)
       ├── CMD_OpenSwitchs   → onOpenSwitchs(value)
       ├── CMD_Feed          → onCmdFeed(value)
       ├── CMD_MemSet        → onMemSet(value, uuid)
       ├── CMD_ShutDown      → onShutDown()
       ├── CMD_CaptureImage  → onCaptureImage(value, tmp)
       ├── CMD_UpdateZKSort  → onUpdateZKSort(value)
       ├── CMD_UpdateIpcModel→ onUpdateZKSort(value)
       ├── CMD_RemoteControl → onRemotecontrol(cmd, time, phonenum)
       ├── CMD_Shell_Cmd     → onShellCmd(value)（执行任意 shell 命令！）
       ├── CMD_SaveCfg       → onSaveCfg()
       ├── CMD_GlobalParaSet  → onGlobalParaSet(value)
       ├── CMD_ParaUpload    → onParaUpload(value)（触发数据上报）
       ├── CMD_BlowCountsUpload → onBlowCountsUpload()
       ├── CMD_LogFileUpload → onLogfileUpload(value)
       ├── CMD_VavleAudioUpload → onVavleAudioUpload()
       └── ...
```

---

## 7. 回传结果（onCmdReturn）

`mqttsrv.cpp:1416-1441` — 每个远程命令都通过这个函数把执行结果通过 MQTT 发布给小程序：

```cpp
void mqttMsgParaseThread::onCmdReturn(string uuid, string cmd, int errorcode)
{
    // 组装 JSON 回包
    Json::Value root;
    root["uuid"]     = uuid;           // 与请求相同的 uuid（小程序用于匹配）
    root["devId"]    = g_connectName;  // 设备编号
    root["cmdCode"]  = cmd;            // 命令字
    root["result"]   = errorcode;      // 结果码（0=成功, 负数=失败原因）
    root["type"]     = "push";         // 推送类型

    // publish 到 "MqttServerClient" → 云端服务器监听这个主题
    myMqttThread->test->publish(NULL, "MqttServerClient", ...);
}
```

**publish 的目标主题是 `"MqttServerClient"`**，不是发回给自己的订阅主题。云端服务器订阅了这个主题，收到回包后转发给微信小程序。

---

## 8. 数据上传（标志位机制）

`mqttMsgParaseThread::run()` 的主循环中，有 20+ 个 `n_uploadXxxFlag` 标志位。当某个标志位被设为 1 时，主循环在下一次轮询中自动执行对应的上传函数：

```
主循环每秒一次：
  ├── Token 过期？ → 续期
  ├── n_uploadPartsStatusFlag    → uploadIpcPartsStatusInfo()
  ├── n_uploadAlarmFlag          → uploadParaAlarm()
  ├── n_uploadRealTimeParaFlag   → uploadRealTimePara()
  ├── n_uploadStatisticsFlag     → uploadStatistics()
  ├── n_uploadBlowCountsFlag     → uploadBlowCounts()
  ├── n_uploadLogFileFlag        → uploadLogFile()
  ├── n_uploadRemoteControlFlag  → uploadRemoteControl()
  └── ...
```

任何代码需要上报数据时，只需将对应的标志位设为 1，主循环会在下个周期自动处理。不需要自己调网络请求——**异步上传，不阻塞 UI。**

---

## 9. 错误码一览

| 码 | 宏 | 含义 |
|-----|------|------|
| 0 | `SUCCESS` | 成功 |
| -1 | `SYSTEM_ERROR` | 系统错误 |
| 2 | `UNKNOWN_CMD` | 未知命令 |
| 3 | `CMD_PARA_ERROR` | 参数错误 |
| 6 | `MEM_SETING` | 正在切换方案中 |
| 8 | `FEED_ING` | 正在供料中，无法执行 |
| 9 | `NOT_MAIN_WND` | 不在主界面，无法远程操控 |
| 10 | `CLEAN_ING` | 正在清灰中 |
| 13 | `RECEIVE` | 指令已收到（不是最终结果，是 ACK） |
| 16 | `USERREJECT` | 用户拒绝 |
| 22 | `NETWORK_ERR` | 网络异常 |
| 35 | `LOCKED_ERROR` | 未锁屏（远程控制前需要先锁屏） |
| 36 | `OCCUPY_ERROR` | 已被其他人远程控制 |
| 37 | `REMOTE_CANCEL` | 远程取消 |

---

## 10. 完整时序（以"远程开关物料通道"为例）

```
微信小程序                   云端                   mqttThread          mqttMsgParaseThread       UI/硬件
    │                         │                        │                      │                    │
    │──MQTT Publish──────────→│                        │                      │                    │
    │  {"cmdCode":            │                        │                      │                    │
    │   "CMD_OpenSwitch",     │──转发────────────────→│                      │                    │
    │   "content":"1:0:1..."} │                        │ on_message()        │                    │
    │                         │                        │ 放入 recvMsgList    │                    │
    │                         │                        │                      │                    │
    │                         │                        │            轮询取出消息                    │
    │                         │                        │             doParseMessage()              │
    │                         │                        │              解析 cmdCode/content/uuid    │
    │                         │                        │                      │                    │
    │                         │                        │        ★ 先回 RECEIVE ACK                │
    │                         │←──MQTT publish─────────│                      │                    │
    │                         │   "result":"13"        │                      │                    │
    │←──"指令已收到"──────────│                        │                      │                    │
    │                         │                        │                      │                    │
    │                         │                        │             onOpenSwitch(value)           │
    │                         │                        │              解析 content 字符串          │
    │                         │                        │              → mySerial 发送串口命令      │
    │                         │                        │                      │                    │
    │                         │                        │                      │     → FPGA 收到    │
    │                         │                        │                      │     执行开关通道   │
    │                         │                        │                      │                    │
    │                         │                        │             uploadPara()                  │
    │                         │                        │              上传最新参数到云端           │
    │                         │                        │                      │                    │
    │                         │                        │             onCmdReturn(uuid, cmd, SUCCESS)│
    │                         │←──MQTT publish─────────│                      │                    │
    │                         │   "result":"0"         │                      │                    │
    │←──"开关通道成功"────────│                        │                      │                    │
```

---

## 11. 线程跨越点

| 环节 | 所在线程 | 说明 |
|------|---------|------|
| `test->loop()` 收发 | mqttThread | mosquitto 内部处理 |
| `on_message()` 回调 | mqttThread | mosquitto 触发 |
| 放入 recvMsgList | mqttThread | 加锁写入 |
| 轮询 recvMsgList | mqttMsgParaseThread | 加锁读取 |
| `doParseMessage()` | mqttMsgParaseThread | JSON 解析 + 命令分发 |
| 业务函数执行 | mqttMsgParaseThread | 串口命令、文件操作、HTTP 上传 |
| `emit StartRemoteControl` | mqttMsgParaseThread | 信号跨线程 → 主线程 |
| UI 弹出锁屏页 | 主线程 | 信号触发的槽函数 |
| `onCmdReturn()` publish | mqttMsgParaseThread | MQTT 发布回包 |
| 数据上传 | mqttMsgParaseThread | HTTPS 上传到云端 HTTP 接口 |
