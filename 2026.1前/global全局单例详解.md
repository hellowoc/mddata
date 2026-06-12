# global.cpp 全局单例详解

## 文件结构

```cpp
// 6 个全局结构体实例（不是单例函数，是直接定义的全局变量）
struct struCnfEngineer struCnfe;    // 工程师配置
struct struCnfGlobal struCnfg;      // 全局配置
struct struCnfProfile struCnfp;     // 方案参数
struct struCnfFileStatus struCnfs;  // 文件状态
struct struShare struGsh;           // 共享状态（线程间通信）
struct struIpcShareInfo struIpcShare; // IPC 共享信息

// 6 个单例函数（Meyers 单例模式）
MiddleWidgetManager &g_MainManager()  → 页面管理器
Runtime &g_Runtime()                  → OS 抽象层
MyInputDialog &g_InputDialog()        → 输入对话框
myInfoWidget &g_infoWidget()          → 信息提示框
myInputMethod &g_inputmethod()        → 自定义输入法
MyLanguage &g_myLan()                 → 多语言管理
```

---

## 逐个解析

### ① `g_MainManager()` — 页面管理器

```cpp
MiddleWidgetManager &g_MainManager()
{
    static MiddleWidgetManager manager;   // 继承 QObject
    return manager;
}
```

**职责：管理所有业务页面的切换。**

- 持有 QStackedWidget 的控制权
- `ShowWidget(pageId)` 切换到指定页面
- 维护页面索引表（pageId → stackedWidget 中的 index）
- 页面间消息转发

ZKSort 构造函数中调用 `g_MainManager().setStackedWidget(...)` 把页面栈交给它管。

---

### ② `g_Runtime()` — OS 抽象层

```cpp
Runtime &g_Runtime()
{
    static Runtime runtime;   // 继承 QObject
    return runtime;
}
```

**职责：封装所有跟 Linux 操作系统交互的操作。**

- 文件操作（删/拷/创建目录/计算MD5）
- 系统命令（执行 shell、获取 USB 路径）
- 网络信息（获取 IP、检查 NFS 挂载）
- 配置持久化（save/saveSetting）

（详见 Runtime 详解文档）

---

### ③ `g_InputDialog()` — 输入对话框

```cpp
MyInputDialog &g_InputDialog()
{
    static MyInputDialog inputdialog;   // 继承 QDialog
    return inputdialog;
}
```

**职责：弹出输入框让用户输入文字。**

场景：工程师需要输入密码、修改参数值、命名新方案。

它是一个全局复用的弹窗——不是每次弹出都 new 一个，而是共用一个实例。

---

### ④ `g_infoWidget()` — 信息提示框

```cpp
myInfoWidget &g_infoWidget()
{
    static myInfoWidget infoWidget;   // 继承 QWidget / QDialog
    return infoWidget;
}
```

**职责：显示全屏居中的提示文字。**

```cpp
// main.cpp 中的使用
g_infoWidget().move((screenW - width)/2, (screenH - height)/2);
g_infoWidget().setLabelText("通信中，请稍候...");
g_infoWidget().delayShow();     // 显示提示
// ... 做初始化 ...
g_infoWidget().hide();          // 隐藏提示
```

这就是开机时屏幕上显示"通信中，请稍候..."的那个框。

---

### ⑤ `g_inputmethod()` — 自定义输入法键盘

```cpp
myInputMethod &g_inputmethod()
{
    static myInputMethod inputMethod;   // 继承 QDialog（一个软键盘）
    return inputMethod;
}
```

**职责：提供触摸屏上的软键盘。**

嵌入式设备没有物理键盘 → 需要屏幕上的虚拟键盘输入。用户在输入框聚焦时弹出软键盘，点选拼音/字母/数字。这是一个完整的自定义输入法。

---

### ⑥ `g_myLan()` — 多语言管理

```cpp
MyLanguage &g_myLan()
{
    static MyLanguage mylan;   // 继承 QObject
    return mylan;
}
```

**职责：管理 23 种语言的翻译切换。**

```cpp
g_myLan().msg_communicating   // 返回当前语言的"通信中"文字
g_myLan().msg_wiping           // 返回当前语言的"擦拭中"文字
g_myLan().ChangeLanguage()     // 切换语言 → 触发所有页面刷新
```

`MyLanguage` 对象里存了 23 种语言的翻译表，切换语言时所有文字自动刷新。

---

## 为什么用单例 — 汇总

| 单例 | 为什么是单例 |
|------|------------|
| `g_MainManager` | 页面栈只有一个，所有页面切换走同一入口 |
| `g_Runtime` | 设备事实只有一份（权限、层数、USB路径） |
| `g_InputDialog` | 全程序只有一个输入框弹窗，复用 |
| `g_infoWidget` | 全程序只有一个提示框，居中复用 |
| `g_inputmethod` | 全程序只有一个软键盘 |
| `g_myLan` | 全程序只使用一种语言，语言表一份即可 |

---

## 6 个全局结构体（非单例，但也是全局变量）

```cpp
struCnfg    // 全局配置（屏幕尺寸、板卡数量、控制模式...）
struCnfe    // 工程师配置（背景、光源、采集参数...）
struCnfp    // 方案参数（算法配置、灵敏度...）
struCnfs    // 文件状态（哪些文件已被修改...）
struGsh     // 共享状态（喂料状态、阀门状态、报警标志、IPC状态...）
struIpcShare // IPC 共享信息
```

它们不是单例函数，而是**直接定义的全局变量**（`struct xxx var`）。用 `_t_` 前缀的版本是临时缓冲区：

```cpp
struct struCnfGlobal struCnfg, _t_struCnfg;
//                     ↑ 正式配置   ↑ 临时副本（编辑时先改 _t_ 版本，确认后再拷贝到正式版本）
```

---

## 为什么分两种：单例函数 vs 全局变量

```
单例函数 (g_xxx):
  → 对象有构造函数、需要初始化逻辑
  → 继承 QObject（有信号槽）
  → 函数是 Meyers 单例模式
  → 如: g_MainManager(), g_Runtime(), g_myLan()

全局变量 (struXxx):
  → 纯数据 struct，没有构造逻辑
  → 不继承 QObject，没有信号槽
  → 直接用 C 风格全局变量
  → 如: struCnfg, struGsh, struIpcShare
```

---

## 一句话

`global.cpp` 是这个程序的**"共享数据中枢"**——6 个单例函数提供全局服务（页面切换、OS 操作、输入输出、语言），6 个全局结构体存储全局状态（配置、运行状态、IPC 信息）。任何 `.cpp` 文件 `#include "global.h"` 后都能直接访问它们。
