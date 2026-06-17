# systeminfo/ 系统信息详解

---

## 一、目录定位

`systeminfo/` 是**系统监控、诊断和测试**页面集合。与前面模块不同——这里大部分页面只**读**不写，用于查看状态而非修改配置。核心是 IPC 测试套件（73KB）和版本信息（26KB）。

---

## 二、导航枢纽 — SysStateWidget

```
SysStateWidget = 系统信息的"主菜单"

9 个按钮 → QStackedWidget:
  [系统信息] [报警设置] [软件版本] [设备信息] [远程协助]
  [IPC信息] [IPC分类信息] [VI板信息] [WiFi设置]
```

IPC 相关按钮仅在 DP 机器（`struCnfg.nDPMachine`）上显示。

---

## 三、系统信息页（6 个页面）

### 3.1 SysInfoGeneralPage — 实时状态总览

```
QWidget → SysInfoGeneralPage（不继承 basewidget，独立实现）
```

1 秒定时器驱动刷新，显示：

```
本次运行时间: 03:25:18
总运行时间:   1523h
系统时间:     2026-05-27 14:30:00
供料状态:     开启 (红) / 关闭 (黑)
喷阀状态:     开启 (红) / 关闭 (黑)
气压报警:     正常 / 异常
风机报警:     正常 / 异常
灯板报警:     正常 / 异常
履带1状态:    运行中
履带2状态:    停止
```

颜色编码：正常=黑/蓝，异常=红。

### 3.2 SysInfoLog — 日志查看器

按日期选择（年/月/日）→ 读取 `/sdcard/logs/log.txt` → 筛选匹配日期的行 → 显示在 QTextEdit。

**AI 分析功能**（隐藏按钮，仅 USB 插入时可见）：发送 UDP 指令到 IPC 板 → 远程执行分析 → 读取结果文件。

### 3.3 SysInfoNet — 网络扫描器

输入起止 IP → 启动 shell 脚本 `findActiveIp.sh` → 定时读 `/sdcard/ts/active_ip.txt` → 显示活跃 IP 列表。

### 3.4 SysInfoDebug — 调试终端

输入 shell 命令 → 执行方式：
- **ARM 目标**：`QProcess` 本地执行或特殊命令（`zktime`、`zkdebug on/off`）本地处理
- **RK/IPC 目标**：通过 UDP 发送到 IPC 板执行 → 读取结果文件

### 3.5 MachineInfoWidget — 设备信息仪表盘

1 秒刷新，类似 SysInfoGeneralPage 但更详细：

```
IP 地址(eth0/eth1)、供料状态、气压报警、料位报警
MQTT 连接状态、IPC 报警、气压值、耗气量
温度传感器 (×5)、料位传感器 (×8)
```

### 3.6 AlarmSetWidget — 报警开关

7 个复选框：供料报警 / 清灰报警 / 气压报警 / 料位报警 / 断电报警 / 停料使能 / IPC 报警。勾选 = 启用该报警类型。

---

## 四、版本/状态（4 个页面 + 2 个图表控件）

### 4.1 SysVersionInfoWidget（26KB）— 固件版本总表

**5 列表格**显示所有硬件板的固件版本：

| 板卡名称 | 状态 | 版本号 | 产品模式 | SVN号 |
|---------|------|--------|---------|-------|
| 接口板 | ✓ | 1.04.A(F) | ZL_RICE | 1234 |
| 前主相机1-1 | ✓ | 2.1.0 | ZL_RICE | 5678 |
| ... | | | | |
| 控制板 | ✗ | — | — | — |

版本号通过 `myFlow.resetCommunication()` 从各硬件板查询获得。

**额外显示**：屏幕固件编译时间（`__DATE__` `__TIME__`）、加密授权信息。

### 4.2 BarChart（16KB）— 柱状图控件

```
QWidget → BarChart（不继承 basewidget，纯手绘）
```

用于显示图像统计数据的柱状图（HSV/RGB/变色比例直方图）。纯 `paintEvent` 手绘——画坐标轴 → 画数据柱 → 画网格和标注。数据来自 `g_Runtime().m_imgDataMap` 或 SQLite 数据库。

### 4.3 PlotCurve（12KB）— 折线图控件

```
QWidget → PlotCurve（不继承 basewidget，纯手绘）
```

用于显示产量/含杂的时间序列折线图。数据点来自 `myQueryStatisticThread`，支持实时和昨日两种查询模式。

### 4.4 vavletest（13KB）— 喷阀测试

```
basewidget → vavletest + vavleTestThread (QThread)
```

三种测试模式：

| 模式 | 间隔 | 说明 |
|------|------|------|
| 单阀测试 | 手动选择 | 测试一个喷阀 |
| 快速测试 | 300ms | 遍历所有喷阀 + 录音 |
| 信号自检 | — | 检查喷阀信号是否正常 |

快速测试会录音（`arecord`），通过声音判断喷阀是否真正动作，录音可通过 MQTT 上传。

---

## 五、IPC 信息（6 个页面）

### 5.1 IPCInfoWidget（73KB）— 最大的系统信息页面

```
basewidget → IPCInfoWidget
```

**10 列表格**显示每个 IPC 相机板的详细状态：

```
板卡名称 | 软件版本 | ROI | 模型状态 | 离线率 | 空间 | NPU/CPU/内存 |
温度 | 行频 | 图像计数
```

