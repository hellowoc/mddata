# mysortwidget/ 自定义控件库详解

---

## 一、目录定位

`mysortwidget/` 是整个项目的**公共 UI 控件库**，包含 30+ 个自定义控件、对话框和工具类。所有其他模块（mainpage、factoryset、engineerset、systeminfo）的页面都由这些控件组合而成。

```
mysortwidget/ 的三大功能:
  ① 自定义可视控件 — 波形图、HSV 色环、开关、按钮、标签、列表、算法复选框
  ② 对话框 — 输入/消息/下载/重命名/设置/SVM 配置/日期设置
  ③ 基础设施 — 页面基类、输入法、加密管理、全局配置
```

---

## 二、基类体系

整个项目的所有页面和控件都建立在这套基类之上：

```
QObject
  └── QWidget
        └── BaseUi               ← 语言切换 + QSS 背景渲染
              ├── basewidget     ← 页面管理器基类（视/层/通道/组）
              │     └── MySvmSampleClass  ← SVM 采样管理页
              │
              └── BaseArithWidget ← 算法模式选择基类（QButtonGroup）
```

### BaseUi — 所有 UI 的公共根

```cpp
class BaseUi : public QWidget {
    virtual void retranslateUiWidget() = 0;   // 语言切换回调（纯虚）
    virtual void showPage(bool) = 0;          // 页面显示控制（纯虚）
protected:
    void changeEvent(QEvent*);    // 拦截 LanguageChange → 调 retranslate
    void paintEvent(QPaintEvent*); // 开启 QSS 背景渲染
};
```

### basewidget — 大多数设置页面的基类

```cpp
class basewidget : public BaseUi, public AbstractInterface {
    QPushButton *m_pViewBtn[8];     // 视按钮（前主/前辅/后主/后辅 × 上下层）
    QPushButton *m_pLayerBtn[2];    // 层按钮（上层/下层）
    UiGroupWidget *m_pGroupWidget;  // 识别组选择
    SetEditValueWidget *m_pChuteWidget; // 通道选择

    virtual void OnViewChange(int view);     // 视切换 → 子类重写
    virtual void OnGroupChange(int group);   // 组切换 → 子类重写
    virtual void OnChuteChange();            // 通道切换 → 子类重写
    virtual void OnLayerChange();            // 层切换 → 子类重写
    void receiveMsg(int msg1, int, int);     // 消息接收
};
```

### AbstractInterface — 页面间消息通信协议

```cpp
class AbstractInterface {
    virtual void receiveMsg(int msg1, int msg2 = 0, int msg3 = 0) = 0;
};
```

这是一个极简的消息总线。任何继承此接口的页面都能互发消息，最常用的消息是 `MSG_DATA_CHANGED`（参数已被修改，需要通知父页面刷新）。

---

## 三、自定义可视控件

### 3.1 MyWaveWidget — 波形/示波器显示（最复杂的绘制控件）

```
QWidget → MyWaveWidget (34KB cpp, 1100+ 行)
```

**作用**：实时显示传感器像素数据的波形曲线（R/G/B/红外/色相/亮度通道），是分选机最核心的显示控件。

**五种绘制模式**：

| 模式 | 用途 |
|------|------|
| `Divide_PIXEL` | 像元划分模式——显示通道边界和分隔线 |
| `Gain_Page` | 增益显示 |
| `BackGround_Page` | 背景阈值显示（水平参考线） |
| `IR_BadPoints` | 红外坏点显示（标记坏点位置和修正值） |
| `IPC_EJECT` | IPC 相机喷阀位置显示（喷气起止位置） |

**绘制流程**（`paintEvent` 中）：

```
paintAxis()         → 画坐标系(X/Y轴、刻度、网格线)
drawWanted()        → 画数据波形（按颜色通道分别绘制）
drawDividePix()     → 画像元分隔线（Divide_PIXEL 模式）
drawIpcEject()      → 画喷阀区域标线（IPC_EJECT 模式）
drawBackgroundValue() → 画背景参考线
drawIrBadPoint()    → 画红外坏点修正标记
```

这个控件是纯手绘（画坐标系 → 画数据曲线 → 画辅助标记），没有子控件。

---

### 3.2 SwitchBtn — iOS 风格滑动开关

```
QWidget → SwitchBtn
```

**作用**：纯绘制的 toggle 开关，带动画效果。

**绘制**：`paintEvent` 中先画圆角矩形轨道，再画圆形滑块。滑块位置根据 `m_bChecked` 决定左右，状态切换时通过 QTimer 以 2px/5ms 的速度滑动滑块。

**信号**：`toggled(bool checked)`。

### MySwitchWidget — 开关 + 标签组合

