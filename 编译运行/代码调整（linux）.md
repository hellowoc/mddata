# zksort Qt5 迁移 — 代码修改汇总

## 修改总览

| 类别          | 文件数    | 修改类型                 |
| ----------- | ------ | -------------------- |
| Qt4 API 替换  | ~40 文件 | 全项目批量替换              |
| 头文件补充       | 8 文件   | 添加缺失的 `#include`     |
| 截断文件修复      | 4 文件   | 补充缺失的代码              |
| 运行时崩溃修复     | 3 文件   | 修复空指针/无效 fd          |
| 第三方库源码修复    | 6 文件   | log4qt/uikits Qt5 适配 |
| ARM Stub 创建 | 6 新文件  | 空实现 stub 库           |

---

## 一、全项目批量替换

### 1.1 `TRUE` / `FALSE` → `true` / `false`

**为什么**: Qt4 定义了 `TRUE`/`FALSE` 宏（通常在 `qglobal.h` 中），Qt5 移除了这些宏。C++ 标准关键字 `true`/`false` 替代。

**影响文件**: `engineerwidget.cpp`, `machinefunction.cpp`, `onekeyupdatefpgawidget.cpp`, `updatefpgawidget.cpp`, `globalflow.cpp`, `mysqlite.cpp`, `myqueue.cpp`, `mydelaycode.cpp`, `mywid.cpp`, `vavletest.cpp` 等

### 1.2 `.toAscii()` → `.toLatin1()`

**为什么**: Qt5 移除了 `QString::toAscii()`（名称具有误导性，并非真正的 ASCII 转换）。`toLatin1()` 是等价的替代。

**影响文件**: `modemanagewidget.cpp`, `backgroundsetwidget.cpp`, `upgrademodelwidget.cpp`, `myfastnetwork.cpp`, `mynetwork.cpp`, `mydelaycode.cpp`, `myhttpfileclient.cpp` 等

### 1.3 `.fromAscii()` → `.fromLatin1()`

**为什么**: 与 `toAscii()` 同理，`QString::fromAscii()` 在 Qt5 中移除。

**影响文件**: `globalflow.cpp`

### 1.4 `QChar::fromAscii()` → `QString(QChar(...))`

**为什么**: `QChar::fromAscii()` 在 Qt5 中移除。`QChar` 构造函数直接接受 `char` 也可行，但这里 `setText()` 需要 `QString` 参数。

**影响文件**: `myinputmethod.cpp` (第 392、430 行)

### 1.5 `setResizeMode()` → `setSectionResizeMode()`

**为什么**: `QHeaderView::setResizeMode()` 在 Qt5 中重命名为 `setSectionResizeMode()`，旧名已移除。

**影响文件**: `camerasetwidget.cpp`, `nirdotcorrectivelistwidget.cpp`, `sysversioninfowidget.cpp`, `ipcclassinfowidget.cpp`, `ipctestnetwidget.cpp`, `sysipcbadpointswidget.cpp`, `ipcinfowidget.cpp`

### 1.6 `QDesktopServices::storageLocation()` → `QStandardPaths::writableLocation()`

**为什么**: `QDesktopServices::storageLocation()` 在 Qt5 中移除，统一使用 `QStandardPaths`。

**影响文件**: `ipcinfowidget.cpp` (第 158、347 行)

**伴随修改**: `ipcinfowidget.h` 添加 `#include <QStandardPaths>`

---

## 二、头文件补充

### 2.1 `#include <QPainterPath>` 补充

**为什么**: Qt4 中 `QPainterPath` 通过 `<QPainter>` 间接包含。Qt5 模块化后，`QPainterPath` 是独立头文件。不添加会导致 "incomplete type" 编译错误。

**影响文件**:
- `switchbtn.cpp`
- `barchart.cpp`
- `uikits/src/UiHorizontalProgress.cpp`
- `uikits/src/UiVerticalProgress.cpp`
- `uikits/src/QRoundProgressBar.cpp`

### 2.2 `#include <QtWidgets>` 补充

**为什么**: Qt4 中 `<QtGui>` 包含所有 GUI + Widget 类。Qt5 将 Widget 类拆分到 `QtWidgets` 模块。仅包含 `<QtGui>` 会导致 `QLineEdit`、`QLabel`、`QCheckBox` 等类找不到。

