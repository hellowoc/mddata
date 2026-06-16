# mqttsrv 文件函数总览

## mqttsrv — mosquitto 回调封装

| 函数 | 作用 |
|------|------|
| `mqttsrv(id)` | 构造函数，记录设备编号到 `m_connect_name` |
| `on_connect(rc)` | 连接成功/失败回调（空函数体，只调试打印） |
| `on_disconnect()` | 断开连接回调（空函数体） |
| `on_publish(mid)` | 发布成功回调（空函数体） |
| `on_subscribe(mid, qos, granted)` | 订阅成功回调（空函数体） |
| `on_message(message)` | 收到消息 — 校验主题匹配 → 加锁放入 `recvMsgList` |

## mqttThread — 网络连接线程

| 函数 | 作用 |
|------|------|
| `mqttThread()` | 构造：设置 `connectName`、创建 `mqttsrv` 对象 |
| `run()` | ping 外网 → TLS 连接 Broker → subscribe → `loop()` 死循环 |
| `reconnect()` | 断线重连 + 重新订阅 |

## mqttMsgParaseThread — 消息解析+命令执行线程

### 生命周期与主循环

| 函数 | 作用 |
|------|------|
| `run()` | Token 认证 → 初始上报 → `for(;;)` 死循环：Token 续期、标志位检查、轮询 `recvMsgList` |
| `doParseMessage()` | JSON 解析 → 参数提取 → 状态检查 → 按 cmdCode 分发到具体处理函数 |
| `onCmdReturn(uuid, cmd, errorcode)` | 构造 JSON 回包 → MQTT Publish 到 `"MqttServerClient"` |

### 远程命令处理函数（CMD_xxx）

| 函数 | 对应命令 | 操作 |
|------|---------|------|
| `onOpenSwitch(value)` | `CMD_OpenSwitch` | 开关物料通道 |
| `onOpenSwitchs(value)` | `CMD_OpenSwitchs` | 批量开关通道 |
| `onCmdFeed(value)` | `CMD_Feed` | 调节供料振动量 |
| `onMemSet(value, uuid)` | `CMD_MemSet` | 切换分选方案 |
| `onSaveCfg()` | `CMD_SaveCfg` | 保存当前配置 |
| `onGlobalParaSet(value)` | `CMD_GlobalParaSet` | 设置全局参数 |
| `onModelParamsSet(value)` | `CMD_ModelParamsSet` | 设置模型参数 |
| `onShutDown()` | `CMD_ShutDown` | 系统关机 |
| `onCaptureImage(value, tmp)` | `CMD_CaptureImage` | 远程采图 |
| `onCaptureAllImage()` | `CMD_CaptureAllImage` | 远程全图采集 |
| `onShellCmd(value)` | `CMD_Shell_Cmd` | 执行任意 shell 命令 |
| `onUpdateZKSort(value)` | `CMD_UpdateZKSort` | 上位机固件升级（解析参数 → emit信号） |
| `onRemotecontrol(cmd, time, phone)` | `CMD_RemoteControl` | 远程锁屏控制 |
| `onParaUpload(value)` | `CMD_ParaUpload` | 触发数据上报（设标志位，函数体空） |
| `onVPNPushCert(value)` | `CMD_VPNPushCert` | 推送 VPN 证书 |
| `onVPNConnect(value)` | `CMD_VPNConnect` | 连接/断开 VPN |
| `onLogfileUpload(value)` | `CMD_LogfileUpload` | 上传指定日期日志 |
| `onCnffileUpload()` | `CMD_CnffileUpload` | 打包并上传全部配置文件 |
| `onWipeSpeedUpload()` | `CMD_WipeSpeedUpload` | 上传清灰速度数据 |
| `onBlowCountsUpload()` | `CMD_BlowCountsUpload` | 上传喷阀吹气次数 |
| `onVavleAudioUpload()` | `CMD_VavleAudioUpload` | 上传喷阀音频数据 |
| `onIpcInfoUpload(value)` | `CMD_IpcInfoUpload` | IPC 信息上传触发 |
| `onAllEjtTest(value)` | CMD_喷阀测试 | 全部喷阀测试 |
| `onSglEjtTest(value)` | CMD_喷阀测试 | 单个喷阀测试 |
| `onReSetEjt(value)` | CMD_喷阀测试 | 复位喷阀 |
| `onEjtTestMode(value)` | CMD_喷阀测试 | 喷阀测试模式 |

### 数据上传函数（uploadXxx）

| 函数 | 触发标志位 | 上传内容 |
|------|---------|---------|
| `uploadPara()` | 同步调用（多个命令执行成功后立即调） | 当前方案参数 |
| `uploadRealTimePara()` | `n_uploadRealTimeParaFlag` | 实时产量、坏点率 |
| `uploadMachinePara()` | 程序启动时 | 机型配置信息 |
| `uploadPartsStatusInfo()` | `n_uploadPartsStatusFlag` | 部件状态 |
| `uploadIpcPartsStatusInfo()` | `n_uploadPartsStatusFlag`（IPC 版本） | IPC 相机状态 |
| `uploadParaAlarm()` | `n_uploadAlarmFlag` | 报警信息 |
| `uploadStatistics()` | `n_uploadStatisticsFlag` | 统计数据 |
| `uploadBlowCounts()` | `n_uploadBlowCountsFlag` | 喷阀吹气次数 |
| `uploadLogFile(date)` | `n_uploadLogFileFlag` | 运行日志文件 |
| `uploadCaptureImage()` | `n_uploadImageFlag` | 采集的图像 |
| `uploadCaptureWave()` | `n_uploadWaveFlag` | 采集的波形 |
| `upLoadSchemeInfo()` | `n_uploadSchemeInfoFlag` | 方案信息 |
| `uploadRemoteControl()` | `n_uploadRemoteControlFlag` | 远程控制确认/拒绝结果 |
| `uploadRemoteAccess(val)` | 同步调用 | VPN 远程访问授权 |
| `uploadModelEvaluate()` | `n_uploadModelEvaluate` | 模型评价 |
| `uploadMemsetReturn()` | 同步调用（方案切换完成后） | 方案切换结果 |
| `uploadOneKeyFixInfo()` | `n_uploadOneKeyFixFlag` | 一键修复信息 |
| `uploadLocalRemoteInfo()` | 程序启动时 | 补传缓存的离线数据 |
| `onModelParamsUpload()` | `n_uploadModelParamsFlag` | 模型参数 |

### 辅助函数

| 函数 | 作用 |
|------|------|
| `cacheLocalRemoteInfo(url, val)` | 缓存上报数据（网络不通时暂存本地） |
| `doStateChange(tmp)` | IPC 状态变化处理 |
| `getModeList()` | 从云端获取模型标签列表 |
| `getModelistResult_sig()` | 信号：模型列表获取完成 |
| `imgaeCaptureOver(h, path)` | 采图完成槽 — 触发图像上传 |
| `waveCaptureOver()` | 波形采集完成槽 — 触发上传 |
| `blowCountsCaptureOver()` | 喷阀计数采集完成槽 — 触发上传 |
| `DealStartRemoteControl(phone)` | 弹出锁屏确认界面 |


# 总结
首先mqttthread创建线程构造对应的函数，然后run链接网络，之后一直在loop中死循环，等待消息，有消息之后就进入消息队列。另一边，mqttMsgParaseThread 的run函数认证token然后死循环中轮询消息队列，解析消息，然后给到各个函数中，回包通过mqtt来publish到对应的端口，数据的上传和下载，调用http来实现