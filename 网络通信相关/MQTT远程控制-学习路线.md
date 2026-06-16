# MQTT 远程控制 — 手机到色选机全过程学习路线

## 总览：手机如何控制色选机

```
┌──────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 手机小程序 │    │ cloud.chinaamd.cn │    │ 色选机 Linux 工控机 │
│          │    │                  │    │                  │
│ 用户点击   │    │ :1884 Broker     │    │ mqttThread       │
│ "开通道1"  │    │ :8900 HTTP       │    │ mqttMsgParaseThd  │
│          │    │                  │    │ myHttpFileClient │
└──────────┘    └──────────────────┘    └──────────────────┘
```

整个过程分五个阶段：

1. **设备注册 + 获取身份**：OAuth2 Token → MQTT 连接 → 订阅主题
2. **手机发命令**：小程序 → Broker → 工控机收到
3. **命令解析**：JSON 解析 → 分发到具体处理函数
4. **命令执行**：操作硬件（串口/GPIO/系统命令）
5. **结果回传**：状态码 → Broker → 小程序

---

## 阶段一：设备上线（准备阶段）

### 涉及的代码文件

| 文件 | 内容 |
|------|------|
| `global/globalflow.cpp:10040-10066` | myHttpFileClient、myMqttThread、myMqttMsgParaseThread 创建 |
| `bus/myhttpfileclient.cpp:12-21` | MyHttpFileClient 构造函数 — 读取 connectName |
| `bus/myhttpfileclient.cpp:177-267` | requestTokenUpdate — OAuth2 获取 Token |
| `bus/mqttsrv.cpp:42-175` | mqttThread::run() — TLS 连接 + 订阅 |
| `bus/mqttsrv.cpp:178-358` | mqttMsgParaseThread::run() — Token 续期 + 标志位轮询 |

### 步骤

**1.1 connectName — 设备编号**

程序启动 → `MyHttpFileClient` 构造函数从 `CFG_APPSet.ini` 读取 `connectName`：
- 如 `ZKGDDEV47`，这是设备在云端的唯一标识
- MQTT 主题 = `"/ZKGDDEV47"`
- Token 申请时提交 `mobile=DEVICE@ZKGDDEV47`

**1.2 OAuth2 Token 获取**

mqttMsgParaseThread 启动后第一件事——Token 认证：
- 向 `cloud.chinaamd.cn:8900/auth/mobile/token/device` 发 POST
- 请求体：`mobile=DEVICE@ZKGDDEV47`
- 认证头：`Basic YXBpOmFwaQ==`（username:password base64）
- 云端返回：`{"access_token":"xxx","expires_in":3600}`
- Token 存入 `g_token`，有效期存入 `g_token_expires`

**1.3 MQTT 连接 + 订阅**

mqttThread 启动 → ping 外网确认通 → TLS 加密连接 `cloud.chinaamd.cn:1884` → 订阅 `"/ZKGDDEV47"` → 进入 `loop()` 死循环

**1.4 初始数据上报**

mqttMsgParaseThread 在进入主循环前，先上报：
- 实时参数 → `uploadRealTimePara()`
- 机器信息 → `uploadMachinePara()`
- 日志文件 → `uploadLogFile()`

---

## 阶段二：手机下发命令（已学，需复习）

### 涉及的代码

| 代码位置 | 内容 |
|---------|------|
| `mqttsrv.cpp:19-33` | mqttsrv::on_message() — 接收 |
| `mqttsrv.cpp:347-356` | 主循环取出消息 |
| `mqttsrv.cpp:362-505` | doParseMessage() — 解析+分发 |
| `mqttsrv.cpp:1416-1441` | onCmdReturn() — 回传结果 |

### 流程