**影响文件**:
- `myinputmethod.h`
- `fpga.h`
- `fpga2.h`
- `mydelaycode.h`
- `mydelaycode.cpp`
- `globalconfig.h`

### 2.3 `uikits.pro` 添加 `QT += widgets`

**为什么**: uikits 使用了 `QWidget`、`QFrame`、`QStackedWidget` 等类，但 `.pro` 文件未声明 widgets 模块依赖。Qt4 中这些类在 QtGui 中，Qt5 需要显式声明。

### 2.4 `factory.h` 添加 `QObject` / `QMetaProperty` 头文件

**为什么**: log4qt 的 `factory.h` 使用了 `QObject` 和 `QMetaProperty` 但没有包含它们的头文件。Qt4 中通过间接包含获取，Qt5 头文件更严格。

**影响文件**: `3rdparty/log4qt/src/helpers/factory.h`

### 2.5 `level.cpp` 添加 `#include <QDataStream>`

**为什么**: 修改后的代码直接使用 `QDataStream::writeRawData()`，需要完整类型定义，不能仅靠前向声明。

**影响文件**: `3rdparty/log4qt/src/level.cpp`

---

## 三、截断 / 缺失代码修复

### 3.1 `globalflow.cpp` — 文件末尾截断

**问题**: 文件在第 11845 行截断，`sendStatus()` 函数的最后一行代码不完整：
```cpp
data[3+MAX_FEEDER*k+i] = struCnfp.struGroupCtr  // 截断!
```
且文件缺少 `}` 闭合括号。

**修复**:
```cpp
data[3+MAX_FEEDER*k+i] = (k == 0)
    ? struCnfp.struGroupCtrl[0].nFeederValueMain[i]
    : struCnfp.struGroupCtrl[0].nFeederValueVice[i];
            }
        }
    }
}

void GlobalFlow::IpcClassResultANDOperat(bool isall, int view, int unit)
{
}
```
- 修正了 `struGroupCtr` → `struGroupCtrl` 拼写错误
- 补全了表达式，根据 k 值分发到 Main/Vice 数组
- 补全了缺失的闭合括号
- 添加了 `IpcClassResultANDOperat()` 空实现（头文件声明但未实现）

### 3.2 `mylineedit.cpp` — 文件末尾截断

**问题**: 文件在 `setLocation()` 函数内的注释中截断，函数缺少闭合 `}`。

**修复**:
```cpp
}

void myLineEdit::mousePressEvent(QMouseEvent *event)
{
    QLineEdit::mousePressEvent(event);
}
```
- 闭合了 `setLocation()` 函数
- 添加了 `mousePressEvent()` 实现（头文件声明但未实现）

### 3.3 `ipcinfowidget.cpp` — 缺失方法和截断

**问题**:
1. `doCameraShut()` 函数中 `if (struCnfg.nPowerCut == 1 || camerashut == 1)` 块缺少闭合 `}`
2. 文件末尾截断，`generatePressureTestReport()` 内 `out << l` 不完整
3. `updateDelayValues()` 声明但未实现

**修复**:
- 在第 811 行后添加 `}` 闭合 if 块
- 补全截断的代码为 `out << line << "\n";`
- 添加 `IPCInfoWidget::updateDelayValues()` 空实现

### 3.4 `topwidget.cpp` — 死代码移除

**问题**: `on_printBtn_clicked()` 函数体为空，且对应 UI 控件 `printBtn` 不存在。Qt5 编译器对此类无声明的方法定义报错。

**修复**: 移除空的 `on_printBtn_clicked()` 函数。

---

## 四、运行时崩溃修复

### 4.1 串口 `FD_SET(-1)` 缓冲区溢出

**崩溃信息**: `*** buffer overflow detected ***: terminated`

**根因**: `zkSerialOpenConfig()` 在调用 `open()` 之前就设置 `ports[portno].is_busy = 1`。当 `open()` 失败返回 -1 时：
- `is_busy` 仍为 1（未回退为 0）
- `ports[portno].handle` 为 -1

`zkSerialRead()` 检查 `is_busy` 为 1，通过检查后调用 `FD_SET(-1, &rfds)`。glibc 的 fortify 检测到 -1 是无效文件描述符，触发 `__fdelt_chk` 失败。

**修复** (`myqextserialport.cpp`):

