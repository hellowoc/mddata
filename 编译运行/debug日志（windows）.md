# Qt 项目 Windows 移植调试日志

## 环境选择过程

- 在 **Qt 5** 环境下运行时存在诸多环境问题。
- **Qt 4.8.7** 适配的 MinGW 环境已配好，但该版本需要翻墙通过官网下载。
- **Qt 4.8.4** 同样需要翻墙下载，MinGW 环境已配好。
- 尝试在 Qt 4.8.7 环境下运行，仍存在诸多问题。
- 最终在 **Qt 4.8.4 + MinGW 4.2** 下运行，环境问题基本解决。

## MinGW 兼容性问题

由于 MinGW 编译器过于古老，一些写法存在不兼容问题，通过 AI 进行了修改。

### 功能屏蔽（Windows 平台不需要）

一些功能在 Windows 不需要实现（例如各种通信），进行了屏蔽和修改，部分函数做了空定义：

- **`svmimagewidget.cpp`**：屏蔽整个 `SvmLearnThread::run()` 函数
- **`modelselectwidget.cpp:69`**：屏蔽
- **`myqextserialport.cpp/.h`**：文件中均进行了一些修改和空定义

### 串口通信空桩函数

```cpp
/*************************************************************************************************************
 * change by zcy 2026.5.28    (.cpp)
 * ************************************************************************************************************/
int sendCom2Cmd(char *data)
{return 0;}
void pushCom2CmdData(int nCmd, char intAddr, char boardType,  char boardAddr, char *data, int nCount)
{}
void com2WriteDirect(int nCmd, char intAddr, char boardType,  char boardAddr, char *data, int nCount)
{}

/*************************************************************************************************************
     * change by zcy 2026.5.28  (.h)
     * ************************************************************************************************************/
    int sendCom2Cmd(char *data);
    void pushCom2CmdData(int nCmd, char intAddr, char boardType,  char boardAddr, char *data, int nCount);
    void com2WriteDirect(int nCmd, char intAddr, char boardType,  char boardAddr, char *data, int nCount);
```

## AI 修改记录

### aichange1

- **`myfastnetwork.cpp`**：已修复，加上了 `(const char*)` 强制类型转换。
- **`mynetwork.cpp`**：已经用 `#ifdef Q_OS_UNIX` / `#else` 正确处理了跨平台兼容，不需要改动。

### aichange2

Windows 的 WinSock2 API 中 `setsockopt` 的第 4 个参数类型是 `const char*`，而 Unix 上是 `const void*`，所以 Windows 下必须显式转换。

> change2 将文件中关于编译器不兼容的问题完成了修改，对于其他内容没有修改过，因此可以作为**回退的锚点版本**。

### aichange3

- **`myqextserialport.cpp:2670-2675`**：全局函数加上了 `MyQextSerialPort::` 前缀，变成成员函数桩。
- **新建 `3rdparty/svm/wheat_stub.cpp`**：提供 `Wheat::Wheat_Main` 空实现（原 `libWheat.a` 是 Linux/ARM 库，Windows 无法链接）。
- **`zksort.pro:544`**：在 `win32` 块中添加 `wheat_stub.cpp` 到编译列表。

### aichange4

- **新建 `3rdparty/svm/svmtool_stub.cpp`**：四个全局函数的空桩：
  - `train` → 返回失败
  - `isBad` → 返回 `false`
  - `shape_sampling` / `shape_classify` → 空操作
- **`zksort.pro`**：将 `svmtool_stub.cpp` 加入 `win32` 编译列表，移除了 `-lsvmtool` 链接（该 `.a` 是 MSVC 编译的，MinGW 无法链接其 MSVC 运行时依赖 `__ms_vsnprintf`）。

## 调试过程

### 尝试 1：Qt 4.8.7 + MinGW 4.2

运行后不显示窗口，程序异常结束。回退到 **aichange3 之前**的版本。

### 尝试 2：重新修改

确认具体错误 —— 就是之前的 `__ms_vsnprintf` 问题。`libsvmtool.a` 是 MSVC 编译的，MinGW 无法链接。

**解决尝试**：用桩函数替代 MSVC 库。

```qmake
SOURCES += $$PWD/../3rdparty/svm/wheat_stub.cpp \
           $$PWD/../3rdparty/svm/svmtool_stub.cpp
# LIBS += -L$$PWD/../3rdparty/svm/win -lsvmtool   # 已移除
LIBS += -L$$PWD/../3rdparty/qrencode/lib/win32 -lqrencoded
```

此时仍然存在问题，保留修改到这个位置的副本以做后续备用。
将版本回退到 **change3 之前**，保留版本。

### 尝试 3：转向 MSVC

二次修改后仍然存在问题。该工程原本是在 **MSVC** 环境下设计的，因此尝试安装 MSVC 环境。

- **VS2010 直接构建**：IDE 过于老旧，出现了报错，放弃该方案。
- **MSVC 中编译**：出现了上百个缺失分号、括号、结构体说明等格式问题，需要排查原因。

### 最终尝试

将 Qt 的各个版本都尝试了，最终确定在 **Qt 4.8.4** 这个版本。MinGW 下有很多问题无法解决，AI 建议在 MSVC 下运行，目前已进入 **在 MSVC 中调试的阶段**。

MSVC环境配置出现问题，配置好了编译器和版本，仍然有黄色感叹号