```
QWidget → MySwitchWidget（内含 SwitchBtn + QLabel）
```

通过 `.ui` 文件组合，对外暴露 `setToggle()` / `setText()` / `switchToggledSignal`。

---

### 3.3 HsvColorCircle — HSV 色环选择器

```
QWidget → HsvColorCircle (11KB cpp, 360 行)
```

**作用**：双拇指 HSV 颜色选择器。用户在外环上拖拽选择色相（H），在内部矩形中拖拽选择饱和度（S）和明度（V）。

**交互流程**：

```
mousePressEvent → 判断点击位置（环上？矩形中？）
mouseMoveEvent  → 拖动更新 H/S/V 值
mouseReleaseEvent → 确认选择 → emit colorChanged()
```

**支持双阈值**：H/S/V 各有 low/high 两个值，用于设置分选算法的上下限。

---

### 3.4 MyArithUi — 单个算法使能开关

```
QWidget → MyArithUi (33KB cpp)
```

**作用**：一个复选框控件，代表一种分选算法（灰度、SVM、HSV、霉斑...共 40+ 种）。每个 `MyArithUi` 实例绑定一种具体算法。

**核心方法**：

```
showWidget()     → 设置算法名称（从 g_myLan() 取多语言文本）
updateWidget()   → 读取全局配置，更新复选框打勾状态
on_checkBox_clicked() → 用户点击 → 修改 struCnfp 结构体
                      → 调 myFlow.materialArithEnableCopy() 复制到关联通道
                      → 调 myFlow.materialParamsCopySendAssemble() 下发到硬件
```

**40+ 种算法**通过一个巨大的 switch 语句处理，每个 case 读写不同的结构体字段。

---

### 3.5 按钮类控件

```
QPushButton → myPushButton   (权限色按钮：客户绿/工程师黄/工厂粉)
QPushButton → myToolButton   (双位图按钮：正常/按下各一张图，叠加文字)
QLabel      → myLabel        (只加了全局字体设置)
QLineEdit   → myLineEdit     (点击弹出输入框/软键盘)
QListWidget → mylistwidget   (文件浏览器，支持拖拽操作)
```

**myLineEdit 的设计模式**：点击时不是弹出原生键盘，而是弹出本项目自定义的 `MyInputDialog`（数字输入）或 `myInputMethod`（文本输入）。这是嵌入式设备无物理键盘的标准做法。

---

### 3.6 组合控件

**UserSenseSetPage** — 灵敏度设置页：

```
QWidget → UserSenseSetPage（内含 3 个 UiVerticalProgressExt 柱状图）
```

用户可以拖动柱状图来调整各算法的灵敏度。每次调整 → 修改 `struCnfp` → 下发到硬件。

**MylightControl** — 单路灯光控制：

```
QWidget → MylightControl（内含 QLabel + myLineEdit + QCheckBox）
```

绑定到一块灯板的某个端口，设置亮度和开关。

---

## 四、对话框群

### 4.1 MyInputDialog — 数字键盘对话框

```
QDialog → MyInputDialog (12KB cpp)
```

**作用**：嵌入式触摸屏上的数字键盘。支持整数、浮点数、百分比、密码模式。

**关键设计 — QSignalMapper 映射**：10 个数字按钮（0-9）通过 QSignalMapper 统一映射到 `OnBtnClick(int digit)`，避免为每个按钮写重复的槽函数。

**数据流**：

```
用户点击数字 → OnBtnClick(digit) → 拼接到内部缓冲区
点击确认     → on_confirmBtn_clicked()
              → 写入绑定变量的内存地址
              → 更新 QLineEdit 显示
              → receiveMsg(MSG_DATA_CHANGED) 通知父页面
```

---

### 4.2 消息/提示对话框

| 对话框 | 基类 | 用途 |
|--------|------|------|
| `myMessageBox` | QDialog | 确认/取消询问框，带倒计时自动关闭 |
| `myLongMessageBox` | QDialog | 长文本消息框（用 QTextBrowser 显示） |
| `MessageDialog` | QDialog | 简单消息弹窗（1-2个按钮） |
| `myInfoWidget` | QDialog | 全屏居中提示（"通信中..."） |

---

### 4.3 RenameProfileDialog — 重命名方案

```
QDialog → RenameProfileDialog（QLineEdit + QDialogButtonBox）
```

极简对话框——一个输入框加确定取消按钮。

---

### 4.4 SetSvmUseDialog — SVM 模型分配

```
QDialog → SetSvmUseDialog (7.6KB ui)
```

**作用**：用 8×5 的复选框矩阵来配置哪个视的哪个识别组使用哪个 SVM 模型。

