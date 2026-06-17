# factoryset/ 工厂设置详解

---

## 一、目录定位

`factoryset/` 是**工厂级配置页面**的集合——比 `mainpage/` 更底层，直接操作硬件参数。所有页面继承 `basewidget`，通过 `FactorySetWidget` 作为导航枢纽。

---

## 二、导航枢纽 — FactorySetWidget

```
FactorySetWidget = 工厂设置的"主菜单"

9 个图标按钮 → 跳转各子页面:
  [语言]    [机器型号]   [相机设置]
  [信号]    [像素划分]   [IPC设置]
  [FPGA升级] [机器功能]   [灯板配置]
```

每个按钮点击 → `g_MainManager().ShowWidget(SM_XXX_WIDGET)`。

---

## 三、FPGA 固件升级（5 个文件）

### 3.1 老方案 — `fpga` / `fpga2`（直接继承 QDialog）

```
QDialog → fpga (76KB × 2)
```

**不继承 basewidget**，是较早的代码风格。提供 5 步升级向导：
1. 挂载 U 盘 → 2. 选升级文件 → 3. 确认信息 → 4. 单板升级 → 5. 整机升级

fpga 用于米机，fpga2 用于玉米机（类名都叫 `fpga`，靠文件名区分）。

**升级流程的核心状态机**：

```
switchToFactory() → sendFile() → burnFile() → switchToUser()
      ↓                ↓            ↓              ↓
   串口发切换指令    通过USB/串口    发烧录指令      串口发切回指令
                   传固件二进制
```

### 3.2 新方案 — `UpdateFpgaWidget` / `OneKeyUpdateFpgaWidget`

```
basewidget → UpdateFpgaWidget    (81KB, 单板升级)
basewidget → OneKeyUpdateFpgaWidget (87KB, 一键整机升级)
```

新方案改为**定时器驱动的异步状态机**，14 个定时器 ID 对应升级的不同阶段，不阻塞 UI。

**OneKeyUpdateFpgaWidget 一键升级流程**：

```
选 zip 升级包 → 点开始 → 自动依次升级:
  接口板 → 前主相机 → 前辅相机 → 后主相机 → 后辅相机
  → 阀板 → 控制板 → 灯板 → 振动检测板 → 电流检测板
每块板: 切工厂模式 → 传固件 → 烧录 → 切用户模式 → 查版本确认
```

### 3.3 `SelectUpdateFileWidget`（文件选择器）

列出 U 盘中的 `.zip` 固件包，解压后跳转到 `OneKeyUpdateFpgaWidget`。

---

## 四、相机设置（7 个文件）

### 4.1 CameraSetWidget（39KB）— 传感器和算法配置

每个相机单元可配置：
- **传感器类型**：黑白/彩色/红外，像素数（256/512/1024/2048）
- **传感器维数**：1（单色）/ 2（双色）/ 3（RGB）/ 4（RGB+IR）
- **位宽**：8/10/12 bit
- **行频**
- **使能的算法**：灰度、异色、SVM、形状SVM、大小、长短、圆长、HSV、霉斑、轮廓、线、芽头、圆、茶色...

**编辑模式**：先在 `m_temCnf`（临时副本）上改，点"应用"后一次性提交到 `struCnfg` → `myFlow.sendAll()`。

### 4.2 CameraSignalWidget（22KB）— 相机信号校准

```
[偏置 R: 128] [模拟增益 R: 32] [目标值 R: 2000]
[偏置 G: 128] [模拟增益 G: 32] [目标值 G: 2000]
[偏置 B: 128] [模拟增益 B: 32] [目标值 B: 2000]
[偏置 N: 128] ...

[明场校正开始] [暗场校正开始] [一键校正]
[波形显示区域]
```

**明场校正**：照射白板 → 捕获各像素值 → 计算增益系数 → 保存
**暗场校正**：关闭光源 → 捕获暗电流 → 计算偏置 → 保存

### 4.3 其他相机相关

| 文件 | 大小 | 用途 |
|------|------|------|
| `cameracolorrestore` | 4.8KB | 3×3 颜色还原矩阵（RGB 通道互相修正） |
| `ipccamerasetwidget` | 27KB | IPC 网络相机配置：IP 地址、ROI 区域、最多 20 路/视 |
| `ipcsetwidget` | 5.9KB | IPC 设置的导航菜单 |
| `ipcaisetwidget` | 4.7KB | IPC AI 参数（骨架，大部分代码注释掉了） |
| `ipcalarmcontrol` | 2KB | IPC 报警处理模式（重启IPC / 报警+停料） |

---

## 五、像素/传感器（4 个文件）

### 5.1 DividePixelWidget（10KB）— 像素通道划分

每个相机单元的像元被划分为多个通道，每个通道对应一段像元区域和一个喷阀：

```
像元: [0] [1] [2] ... [256] ... [512] ... [1024]

通道1: [起始: 0] ───────────────────── [截止: 256]  → 喷阀 [起始:0] ~ [截止:16]
通道2: [起始: 257] ─────────────────── [截止: 512]  → 喷阀 [起始:17] ~ [截止:32]
```

这是分选机最核心的硬件配置——每个喷阀吹哪些像元。

### 5.2 其他像素相关

| 文件 | 用途 |
|------|------|
| `pixeldivisionshowwidget` | 所有层/视/单元的像素划分汇总表（只读） |
| `pixeladjust` | 亚像素校正：R/G/B/N 通道的垂直偏移（-15 ~ +15） |
| `nirdotcorrectivelistwidget` | 红外坏点管理：捕获/显示/发送/清除坏点列表 |

---

