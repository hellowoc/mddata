# CameraTest (aiuse) 工程全景框架

## 项目定位

这是一个**基于 Qt 的工业色选机（光学分拣机）上位机控制系统**。目标硬件是嵌入式 Linux 设备（ARM 平台），同时也支持 Windows 桌面调试。产品名称为 `zksort`（中科分选），版本号 V2.05PRE。

---

## 三层架构总览

```
solution.pro  (顶层subdirs)
├── 3rdparty/      ← 第三方依赖库层
├── uikits/        ← 自定义UI组件库层
└── zksort/        ← 主应用程序层 (核心)
```

---

## 第一层：`3rdparty/` — 第三方依赖（9个库）

| 库名 | 用途 |
|------|------|
| `log4qt` | 日志框架（Java log4j 的 Qt 移植版） |
| `qextserialport` | 串口通信库 |
| `jsoncpp` | JSON 解析库 |
| `qwt` | 图表/曲线绘制库 |
| `qrencode` | 二维码生成库 |
| `mqtt` | MQTT 物联网协议客户端（仅Linux） |
| `encrypt` | 加密/授权库 |
| `svm` | SVM（支持向量机）算法库，用于物料分类 |
| `fToFp` / `libsubpixel` | 图像处理/像素分析相关 |

---

## 第二层：`uikits/` — 共享 UI 组件库

编译为动态库（`libuikits.so` / `uikits.dll`），提供通用的自定义 UI 控件。

---

## 第三层：`zksort/` — 主应用程序（核心）

这是工程的核心，按功能模块划分如下：

### 1. 入口和框架层

| 文件 | 作用 |
|------|------|
| `main.cpp` | 程序入口，初始化日志、编码、启动画面、加载资源 |
| `zksort.h/cpp` | 主窗口（继承 BaseUi），全屏应用（1024×768），处理全局按键事件 |
| `middlewidgetmanager.h/cpp` | **页面管理器**（核心调度中心），用 QStackedWidget 管理所有页面的切换/创建/消息路由 |

### 2. 主窗口布局

| 文件 | 作用 |
|------|------|
| `topwidget.h/cpp` | 顶部状态栏 |
| `bottomwidget.h/cpp` | 底部导航栏 |
| `middlewidgetmanager.h/cpp` | 中间区域页面栈管理 |

### 3. 功能模块目录

| 目录 | 功能领域 | 关键文件举例 |
|------|---------|------------|
| `mainpage/` | **主页面群** — 用户主页、模式管理、供料设置、AI擦拭、自检、模拟 | `newmainpage`, `modemanagewidget`, `feedsetwidget`, `aiwipesetwidget`, `selfcheckwidget` |
| `factoryset/` | **工厂设置** — 相机配置、FPGA升级、灯板配置、语言设置、分像素调节 | `camerasetwidget`, `fpga.cpp`(超大文件76KB), `updatefpgawidget`, `ipccamerasetwidget` |
| `engineerset/` | **工程师调试** — 图像采集/分析/管理、SVM图像、背景设置、光源设置、AI参数 | `imagecapturewidget`, `imagewidget`(87KB), `svmimagewidget`, `aisetdialog`, `upgrademodelwidget` |
| `systeminfo/` | **系统信息** — 运行状态、版本信息、日志、IPC（网络相机）信息、WiFi配置、告警 | `sysstatewidget`, `sysversioninfowidget`, `ipcinfowidget`, `wificonfigwidget` |
| `arithprofilewidget/` | **算法方案参数** — 各种分选算法参数配置（20+种算法） | 灰度/HSV/颜色饱和度/色相/形状SVM/霉斑/芽头/茶色等分选方案 |
| `autocontrol/` | **PLC自动控制** — PLC联动控制逻辑 | `plcautoctrlmanager`, `plcctrlwidget` |
| `bus/` | **通信层** — 串口、网络(TCP/UDP)、HTTP文件、MQTT | `myqextserialport`(99KB), `mynetwork`(107KB), `myudpthread`, `myhttpfileclient` |
| `global/` | **全局基础设施** — 运行时环境、多语言(23种语言)、流程控制、全局参数 | `runtime`, `mylanguage`, `globalflow`(551KB超大状态机), `globalparams`(115KB), `material` |
| `data/` | **数据层** — JSON数据转换、SQLite数据库 | `jsondataconvert`, `myjson`, `mysqlite` |
| `mysortwidget/` | **自定义UI控件库** — 波形图、开关按钮、HSV色环、列表控件等 | `mywavewidget`, `myswitchwidget`, `hsvcolorcircle`, `mylistwidget` |
| `plastic/` | **塑料分选专用** — 加速度传感器、红外坏点、红外偏移 | `accelerationsensorsetwidget`, `irbadpointsetwidget` |

---

## 代码量级参考

- `globalflow.cpp` — **551KB**（核心状态机/业务流程）
- `globalparams.h` — **115KB**（全局参数定义）
- `mynetwork.cpp` — **107KB**（网络通信）
- `mylanguage.cpp` — **96KB**（多语言支持）
- `ipcclassparams.ui` — **412KB**（IPC分类参数界面）

---

## 关键技术栈

- **UI框架**: Qt Widgets（.ui 文件 + 代码混合方式）
- **构建系统**: qmake（.pro 文件）
- **目标平台**: 主要 Linux ARM 嵌入式，辅助 Windows 调试
- **通信方式**: 串口 `/dev/tty*`、TCP/UDP、MQTT、HTTP
- **核心算法**: SVM 支持向量机分类、多光谱图像处理
- **国际化**: 23种语言的翻译文件

---

## 学习路线建议

1. **先从 `main.cpp` 看起** — 理解程序启动和初始化流程
2. **再看 `zksort.h/cpp`** — 理解主窗口结构和全局事件处理
3. **接着看 `middlewidgetmanager.h/cpp`** — 理解页面切换/管理机制（这是UI的核心调度中心）
4. **然后看 `global/`** — 理解全局基础设施（runtime, globalflow, globalparams）
5. **再看 `bus/`** — 理解通信层（串口+网络）
6. **最后深入各业务模块** — 按 `mainpage` → `factoryset` → `engineerset` → `arithprofilewidget` 顺序
