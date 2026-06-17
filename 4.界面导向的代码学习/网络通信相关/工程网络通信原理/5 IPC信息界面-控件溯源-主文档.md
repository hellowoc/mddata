# IPC 信息界面 — 控件溯源

## 涉及的文件

| 文件 | 内容 |
|------|------|
| `CameraTest(aiuse)/zksort/systeminfo/ipcinfowidget.h` | IPC 信息页类定义、IPCVersionInfo 数据类 |
| `CameraTest(aiuse)/zksort/systeminfo/ipcinfowidget.cpp` | 约 2000 行，全部控件逻辑 |
| `CameraTest(aiuse)/zksort/systeminfo/ipcinfowidget.ui` | UI 布局 |

---

## 1. 这个页面是干什么的

IPC 信息页是**工程师调试 IPC AI 工控机的核心页面**。提供三类功能：

| 功能 | 说明 |
|------|------|
| 状态显示 | 表格展示每台 IPC 相机的版本、ROI、模型状态、CPU/内存/温度 |
| 测试工具 | 开关机循环测试、压力测试、行连续性测试 |
| 关机控制 | FPGA 相机板循环关机测试 |

---

## 2. 页面生命周期

```
IPCInfoWidget 构造（ipcinfowidget.cpp:15-146）
  ├── setupUi
  ├── 连接信号：
  │   ├── myFastNetThread → handleCaptureTestResult  （行连续性测试结果）
  │   ├── myFastNetThread → handleImageReceived       （抓图结果）
  │   ├── myFastNetThread → handleCaptureStatus       （抓图状态）
  │   ├── myUpdateStatusThread → updateALl            （IPC 状态刷新）
  │   ├── myFlow.myNetWork->udpthread → doCameraShut  （波形采集完成 → 相机关机）
  │   └── m_updateTimer → updateDelayValues           （每秒更新模型延迟）
  ├── initSettings()                                  从 INI 读测试参数
  └── initSettingsDialog()                            创建设置对话框

showPage(true)
  → m_timer->start()      启动行信息记录定时器
  → updateVersionInfo()   立即刷新 IPC 版本信息
  → updateALl()           立即刷新所有 IPC 状态

showPage(false)
  → m_timer->stop()
```

---

## 3. 控件逐一溯源

### 3.1 IPC 状态表格 — tableWidget

**创建**：`ipcinfowidget.ui` + 构造函数配置列（第 65-88 行）

**列定义**（共 10 列）：

| 列 | 宽度 | 标题 | 内容 |
|-----|------|------|------|
| 0 | 150 | 电路板 | 层/视野/单元名称 |
| 1 | 250 | 版本 | IPC 固件版本 + SDK 版本 |
| 2 | 150 | ROI | roiWidth × roiHeight |
| 3 | 150 | 模型状态 | 未加载/加载中/成功/失败 |
| 4 | 250 | 丢行 | 丢失行数 / 总行数 |
| 5 | 100 | 剩余空间 | IPC 磁盘剩余空间（MB） |
| 6 | 150 | 资源 | NPU/CPU/内存使用率 |
| 7 | 100 | 温度 | 芯片温度（℃） |
| 8 | 100 | 行频 | 相机行频（Hz） |
| 9 | 150 | 图片数 | IPC 端抓图数量 |

**刷新函数**：`updateTableWidget()`（用 `m_verinfoVec` 数据填充表格）

**数据来源**：`[[references/全局参数|struIpcShare]].struIpcInfo[view][unit]`

**数据写入者**：`SendThread` 中的 `processTheDatagram()` 收到 IPC 心跳包（`CMD_UDP_IPC_REQ_INFO`, `0x0001`）后写入 `struIpcShare`。

**刷新触发**：

```
myUpdateStatusThread → emit sUpdateIpcInfo() → updateALl()
  → updateVersionInfo()    读 struIpcShare → 填 m_verinfoVec
  → updateTableWidget()    读 m_verinfoVec → 填表格
```

