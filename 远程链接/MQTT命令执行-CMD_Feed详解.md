# MQTT 命令执行 — CMD_Feed 详解

## 命令用途

调节供料振动器的振幅——每个振动器对应一个供料通道，振幅越大物料流速越快。手机端通过冒号分隔的字符串指定每个振动器的供料值。

## 完整数据流

```
手机小程序
  │ MQTT Publish
  │ {"cmd":[{"cmdCode":"CMD_Feed","content":"50:20:30:...","uuid":"xxx"}]}
  ↓
cloud.chinaamd.cn:1884 (Broker)
  │ 转发到订阅主题
  ↓
mqttThread::loop() → mqttsrv::on_message()
  → recvMsgList.append(json)
    ↓
mqttMsgParaseThread 主循环轮询
  → doParseMessage()
    ├── 解析 cmdCode = "CMD_Feed"
    ├── 先回 ACK → onCmdReturn(uuid, "CMD_Feed", RECEIVE)
    ├── 状态检查（清灰？采图？特殊页面？）
    ├── onCmdFeed("50:20:30:...")
    │   ├── split(":") → ["50","20",...]
    │   ├── 校验长度 == nChuteTotal
    │   ├── 写入 struCnfp.nFeederValueMain[]
    │   └── myFlow.resetFeederRG()
    │       └── mySerial.pushCom1CmdData(...)
    │           └── 串口线程 → 串口线 → PLC/控制板
    │               └── DAC/PWM 改变 → 振动器振幅改变
    │                   └── 供料量变化 → 物料流速改变
    │
    ├── uploadPara() → 最新参数上报云端
    └── onCmdReturn(uuid, "CMD_Feed", SUCCESS)
        → MQTT Publish → Broker → 手机收到 "执行成功"
```

## doParseMessage 中的分发

```cpp
else if (tmp == string("CMD_Feed"))
{
    int errorcode = onCmdFeed(value);
    if (errorcode == SUCCESS){
        uploadPara();                          // 执行成功 → 立即上报最新参数
    }
    onCmdReturn(uuid, tmp, errorcode);         // 回传结果给手机
    return;
}
```

## onCmdFeed — 解析参数 + 写入配置 + 应用到硬件

```cpp
int mqttMsgParaseThread::onCmdFeed(string cmd_value)
{
    // 1. 解析参数：冒号分隔的字符串 → QStringList
    QString strVec = QString().fromStdString(cmd_value);
    QStringList vecList = strVec.split(":");
    // "50:20:30:80:50:20:30:80:50:20"
    // → ["50","20","30","80","50","20","30","80","50","20"]

    // 2. 参数校验：长度必须等于振动器总数
    if (vecList.size() != struCnfg.nChuteTotal)
    {
        return CMD_PARA_ERROR;    // 返回错误码 3
    }

    // 3. 写入全局方案参数
    for (int i = 0; i < vecList.size(); i++)
    {
        struCnfp.struGroupCtrl[struGsh.ctrlboardIndex].nFeederValueMain[i]
            = vecList.at(i).toInt();
        // nFeederValueMain[0] = 50
        // nFeederValueMain[1] = 20
        // ...
    }

    // 4. 让新配置在硬件上生效
    myFlow.resetFeederRG();
    return SUCCESS;
}
```

## resetFeederRG() — 配置到硬件的最后一环

内部实现（`globalflow.cpp`）：
1. 从 `struCnfp.nFeederValueMain[]` 读取新的供料值
2. 组装串口命令帧（CMD 命令字 + 板卡地址 + 数据字节）
3. 调用 `mySerial.pushCom1CmdData(...)` 入队发送
4. 串口发送线程 → 串口线 → PLC/控制板
5. PLC 收到 → 改变振动器的 PWM 占空比/DAC 输出
6. 振动器振幅改变 → 供料量改变

## onParaUpload — 与 CMD_Feed 的对比

```cpp
int mqttMsgParaseThread::onParaUpload(QString cmd_value)
{
    return SUCCESS;   // 函数体为空
}
```

doParseMessage 中的调用：

```cpp
else if (tmp == string("CMD_ParaUpload"))
{
    onParaUpload(value);                // 返回 SUCCESS
    onCmdReturn(uuid, tmp, SUCCESS);    // 先回传成功
    n_uploadRealTimeParaFlag = 1;       // 设标志位 → 异步上传
}
```

| | CMD_Feed | CMD_ParaUpload |
|------|---------|---------------|
| 操作类型 | 控制硬件（写振动器值 → 串口发送） | 数据上报（HTTP POST 到云端） |
| 执行方式 | 同步（在 doParseMessage 中调 onCmdFeed） | 异步（返回 SUCCESS + 设标志位） |
| 耗时 | 微秒级（写内存 + 串口入队） | 可能几百毫秒（HTTP 网络请求） |
| 回传时机 | 操作完成后立即回传 | 先回传 SUCCESS，数据异步上报 |