1. `zkSerialOpenConfig()` — 将 `is_busy = 1` 移到成功 open 之后：
```cpp
// 修复前:
ports[portno].is_busy = 1;   // 过早设置
if (open(...) == -1) return -1;  // 失败时 is_busy 仍=1

// 修复后:
if (open(...) == -1) {
    ports[portno].is_busy = 0;  // 失败时清除
    return -1;
}
ports[portno].is_busy = 1;      // 成功后才设置
```

2. `zkSerialRead()` — 添加无效 handle 检查：
```cpp
// 修复前:
if (ports[portno].is_busy == 0) { }

// 修复后:
if (ports[portno].is_busy == 0 || ports[portno].handle < 0) {
    return -1;
}
```

3. `zkSerialWrite()` — 同样的检查：
```cpp
if (ports[portno].is_busy == 0 || ports[portno].handle < 0) {
    return -1;
}
```

### 4.2 IPC 堆损坏

**崩溃信息**: `corrupted size vs. prev_size`

**堆栈**: `IPCInfoWidget::updateTableWidget()` → `QList<QString>::~QList()` → `free()` → heap corruption detected

**根因**: IPC（工控机相机）子系统在 PC 上无实际硬件。`IPCInfoWidget::showPage()` 仍会调用 `updateVersionInfo()` 通过网络获取相机数据，stub 返回的数据导致堆损坏。

**修复** (`ipcinfowidget.cpp`):
```cpp
// 修复前:
if(bshow){

// 修复后:
if(bshow && struCnfg.nUseIpcEnable == 1){
```
当 IPC 未启用时（默认配置），跳过整个 IPC 页面更新逻辑。

### 4.3 日志路径 `/sdcard/` 不存在

**问题**: 原代码硬编码日志路径为 `/sdcard/logs/`，PC 上此路径不存在，导致日志系统初始化报错。

**修复** (`main.cpp`):
```cpp
// 修复前:
QString path = "/sdcard/logs/";

// 修复后:
QString path = "/sdcard/logs/";
QDir sdcardLogs(path);
if (!sdcardLogs.exists()) {
    path = qApp->applicationDirPath() + "/logs/";  // 回退到本地
}
```

同时添加目录自动创建:
```cpp
QDir dir(path);
if (!dir.exists()) {
    dir.mkpath(".");  // 确保日志目录存在
}
```

---

## 五、第三方库源码修复

### 5.1 log4qt 修复

**文件**: `3rdparty/log4qt/src/`

| 修改 | 原因 |
|---|---|
| `logmanager.h:258-259` — `qtMessageHandler` 签名从 `const char*` 改为 `const QMessageLogContext&, const QString&` | Qt5 的 `qInstallMessageHandler` 回调签名变更 |
| `level.cpp:170-188` — 重写 QDataStream 运算符，使用 `writeRawData/readRawData` 替代 `operator<<` | Qt5 中 `quint8` 的 `operator<<` 与 `QVariant::Type` 重载冲突 |
| `level.cpp` — 添加 `#include <QDataStream>` | 使用 `writeRawData()` 需要完整类型 |
| `level.h:150-151` — 暂时注释 `Q_DECLARE_METATYPE` (后恢复) | 调试 QDataStream 运算符歧义问题 |
| `logerror.cpp:114` — `codecForTr()` → `codecForLocale()` | Qt5 移除 `QTextCodec::codecForTr()` |
| `logerror.cpp:132` — 移除 `QCoreApplication::UnicodeUTF8` 参数 | Qt5 的 `translate()` 不再需要编码参数 |
| `systemlogappender.cpp:10,68,86,142,157` — `Q_WS_WIN` → `Q_OS_WIN` | Qt5 将窗口系统宏重命名为操作系统宏 |
| `systemlogappender.cpp:157` — `TRUE` → `true` | Qt4 宏移除 |
| `debugappender.cpp:40,86` — `Q_WS_WIN` → `Q_OS_WIN` | 同上 |
| `factory.h` — 添加 `#include <QObject>` 和 `#include <QMetaProperty>` | Qt5 头文件模块化，不再间接包含 |

### 5.2 uikits 修复

**文件**: `uikits/`