**线程**：`myUpdateStatusThread`（子线程）emit → 主线程槽执行刷新。

---

### 3.2 模型延迟值 — 各标签

**刷新函数**：`updateDelayValues()`（第 132-134 行设置定时器每秒调用）

```cpp
m_updateTimer = new QTimer(this);
m_updateTimer->setInterval(1000);
connect(m_updateTimer, SIGNAL(timeout()), this, SLOT(updateDelayValues()));
m_updateTimer->start();
```

**数据来源**：`[[references/全局参数|struGsh]].sRowTotal[]` 数组。

**数据写入者**：FPGA 相机线程采集波形时写入。

---

### 3.3 行连续性测试按钮 — pushButton_StartTest

**创建**：`ipcinfowidget.ui`

**点击→槽**：`on_pushButton_StartTest_clicked()`（`ipcinfowidget.cpp:853`）

**操作链路**：

```
用户点击"开始测试"
  → 清除上次测试报告文件
  → 重置计数器（m_offlineCount 等）
  → 遍历所有启用的 IPC 相机：
      检测在线状态 → 记录到日志
  → m_timer->start()            启动行信息记录定时器
  → m_bAutoShutdown = true       启用自动关机循环
  → updateShutdownCount(0)       重置关机计数器
  → performShutdown()            开始第一轮关机测试
```

**背后线程**：用户点击在主线程。`performShutdown()` 流程见下文。

---

### 3.4 压力测试按钮 — pushButton_PressureTest

**创建**：`ipcinfowidget.ui`

**点击→槽**：`on_pushButton_PressureTest_clicked()`

**操作链路**：

```
用户点击"压力测试"
  → 从设置读取测试时长（默认 8 小时）
  → 启动 myFastNetThread 在 FAST_CAPTURE_IMAGE 模式
    → myFastNetThread->isrunning = true;
    → myFastNetThread->start();
  → 启动定时器记录丢行和模型延迟
  → UI 切换到"压力测试进行中"状态
  → 时间到或用户点击"停止" → stopPressureTest()
    → myFastNetThread->isrunning = false
    → generatePressureTestReport()  生成测试报告
```

**背后线程**：
- `myFastNetThread`（子线程）持续接收图像，检测丢行
- 定时器在主线程，每秒读取统计值更新 UI
- `myFastNetThread` emit `captureTestResult(view,unit,lost,total)` → 主线程槽更新丢行显示

---

### 3.5 模型状态代码含义

表格第 3 列显示的模型状态，对应 `stu_ipc_info.modelStat`：

| 值 | 含义 |
|-----|------|
| 0 | 未加载模型 |
| 1 | 模型加载中 |
| 2 | 加载成功 |
| 3 | 加载失败 — 模型文件不存在 |
| 4 | 加载失败 — 模型文件错误 |
| 5 | 加载失败 — 分选状态不匹配 |

---

### 3.6 设置按钮 — pushButton_Setting

**创建**：`ipcinfowidget.cpp:222-299`（`initSettingsDialog()`）

**点击→槽**：`on_pushButton_Setting_clicked()`（第 216 行）→ `m_settingsDialog->show()`

**设置对话框**包含：
- `m_shutdownSpinBox`（QSpinBox）：关机次数 1-100
- `m_pressureSpinBox`（QDoubleSpinBox）：推理时长 0.1-48 小时
- `m_okButton`：点击 → `onSettingsDialogAccepted()` → 保存设置到 INI 文件 + 更新日志文件

**线程**：全部在主线程。

---

### 3.7 FPGA 相机板循环关机按钮 — pushButton_Camerashut

**创建**：`ipcinfowidget.ui`，默认隐藏（第 139 行）

**点击→槽**：`on_pushButton_Camerashut_clicked()`（第 836 行）

**操作链路**：

```
用户点击"循环关机"
  → 弹确认框
  → camerashut = true
  → doCameraShut()（立即执行第一次）
```

### doCameraShut() 的完整链路（第 785-833 行）：