## 六、机器配置（4 个文件）

### 6.1 MachineSetWidget（41KB）— 核心机器型号配置

```
机器类型: [立式米机] [履带机LD] [双层履带LD2]
层数: [1] [2]
每层视数: [1] [2]
每单元通道数: [64]
每通道阀数: [1] [2]
阀分组: 通道1-20→组1, 通道21-40→组2, ...
拼接模式: [无] [双机拼接]
```

**阀分组算法**（约 100 行）是最复杂的部分——将物理通道按规则分配到逻辑识别组，处理拼接机的特殊分组逻辑。

### 6.2 MachineFunction（55KB）— 杂项系统功能

13 个 Tab 页：

```
机器型号 | 报警 | 流量 | 恢复 | SVM | 参数 | ACC | 智能控制 | PLC | 阀/滤/擦/灯寿命
```

关键功能：
- **报警配置**：供料器料位、温度阈值
- **均衡供料**：料斗深度、空/满位、步长、模式选择
- **SVM 系数**：调整 SVM 算法的 10 个系数
- **QR 码生成**：用 libqrencode 生成设备二维码
- **Logo 替换**：从 U 盘更新 boot.bmp / homepage.bmp
- **调试模式**：开关 debug 日志输出

### 6.3 其他配置

| 文件 | 用途 |
|------|------|
| `languagesetwidget` | 26 种语言选择，设置 `g_languageIndex` → `g_myLan().ChangeLanguage()` |
| `lampboardcfg` | 灯板（恒流源板）配置：每板 8 个灯口，支持多层/多视/背景光/物料光 |
| `feedersensorsetwidget` | 8 个供料器料位传感器：启用/位置/名称 |
| `tempsensorsetwidget` | 5 个温度传感器：阈值/启用/名称 |

---

## 七、辅助功能（2 个文件）

| 文件 | 用途 |
|------|------|
| `updateacswidget` | IPC ACS 算法文件管理——NFS 挂载 IPC、浏览/复制/删除文件，验证 MD5 |
| `onekeyupdatefpgawidget` | 已在 FPGA 组介绍 |

---

## 八、跨文件一致性模式

### 8.1 临时副本编辑模式

多个页面使用 `m_temCnf`（`struCnfGlobal` 的副本）：

```
showPage() → m_temCnf = struCnfg      (复制)
用户编辑   → 改的是 m_temCnf
点"应用"   → struCnfg = m_temCnf     (提交)
           → myFlow.saveGlobal()
           → myFlow.sendAll()         (下发到硬件)
```

好处：用户乱改后点"取消"就恢复，不会弄坏硬件参数。

### 8.2 myFlow 汇聚模式

所有硬件通信都经过 `myFlow`：

```
UI 操作 → 改全局结构体 → myFlow.xxx() → 串口/TCP/UDP → 硬件
```

### 8.3 定时器驱动的异步升级

老方案（fpga/fpga2）用同步阻塞方式（sleep 等待），新方案（UpdateFpgaWidget）用定时器分步执行——不阻塞 UI，升级过程中用户可以看进度。

### 8.4 新旧代码并存

`fpga`（直接继承 QDialog）和 `UpdateFpgaWidget`（继承 basewidget）是同一功能的两个实现。旧代码保留未删，新代码在功能上更完整（支持更多板类型、异步升级）。

---

## 九、文件总览

| 文件 | 规模 | 继承 | 用途 |
|------|------|------|------|
| factorysetwidget | 7.4KB | basewidget | 工厂设置主菜单（9 图标网格） |
| machinesetwidget | 41KB | basewidget | 机器型号/层数/通道/阀分组 |
| machinefunction | 55KB | basewidget | 13 Tab 杂项系统功能 |
| camerasetwidget | 39KB | basewidget | 传感器类型/像素/算法使能 |
| camerasignalwidget | 22KB | basewidget | 偏置/增益/明暗场校正 |
| cameracolorrestore | 4.8KB | basewidget | 3×3 颜色还原矩阵 |
| ipccamerasetwidget | 27KB | basewidget | IPC 相机 IP/ROI 配置 |
| ipcsetwidget | 5.9KB | basewidget | IPC 设置导航菜单 |
| ipcaisetwidget | 4.7KB | basewidget | IPC AI 参数（骨架） |
| ipcalarmcontrol | 2KB | basewidget | IPC 报警控制 |
| dividepixelwidget | 10KB | basewidget | 像素通道/喷阀范围划分 |
| pixeldivisionshowwidget | 5.7KB | basewidget | 像素划分汇总表 |
| pixeladjust | 4KB | basewidget | 亚像素垂直偏移校正 |
| nirdotcorrectivelistwidget | 7.4KB | basewidget | 红外坏点管理 |
| languagesetwidget | 4.6KB | basewidget | 26 种语言选择 |
| lampboardcfg | 15KB | basewidget | 灯板端口配置 |
| feedersensorsetwidget | 4.7KB | basewidget | 供料器料位传感器 |
| tempsensorsetwidget | 3.5KB | basewidget | 温度传感器 |
| updateacswidget | 11KB | basewidget | IPC ACS 文件管理 |
| fpga | 76KB | QDialog | 米机 FPGA 升级向导（旧版） |
| fpga2 | 76KB | QDialog | 玉米机 FPGA 升级向导（旧版） |
| updatefpgawidget | 81KB | basewidget | 定时器驱动 FPGA 单板升级（新版） |
| onekeyupdatefpgawidget | 87KB | basewidget | 一键整机多板 FPGA 升级（新版） |
| selectupdatefilewidget | 5.5KB | basewidget | 固件 zip 文件选择器 |