```
手机小程序点击"开通道1"
  → 小程序：构造 JSON
    {
      "cmd": [{
        "cmdCode": "CMD_OpenSwitch",
        "content": "1:1:...",      // 参数：哪个通道，开还是关
        "uuid": "..."
      }]
    }
  → 小程序：MQTT Publish 到 "MqttServerClient"（云端控制主题）
  → Broker 转发到工控机订阅的 "/ZKGDDEV47"
  → mqttThread::on_message() → recvMsgList
  → mqttMsgParaseThread 轮询取出 → doParseMessage()
    → 解析 cmdCode="CMD_OpenSwitch"
    → 先回 ACK："指令已收到，result=13"
    → 状态检查（清灰中？采图中？特殊页面？）
    → 调用 onOpenSwitch(content)
    → 执行成功 → 回传 "result=0"
```

---

## 阶段三：具体命令处理函数（未深入）

`mqttsrv.cpp` 中 20+ 个命令处理函数，每个都需要深入：

| 命令 | 处理函数 | 实际操作 |
|------|---------|---------|
| `CMD_OpenSwitch` | `onOpenSwitch(content)` | 打开/关闭指定物料通道 |
| `CMD_Feed` | `onCmdFeed(content)` | 调节供料振动量 |
| `CMD_MemSet` | `onMemSet(content, uuid)` | 切换分选方案 |
| `CMD_ShutDown` | `onShutDown()` | 系统关机（`system("shutdown -h now")`) |
| `CMD_CaptureImage` | `onCaptureImage(content)` | 远程触发相机采图 |
| `CMD_RemoteControl` | `onRemotecontrol(cmd,time,phonenum)` | 远程锁屏控制 |
| `CMD_SaveCfg` | `onSaveCfg()` | 保存当前配置 |
| `CMD_Shell_Cmd` | `onShellCmd(content)` | 执行任意 shell 命令 |
| `CMD_UpdateZKSort` | `onUpdateZKSort(content)` | 上位机 OTA 升级 |
| `CMD_UpdateIpcModel` | `onUpdateZKSort(content)` | IPC AI 模型升级 |

---

## 阶段四：标志位驱动的数据上传（未深入）

mqttMsgParaseThread 主循环每秒轮询 15+ 个标志位：

| 标志位 | 触发者 | 执行的上传函数 |
|--------|-------|-------------|
| `n_uploadPartsStatusFlag` | IPC 测试结束后 | `uploadIpcPartsStatusInfo()` |
| `n_uploadAlarmFlag` | `updateStatusThread` — 报警状态变化 | `uploadParaAlarm()` |
| `n_uploadRealTimeParaFlag` | 定时任务 — 每秒 | `uploadRealTimePara()` |
| `n_uploadLogFileFlag` | `TopWidget::updateDateTime()` — 23:59:30 | `uploadLogFile()` |
| `n_uploadRemoteControlFlag` | `RemoteControlWidget` — 用户确认/拒绝 | `uploadRemoteControl()` |
| `n_uploadBlowCountsFlag` | 喷阀测试结束后 | `uploadBlowCounts()` |

---

## 阶段五：Token 续期 + 断线重连（已学，需复习）

- Token 过期检测：`g_token_expires < struGsh.nCounter - m_nextWeek`
- Token 续期：调 `requestTokenUpdate()`
- MQTT 断线：`loop()` 返回非 0 → `reconnect()` → 重新 `subscribe()`

---

## 学习顺序建议

```
第1步（已学）：阶段二 — 手机发命令到设备收到的完整链路
    文件：mqttsrv::on_message → doParseMessage → onCmdReturn

第2步：阶段一中的 Token 认证链路
    文件：myhttpfileclient::requestTokenUpdate → g_token

第3步：阶段一中的 MQTT 连接细节
    文件：mqttThread::run() — TLS + connect + subscribe + loop

第4步：阶段四 — 标志位驱动的上传机制
    文件：mqttMsgParaseThread::run() 主循环 + 各上传函数

第5步：阶段三 — 具体命令处理函数逐个拆解
    文件：mqttsrv.cpp 后半部分（onXxx 函数）

第6步：阶段五 — Token 续期 + 断线重连
    文件：mqttMsgParaseThread::run() + mqttThread::run()
```