```
         组1   组2   组3   组4   组5
前主视   [✓]   [ ]   [✓]   [ ]   [ ]
前辅视   [ ]   [✓]   [ ]   [ ]   [ ]
...
```

确定后调 `myFlow.setSvmParas()` → 下发到硬件。

---

### 4.5 DownloadDialog — 固件下载进度

```
QDialog → DownloadDialog
```

**作用**：显示远程升级的下载进度、速度和剩余时间。

使用 `httpDownloadManager` 进行 HTTP 下载，通过信号槽接收进度通知。下载完成后弹出安装确认，执行 shell 脚本完成升级。

**关键连接**：

```
connect(httpDownloadManager, SIGNAL(sigDownloadProcess(bytes, total)),
        this,                SLOT(onDownloadProcess(bytes, total)));
connect(httpDownloadManager, SIGNAL(sigDownloadFinished(status)),
        this,                SLOT(onReplyFinished(status)));
connect(httpDownloadManager, SIGNAL(sigDownloadError()),
        this,                SLOT(onDownloadError()));
```

---

### 4.6 SetEditValueWidget — 加减值控件

```
QWidget → SetEditValueWidget（实现 AbstractInterface）
```

**作用**：通用的 +/- 增减控件，用于修整数或浮点数参数。

**两个版本的对比**：

| | 版本1 | 版本2 |
|---|-------|-------|
| 按钮重复方式 | `setAutoRepeat`（500ms延迟 + 100ms间隔） | QTimer（100ms 周期） |
| 发送时机 | 每次点击都发送 | 松手时才发送 |
| float 步长 | 0.1 | `1.0 / bitCount` |

版本2 的"松手才发送"设计是为了减少通信量——用户在连续加值时，中间值不需要发给硬件，只需要发最终值。

---

## 五、输入法系统

### 5.1 myInputMethod — 完整屏幕键盘

```
QDialog → myInputMethod (21KB cpp)
```

**作用**：触摸屏上的全功能软键盘，支持英文、中文拼音、数字、特殊字符。

**三种键盘模式**：

| 模式 | 用途 |
|------|------|
| `KB_NORM` | 正常模式：支持拼音中输入 |
| `KB_DC` | 加密码模式：隐藏拼音区 |
| `KB_PD` | 密码模式：隐藏拼音区 + 密码回显 |

**拼音输入流程**：

```
用户按下字母键 → sendChar('z') → 拼入拼音缓冲区 "z"
                  → transPy("z") → 查 PY_IdxTable
                  → 找到: "za 杂砸匝咂..."
                  → showPage() 显示 8 个候选汉字
用户点选汉字 → chineseSelectFont() → 汉字插入文本框
```

**键盘布局**：37 个输入按钮通过 `QSignalMapper` 统一映射到 `sendChar(int)`。

---

### 5.2 mypinyin.h — 拼音字库

```cpp
// 34KB 静态数据
typedef struct {
    const char *PY;   // 拼音: "zhong"
    const char *MB;   // 汉字: "中盅忠钟衷终种肿重仲众..."
} PY_IDX;

PY_IDX PY_IdxTable[] = {
    {"a",   "阿啊锕呵吖腌..."},
    {"ai",  "爱埃艾碍哀矮挨..."},
    // ... 440 个拼音条目 ...
};
```

约 440 个拼音音节，每个对应一组候选汉字。

---

## 六、加密管理系统

### myDelayCode（47KB cpp）— 核心加密逻辑

```
QObject → myDelayCode
```

**不是 UI 类，是纯逻辑对象。** 管理设备的软件授权和加密保护。

**关键数据在 EEPROM 中的布局**：

```
EEPROM 地址:
  0x0000  BASE_ADDR_DELAY          主加密区（加密使能标记、密码、到期时间）
  0x0100  BASE_30DAY_USE_ADDR      30 天临时授权区
  0x0200  BASE_30DAY_FPGA_CODE_ERR FPGA 密码错误计数
  0x0300  BASE_UI_LANGUAGE_SET     界面语言设置
```

**加密校验流程**：

```
delayCodeCheck()
  → 从 EEPROM 读取加密信息
  → 用 time_encrypt 库解密
  → 验证系统时间（防 RTC 篡改：和上次存储的时间比较，不能倒退）
  → 检查是否到期
  → 启动 ./zktimer 守护进程
  → 返回: 0=有效, 1=过期
```

**三个 UI 对话框**：

| 对话框 | 场景 |
|--------|------|
| `myDccrypt` | 输入永久解密密码 |
| `myEncrypt` | 制造商加密（设置到期时间） |
| `myDccrypt30Day` | 30 天临时解密 + 再输永久密码 |