**三大自动化测试**：

**① 关机循环测试**：

```
配置次数 N（默认 50 次）
循环: 开机 → 等待模型加载 → 记录在线状态 → 关机
      → 重复 N 次 → 生成报告
```

测试每块相机板在反复上电后的稳定性。报告记录每块板的在线次数、模型加载成功次数。

**② 压力测试**：

```
配置时长 M 小时（默认 8 小时）
连续运行图像采集和推理
记录: 离线率、模型延迟、行频、温度
```

**③ 采集测试**：发送采集指令 → 接收结果 → 记录成功率。

---

### 5.2 其他 IPC 页面

| 页面 | 大小 | 用途 |
|------|------|------|
| `ipcclassinfowidget` | 3.4KB | 显示 IPC AI 模型分类名称（类别号、类别名） |
| `ipctestnetwidget` | 16KB | IPC 网络丢包率测试——发送测试包、统计总数/丢包/速度 |
| `sysipcbadpointswidget` | 8.2KB | IPC 坏点/含杂查询——按日期范围查 SQLite 数据库的含杂数据 |
| `videtectboardinfowidget` | 9.8KB | VI（电压-电流）检测板监控——每板电压/电流、超限报警 |
| `ipcruntestwidget` | — | IPC 运行测试（仅有 .h 和 .ui，无 .cpp 实现） |

---

## 六、网络设置（4 个页面）

### 6.1 IPSetWidget（14KB）— 有线网络配置

```
eth0 / eth1 双网口:
  IP 地址 | 子网掩码 | 网关 | DNS

OpenVPN 远程连接:
  [连接] [断开] [导入证书] [申请证书]
  VPN 心跳 ping 10.18.0.1（每 20 秒）
```

### 6.2 WifiConfigWidget / WifiInterface — WiFi 管理

```
WifiConfigWidget (UI层) → WifiInterface (底层，继承 QObject)
                              ↓
                         QProcess → /sdcard/wifi/wifiCtrol.sh
```

两种模式：
- **STA 模式**：扫描 → 选 SSID → 输密码 → 连接 → DHCP/静态 IP
- **AP 模式**：配置 SSID/密码/IP → 查看已连接设备 MAC 列表

### 6.3 RemoteWidget — 远程设置容器

Tab 容器：网络配置 + PLC 控制（PLC Tab 仅在 `bautoplcctrlEnable` 时可见）。

---

## 七、AutoCtrlBoardIPSetWidget — PLC 自动控制板配置

通过串口（`CMD_CTRL_BOARD_IP_PORT_QUERY/SET`）配置自动控制板的：

```
IP 地址 | 端口 | 网关 | 子网掩码 | Modbus 从站 ID | 波特率
```

---

## 八、关键设计模式

### 8.1 定时器驱动刷新

多个页面用 QTimer 每秒刷新状态：

```
SysInfoGeneralPage: 1 秒 → updateGeneralPage()
MachineInfoWidget:  1 秒 → updateInfo()
SysVersionInfoWidget: 查询触发 → updateVersionInfo()
IPCInfoWidget:      10 秒 → recordLineInfo()
```

### 8.2 纯手绘图表

BarChart 和 PlotCurve 都直接继承 QWidget（不继承 basewidget），在 `paintEvent` 中完全手绘。坐标轴自动缩放、网格线、数据标注全部手动计算和绘制。

### 8.3 后台线程模式

耗时的查询操作放在独立线程中：

```
vavleTestThread:         喷阀逐个测试（300ms/阀）
getBadPointsThread:      SQLite 数据库查询
ipcTestNetThread:        IPC 网络丢包测试
SysInfoDebugThread:      远程调试命令执行
pingThread:              VPN 心跳 ping
```

---

## 九、文件总览

| 文件 | 规模 | 继承 | 用途 |
|------|------|------|------|
| **ipcinfowidget** | **73KB** | basewidget | IPC 关机循环/压力/采集测试套件 |
| **sysversioninfowidget** | **26KB** | basewidget | 所有硬件板固件版本表 |
| **barchart** | **16KB** | QWidget | 手绘柱状图 |
| **ipctestnetwidget** | **16KB** | basewidget | IPC 网络丢包测试 |
| **ipsetwidget** | **14KB** | basewidget | 双网口 IP + OpenVPN 配置 |
| **vavletest** | **13KB** | basewidget | 喷阀遍历测试 |
| **plotcurve** | **12KB** | QWidget | 手绘折线图 |
| **machineinfowidget** | **12KB** | basewidget | 设备状态仪表盘 |
| videtectboardinfowidget | 9.8KB | basewidget | VI 检测板监控 |
| wificonfigwidget | 9.1KB | basewidget | WiFi 配置 |
| sysinfolog | 8.4KB | basewidget | 日志查看器 |
| sysipcbadpointswidget | 8.2KB | basewidget | IPC 含杂查询 |
| sysinfodebug | 6.8KB | basewidget | 调试终端 |
| wifiinterface | 5.7KB | QObject | WiFi 底层控制 |
| sysinfogeneralpage | 5.1KB | QWidget | 实时状态总览 |
| autoctrlboardipsetwidget | 7.4KB | basewidget | PLC 控制板 IP |
| 其余 | < 4KB | — | 容器/简单页面 |
