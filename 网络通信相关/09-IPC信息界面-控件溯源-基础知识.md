# IPC 信息界面控件溯源 — 基础知识清单

按在 [[09-IPC信息界面-控件溯源-主文档|主文档]] 中首次出现的顺序排列。

---

## 第 1 章出现的

### ROI（§1）

ROI = Region Of Interest（感兴趣区域）。相机拍摄的图像中，只有 ROI 范围内的像素参与 AI 推理。设置 ROI 可以裁剪掉不关心的边缘区域，减少计算量。

在 `struIpcShare.struIpcInfo[view][unit]` 中以 `roiWidth` × `roiHeight` 表示。

### NPU（§1）

NPU = Neural Processing Unit（神经网络处理单元）。IPC 工控机上专用的 AI 推理加速芯片。与 CPU/GPU 的区别：
- CPU：通用计算，什么都能做但 AI 推理慢
- GPU：图形/并行计算，AI 推理比 CPU 快
- NPU：专为 AI 推理设计，能效比最高（本工程用的 IPC 使用 RK 系列 NPU）

---

## 第 2 章出现的

### IPCVersionInfo（§2 和 §3.1）

`ipcinfowidget.h:31-53` 中定义的数据类。不是 Qt 控件，而是一个**纯数据载体**——把一行表格数据打包成一个对象，方便在 `QList<IPCVersionInfo>` 中管理。

```cpp
class IPCVersionInfo {
    QString name;        // "层1-视野1-单元1"
    QString ver;         // "v1.2.3 / SDK v4.5"
    QString roiInfo;     // "1024×832"
    QString modelStat;   // "加载成功"
    QString lineInfo;    // "0 / 50000"
    QString freeSpace;   // "2048 MB"
    QString cpuInfo;     // "45/30/60"
    QString tempInfo;    // "55℃"
    QString lineFreq;    // "15000 Hz"
    QString picCnt;      // "1234"
    int offlineStat;     // 0=在线, 1=离线
};
```

### updateVersionInfo()（§2）

从 `[[references/全局参数|struIpcShare]]` 读取每台 IPC 的状态，填写到 `m_verinfoVec`（`QList<IPCVersionInfo>`）中。遍历所有层/视野/单元，对每台启用的 IPC 相机创建一个 `IPCVersionInfo` 对象。

### updateTableWidget()（§2）

从 `m_verinfoVec` 读取数据，逐行填充到 `ui->tableWidget`。每台 IPC 相机一行。

### initSettings()（§2）

从 INI 配置文件读取测试参数（关机次数、压力测试时长），写入日志文件。

### initSettingsDialog()（§2）

创建"设备测试设置"弹窗。使用 QDialog + 纯代码布局（QVBoxLayout + QHBoxLayout + QLabel + QSpinBox + QDoubleSpinBox）。

---

## 第 3.1 节出现的

### struIpcShare（§3.1）

见 [[references/全局参数#struIpcShare (struIpcShareInfo)]]。

### sUpdateIpcInfo()（§3.1）

`myUpdateStatusThread` 的信号。发射频率由线程内部控制（通常每秒或每 500ms 一次）。IPC 信息页连接此信号来定时刷新。

### updateALl()（§3.1）

`IPCInfoWidget` 的槽。内部调用 `updateVersionInfo()` → `updateTableWidget()`，完成一次完整的状态表刷新。

---

## 第 3.3 节出现的

### m_timer（§3.3）

静态成员变量 `IPCInfoWidget::m_timer`（`ipcinfowidget.h:129`）。记录行信息的定时器。

`static` 意味着所有 `IPCInfoWidget` 实例共享同一个定时器（虽然实际上只有一个实例）。

### getShutdownCount()（§3.3）

从日志文件 `ipc_info_log.txt` 中读取当前关机次数。

---

## 第 3.4 节出现的

### QElapsedTimer（§3.4）

Qt 的计时器类（`<QElapsedTimer>`）。用于精确测量时间间隔。比 `QTime` 更适合性能测量。

```cpp
QElapsedTimer timer;
timer.start();                          // 开始计时
// ... 做一些事情 ...
qint64 elapsed = timer.elapsed();       // 获取经过的毫秒数
```

在本页中用于跟踪压力测试的运行时长。

---

## 第 3.7 节出现的

### myMessageBox（§3.7）

本工程的自定义消息框（`mysortwidget/mymessagebox.cpp`）。替换 Qt 标准的 `QMessageBox`，支持自定义样式。

```cpp
myMessageBox msgBox(MSG_QUES, "确认开始循环关机");
int ret = msgBox.exec();                    // 模态显示
if (ret == QDialog::Accepted)               // 用户点了"确定"
    doCameraShut();
```

### CMD_UPPER_COMPUTER_OUTAGE_REQUEST（§3.7）

串口命令宏（`constant.h`）。上位机通知 PLC/控制板"我要断电了，请做好准备"。

### BOARD_TYPE_CONTROL（§3.7）

板卡类型常量（`constant.h`），值 = `0x07`。表示目标板卡是"控制板"（与 `BOARD_TYPE_ALL_CAMERA`（0x09）等区分）。

### system("shutdown -h now")（§3.7）

C 标准库函数 `system()`，执行 shell 命令。`"shutdown -h now"` 是 Linux 立即关机命令。

`-h` = halt（停止系统），`now` = 立即执行。这条命令会让 Linux 内核立即开始关闭所有服务、卸载文件系统、断电。

### 循环关机的"循环"是如何实现的（§3.7）

不是软件循环——软件本身也随着 Linux 关机而终止了。循环依赖的是**硬件层面的自动上电**：

```
1. 软件执行 system("shutdown -h now") → Linux 关机 → 整机断电
2. PLC 控制板检测到断电 → 开始计时
3. 计时到（如 30 秒后） → PLC 控制继电器闭合 → 整机重新上电
4. Linux 启动 → 自动启动 zksort 程序
5. zksort 初始化 → IPCInfoWidget 构造函数读日志
   → 发现 shutdownCount < m_customShutdownCount
   → m_bAutoShutdown = true
   → 继续下一轮测试
```

### PLC（§3.7）

PLC = Programmable Logic Controller（可编程逻辑控制器）。工业设备的"管家"——控制电源通断、电机启停、传感器采集。在这个工程中，PLC 负责根据上位机的命令执行断电和重新上电。

---

## 第 4 章出现的

### 跨线程信号连接的默认行为（§4）

`connect(myFastNetThread, SIGNAL(captureTestResult(...)), this, SLOT(handleCaptureTestResult(...)))` 使用默认 `AutoConnection`。

`myFastNetThread` 在子线程，`this`（IPCInfoWidget）在主线程。Qt 自动使用 `QueuedConnection`——子线程 emit → 事件入主线程队列 → 主线程事件循环取出执行。

### m_verinfoVec（§4）

`QList<IPCVersionInfo>` 成员。作为表格数据的**中间缓存**——`updateVersionInfo()` 从 `struIpcShare` 读数据写入 `m_verinfoVec`，然后 `updateTableWidget()` 从 `m_verinfoVec` 读数据写入 QTableWidget。

为什么需要中间缓存？因为填充 QTableWidget 是一项耗时的 UI 操作。先把数据准备好（纯数据操作，快），再一次性填入表格（UI 操作），比边读边填更清晰。