```
doCameraShut()
  ├── 如果 wavechecked == 0 → 停止采图，返回
  │
  ├── myFlow.shutdownIpc(true, 0, 0, 0)          // 1. 关闭 IPC
  │     → g_Udp.pushUdpCmdData(CMD_UDP_IPC_REQ_SHUTDOWN, ...)
  │     → SendThread → writeDatagram → IPC 收到关机命令
  │
  ├── mySerial.pushCom2CmdData(CMD_UPPER_COMPUTER_OUTAGE_REQUEST, ...)
  │     // 2. 通过串口通知 PLC 控制板断电
  │     // BOARD_TYPE_CONTROL：发送给控制板
  │     // data[0-1]：延时关机时间
  │
  ├── g_Runtime().save()                          // 3. 保存所有配置到文件
  │
  ├── myMqttMsgParaseThread->n_uploadLogFileFlag = 1  // 4. 通知 MQTT 上传日志
  │
  └── system("shutdown -h now")                   // 5. Linux 系统关机
```

**背后线程**：
- 关机命令 `g_Udp.pushUdpCmdData` 在主线程调，由 `SendThread`（子线程）发出
- 串口命令 `mySerial.pushCom2CmdData` 在主线程调，由串口发送线程发出
- `system("shutdown -h now")` 是阻塞调用——Linux 内核立即开始关机流程

### 循环关机的工作机制

循环关机依赖硬件自动上电（PLC 控制的定时上电）。每次关机前记录一次，上电后程序自动启动 → `IPCInfoWidget` 构造函数读日志发现 `m_bAutoShutdown == true` → 继续新一轮测试。

**检测自动重启**：构造函数第 99 行：
```cpp
m_bAutoShutdown = (shutdownCount > 0 && shutdownCount < m_customShutdownCount);
```

**波形采集完成后触发下一次关机**：第 141 行的信号连接：
```cpp
connect(myFlow.myNetWork->udpthread, SIGNAL(SignalWaveRecvFinished()),
        this, SLOT(doCameraShut()));
// 波形采集成功 → 说明系统已正常启动 → 执行下一次关机
```

---

## 4. 信号连接全景图

```
IPCInfoWidget 的信号连接（构造函数中建立）

myFastNetThread（通道3 子线程）
  ├── captureTestResult(view,unit,lost,total)
  │     → handleCaptureTestResult()      ← 主线程
  ├── SignalImageRecvFinished(pathList,timeList)
  │     → handleImageReceived()          ← 主线程
  └── captureStatus(view,unit,success)
        → handleCaptureStatus()          ← 主线程

myUpdateStatusThread（状态轮询子线程）
  └── sUpdateIpcInfo()
        → updateALl()                    ← 主线程

myFlow.myNetWork->udpthread（通道1 子线程）
  └── SignalWaveRecvFinished()
        → doCameraShut()                 ← 主线程

m_timer（主线程定时器）
  └── timeout()
        → recordLineInfo()               ← 主线程

m_updateTimer（主线程定时器）
  └── timeout()
        → updateDelayValues()            ← 主线程

myTimer（主线程定时器）
  └── timeout()
        → TimeOutSlt() → updateInfo()    ← 主线程
```

---

## 5. 三个测试的线程参与

| 测试 | UI 线程 | 命令线程 | 数据线程 | 硬件 |
|------|---------|---------|---------|------|
| 开关机循环 | 点击按钮、定时刷新 | SendThread 发关机命令 | myNetWorkThread 接收波形确认启动 | IPC + PLC + Linux |
| 压力测试 | 点击按钮、定时刷新 | SendThread 发启动命令 | **myFastNetThread** 收图像、检测丢行 | IPC 推送图像 |
| 行连续性测试 | 点击按钮 | SendThread 发启动命令 | **myFastNetThread** 收包、检查行号 | IPC 推送图像 |
| FPGA 相机循环关机 | 点击按钮 | 串口线程发 PLC 命令 | myNetWorkThread 收波形触發下一次 | FPGA 相机板 + PLC |