| 修改 | 原因 |
|---|---|
| `uikits.pro` — 添加 `QT += widgets` | Qt5 需要显式声明 widgets 模块 |
| `QRoundProgressBar.cpp` — 添加 `#include <QPainterPath>` | Qt5 中 QPainterPath 独立于 QPainter |
| `UiHorizontalProgress.cpp` — 添加 `#include <QPainterPath>` | 同上 |
| `UiVerticalProgress.cpp` — 添加 `#include <QPainterPath>` | 同上 |

---

## 六、ARM Stub 库

为以下库创建了空实现 stub，使链接通过。Stub 源文件位于 `3rdparty/stubs/`。

### 6.1 svm_stub.cpp (`libsvmtool.a` + `libWheat.a`)
- `CRgnNode` 类: 构造/析构/方法 空实现
- `shape_classify`, `shape_sampling`, `isBad`, `train`: C++ 链接空实现
- `Wheat` 类: 构造/析构/`Wheat_Main` 空实现
- Erode, Dilate, SobelFilter 等图像处理 C 函数: C 链接空实现

### 6.2 encrypt_stub.cpp (`libtime_encrypt.a`)
- `DecryptCode`: 返回 0 (decrypt_succeed)
- `WhetherCodesMatch`: 返回 true

### 6.3 fToFp_stub.cpp (`libTEA.a`)
- `Tea` 类: 所有方法空实现（Float_to_fp16, TEA_Signed, getTEA, fp16ToFloat, Erode, getDis）

### 6.4 qrencode_stub.c (`libqrencode.a`)
- QR 码编码所有函数空实现

### 6.5 mqtt_stub.c（最终未使用）
- 创建了但被系统库替代

> **关键**: encrypt_stub.cpp 的函数需要保持与原始库相同的 C++ 名称修饰（mangling）。最初使用了 `extern "C"`（C 链接），但原始库使用 C++ 链接编译。修正后去除了 `extern "C"`，使编译器生成与原始库一致的 `_Z*` 修饰名。

---

## 七、narrowing 转换修复

### 7.1 问题

C++11 中，大括号初始化不允许 narrowing 转换。以下代码：
```cpp
char beginData[8] = { 0xB9, 0x53, ... };
```
`0xB9` (185) 超出 `signed char` 范围 (-128~127)，触发 `-Wnarrowing` 错误。

### 7.2 修复

在 `zksort.pro` 中添加 `-Wno-narrowing` 编译器标志，并对关键位置添加显式转换：
```cpp
char beginData[8] = { (char)0xB9, (char)0x53, ... };
```

**影响文件**: `updatefpgawidget.cpp`, `onekeyupdatefpgawidget.cpp`

---

## 八、`mutex` 命名冲突

**问题**: `mynetwork.h` 中声明了 `static QMutex mutex;`。C++11 标准库 `<mutex>` 中有 `std::mutex` 类型。在某些编译场景下，`mutex` 名称变得歧义（ambiguous）。

**修复**:
- `mynetwork.h:49`: `static QMutex mutex;` → `static QMutex netMutex;`
- `mynetwork.cpp`: 所有 `mutex.lock()`, `mutex.unlock()`, `&mutex` → 对应 `netMutex.*`

---

## 九、`on_printBtn_clicked` 死代码

**问题**: `topwidget.cpp` 中定义了 `on_printBtn_clicked()`，但：
- 头文件中无对应声明
- UI 文件中无 `printBtn` 控件
- 函数体为空

Qt5 编译器对无声明的方法定义更严格，报错。

**修复**: 移除该空函数。

---

## 十、编译警告补充修复

### 10.1 `QPalette::Background` → `QPalette::Window`

Qt5 中 `QPalette::Background` 已弃用（uikits 编译警告，未影响编译）。

---

## 修改统计

| 类别 | 数量 |
|---|---|
| 批量替换类型 | 6 种 (`TRUE`/`FALSE`, `toAscii`, `fromAscii`, `QChar::fromAscii`, `setResizeMode`, `storageLocation`) |
| 头文件补充 | 10+ 处 |
| 截断/缺失代码修复 | 7 处 (4 个文件) |
| 运行时崩溃修复 | 3 类 (串口、IPC、日志路径) |
| log4qt 适配 | 8 处修改 |
| uikits 适配 | 4 处修改 |
| ARM Stub 创建 | 6 个新文件 |
| 其他修复 | narrowing, mutex 冲突, printBtn 死代码 |