**30 天临时授权**：设备到期后用户可以申请 30 天临时延期，期间必须联系厂商获取永久密码。

---

## 七、全局配置工具

### g_Config — 自适应屏幕的尺寸管理

```cpp
template<class T> class CSingletonBase { ... };
class g_Config : public CSingletonBase<g_Config>
```

**单例模式**。根据 `LCD_WIDTH`（1024 或 640）自动计算所有按钮和字体的标准尺寸：

```cpp
#ifdef LCD_WIDTH == 1024
    #define BTN_HEIGHT 60
    #define BTN_WIDTH 160
    #define DEFAULT_FONT_SIZE 24
#else
    #define BTN_HEIGHT 40
    #define BTN_WIDTH 100
    #define DEFAULT_FONT_SIZE 16
#endif
```

---

## 八、跨模块串联机制

### 8.1 消息总线 — AbstractInterface

几乎所有的设置类控件都实现了 `AbstractInterface::receiveMsg()`。数据流是这样的：

```
SetEditValueWidget (用户调了参数)
    │
    ├─ sendParams() → 写指针地址 → 改全局结构体
    └─ parent->receiveMsg(MSG_DATA_CHANGED, msgPar)
            │
            ▼
        basewidget (中间层页面)
            │
            ├─ 刷新自身显示
            └─ 继续向上传 receiveMsg(MSG_DATA_CHANGED)
                    ↓
                顶层页面 → 可能触发 GlobalFlow 下发到硬件
```

### 8.2 全局单例的普遍使用

所有控件都直接引用全局单例，不通过参数传递：

```
g_myLan()       → 获取当前语言的文字
struCnfg        → 读写全局配置
struGsh         → 读写运行状态
myFlow          → 下发参数到硬件
g_infoWidget()  → 显示提示信息
g_Runtime()     → 文件/系统操作
g_Config        → 获取按钮/字体尺寸
```

这是嵌入式 C++ 项目的典型风格——全局变量比依赖注入更直接，减少参数传递的复杂度。

### 8.3 语言切换的统一处理

```
g_myLan().ChangeLanguage()
    → QEvent::LanguageChange 事件
    → BaseUi::changeEvent() 拦截
    → retranslateUiWidget() 被调用
    → 每个控件重新从 g_myLan() 取文字 → setText()
```

所有自定义控件都在 `retranslateUiWidget()` 中刷新自己的文字，保证语言切换后整个界面立即更新。

---

## 九、文件总览

| 文件 | 行数 | 继承 | 用途 |
|------|------|------|------|
| baseui | 30 | QWidget | UI 根基类（语言+QSS） |
| basewidget | 270 | BaseUi | 页面管理器基类（视/层/通道） |
| basearithwidget | 30 | BaseUi | 算法模式选择基类 |
| mywavewidget | 1100 | QWidget | 实时波形显示 |
| hsvcolorcircle | 360 | QWidget | HSV 色环双值选择器 |
| switchbtn | 177 | QWidget | 滑动开关（手绘+动画） |
| myswitchwidget | 37 | QWidget | 开关+标签组合 |
| mypushbutton | 373 | QPushButton | 权限色按钮 |
| mytoolbutton | 92 | QPushButton | 双位图+文字按钮 |
| mylineedit | 230 | QLineEdit | 点击弹键盘编辑框 |
| mylabel | 18 | QLabel | 全局字体标签 |
| mylistwidget | 85 | QListWidget | 文件列表浏览器 |
| myarithui | 793 | QWidget | 40+ 种算法使能复选框 |
| mylightcontrol | 58 | QWidget | 单路灯光控制面板 |
| usersensesetpage | 200 | QWidget | 灵敏度柱状图页 |
| myinputdialog | 430 | QDialog | 数字键盘 |
| mymessagebox | 150 | QDialog | 消息/确认弹窗 |
| messagedialog | 80 | QDialog | 简单弹窗 |
| downloaddialog | 370 | QDialog | 固件下载进度 |
| renameprofiledialog | 30 | QDialog | 方案重命名 |
| setsvmusedialog | 280 | QDialog | SVM 模型分配矩阵 |
| mysetdatetime | 100 | QDialog | 系统日期时间设置 |
| seteditvaluewidget | 150 | QWidget | +/- 增减控件 |
| mywid | 700 | (多个类) | 通用控件集合 |
| myinputmethod | 730 | QDialog | 全功能屏幕软键盘 |
| mypinyin | — | (数据) | 拼音-汉字对照表(34KB) |
| mydelaycode | 1500 | QObject | 加密/授权管理核心 |
| globalconfig | 70 | 单例 | 屏幕尺寸适配配置 |
