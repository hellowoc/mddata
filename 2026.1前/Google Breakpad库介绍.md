# Google Breakpad 库介绍

---

## 它是什么

Google Breakpad 是 Google 开源的一套**跨平台崩溃转储（crash dump）收集系统**。它的核心任务就是一句话：

> 程序崩溃时，自动生成一份 `.dmp` 文件（minidump），供开发者事后分析崩溃原因。

可以理解为程序自带的"黑匣子"。

---

## 为什么需要它

没有 Breakpad 时：

```
程序崩溃 → 直接消失 → 开发者：？？？用户也说不出发生了什么
```

有 Breakpad 之后：

```
程序崩溃 → 自动写 .dmp 文件 → 开发者用工具解析 → 精确到崩溃时的调用栈、寄存器、各线程状态
```

对于嵌入式设备尤其重要——设备装在客户工厂，开发者不在现场，只能靠 dump 文件复现问题。

---

## 三大组件

Breakpad 分为三个独立部分：

| 组件 | 运行位置 | 作用 |
|------|---------|------|
| **client** | 目标设备（程序内链接） | 捕获崩溃信号，生成 minidump 文件 |
| **symbol dumper** | 开发者 PC | 从编译出的二进制中提取调试符号，生成 `.sym` 文件 |
| **processor** | 开发者 PC | 把 `.dmp` + `.sym` 合并，还原出人类可读的调用栈 |

这个项目只用到了 **client** 部分——在设备上生成 dump。

---

## client 的工作原理

以 Linux 为例：

```
1. 注册信号处理器
   ExceptionHandler 构造时，通过 sigaction() 拦截这些信号：
   - SIGSEGV  (段错误 / 空指针)
   - SIGABRT  (abort / assert 失败)
   - SIGFPE   (除零等浮点异常)
   - SIGILL   (非法指令)

2. 崩溃发生
   信号触发 → Breakpad 的信号处理器接管

3. 生成 minidump
   在信号处理器中，用 ptrace 类机制读取当前进程的：
   - 各线程的寄存器状态
   - 调用栈内存
   - 已加载模块列表（.so/.exe 的加载基址）
   - 部分堆内存
   写入 .dmp 文件（Microsoft minidump 格式）

4. 调用回调
   写完后调用用户注册的 callback（本项目中的 dumpCallback）

5. 程序退出
   信号处理器返回，进程终止
```

---

## 这个项目中 Breakpad 的使用

```cpp
// main.cpp

// 1. 依赖头文件（仅 Linux）
#ifdef breakpad
#include "client/linux/handler/exception_handler.h"

// 2. 回调：dump 写完后打印路径
static bool dumpCallback(const MinidumpDescriptor& descriptor, 
                         void* context, bool succeeded) {
    qDebug() << "Dump path: " << descriptor.path();
    return succeeded;
}

// 3. 启动时创建存放目录
void setDumpPath(const QString& dirName) { ... }

// 4. main() 中注册
int main(int argc, char *argv[]) {
#ifdef breakpad
    setDumpPath("/crash");
    MinidumpDescriptor descriptor("./crash");
    ExceptionHandler eh(descriptor, NULL, dumpCallback, NULL, true, -1);
#endif
    ...
}
```

关键点：
- 所有 Breakpad 相关代码都在 `#ifdef breakpad` 宏保护下，**只存在于调试版本**
- dump 文件写入 `./crash` 目录
- 回调只做最简单的日志记录

---

## 开发者如何使用 dump 文件

假设设备上拿到了 `/crash/abc123.dmp`：

```bash
# 步骤1: 提取符号（在编译机上，对同一个未strip的二进制执行）
dump_sym zksort > zksort.sym

# 步骤2: 将符号文件放入正确的目录结构
# 目录名规则: <模块名>/<hash>/<模块名>.sym
mkdir -p symbols/zksort/ABC123DEF456
mv zksort.sym symbols/zksort/ABC123DEF456/

# 步骤3: 解析 dump
minidump_stackwalk abc123.dmp symbols/ > crash_report.txt
```

输出 `crash_report.txt` 示例：

```
Crash reason:  SIGSEGV /SEGV_MAPERR
Crash address: 0x0
Thread 0 (crashed)
 0  libc.so + 0x12345
 1  zksort!MyClass::crashHere() + 0x1a    ← 直接定位到崩溃函数
 2  zksort!MyClass::caller() + 0x2f
 3  zksort!main + 0x88
```

---

## 与 signal handler 的兼容注意事项

Breakpad 通过 `sigaction()` 注册信号处理器。这意味着：

- 如果项目里还有其他地方也注册了 SIGSEGV 处理器，**会发生冲突**（后注册的覆盖先注册的）
- Breakpad 自己的信号处理器在执行时，**能用的系统调用非常有限**（信号处理器上下文是异步信号安全的），所以回调函数里不能做复杂操作——只能写文件、打印简单日志。`dumpCallback` 里只调 `qDebug()` 是安全的，但如果尝试弹窗、分配内存、创建 QProcess 就可能死锁或二次崩溃。

---

## 关键对象详解

### MinidumpDescriptor

**定位：配置对象（告诉 Breakpad "写到哪里"）**

```cpp
MinidumpDescriptor descriptor("./crash");
```

这是一个轻量级的**描述符/配置对象**，只做一件事：指定 minidump 文件的输出位置。它不参与崩溃捕获，只是一个路径的载体。

两种构造方式：

| 构造方式 | 示例 | 含义 |
|---------|------|------|
| 目录路径 | `MinidumpDescriptor("./crash")` | 写入该目录下，文件名自动生成（UUID.dmp） |
| 完整文件路径 | `MinidumpDescriptor("./crash/my.dmp")` | 写入指定文件 |
| 文件描述符 | `MinidumpDescriptor(fd)` | 写入已打开的文件描述符 |

**核心方法就一个：**

```cpp
descriptor.path()   // 返回实际生成的 .dmp 文件完整路径
                    // 如 "./crash/7a8b9c3d-xxxx-xxxx.dmp"
```

这就是 `dumpCallback` 参数里 `descriptor.path()` 能拿到文件路径的原因——Breakpad 写完之后把实际路径填回 descriptor，回调里通过它拿到。

**在这个项目中的角色：**

```cpp
MinidumpDescriptor descriptor("./crash");   // ← 纯配置，指定输出到 ./crash 目录
ExceptionHandler eh(descriptor, ...);        // ← descriptor 被传给真正干活的对象
```

它本身不做任何事，只是一个"地址条"，被塞给 `ExceptionHandler` 后才起作用。

---

### ExceptionHandler

**定位：核心工作对象（真正干活的）**

```cpp
ExceptionHandler eh(descriptor,      // ① 写到哪里
                    NULL,            // ② 过滤器
                    dumpCallback,    // ③ 写完后的回调
                    NULL,            // ④ 回调的上下文参数
                    true,            // ⑤ 是否安装信号处理器
                    -1);             // ⑥ 外部 dump server fd
```

这个对象在**构造的那一刻**就完成了所有注册工作。它一诞生，Breakpad 就开始监听崩溃信号了。所以它是一个 **RAII 对象**——构造即激活，析构即卸载。

#### 构造参数详解

| # | 参数 | 本项目值 | 作用 |
|---|------|---------|------|
| ① | `descriptor` | `MinidumpDescriptor("./crash")` | 指定 dump 文件写入位置 |
| ② | `filter` | `NULL` | 崩溃过滤器。`NULL` = 所有崩溃都记录。可以传一个函数指针来选择性过滤（比如只记录特定类型的崩溃） |
| ③ | `callback` | `dumpCallback` | dump 写入完成后调用的回调函数。回调签名：`bool(const MinidumpDescriptor&, void* context, bool succeeded)` |
| ④ | `callback_context` | `NULL` | 传给回调的自定义数据指针。这里不需要额外数据所以传 NULL |
| ⑤ | `install_handler` | `true` | 是否在构造函数中自动安装信号处理器。`true` = 构造完立刻生效。设为 `false` 则表示"先不激活，等需要时再手动调用 Install()" |
| ⑥ | `server_fd` | `-1` | 外部 crash server 的文件描述符。`-1` = 不使用 out-of-process 模式，dumped 在当前进程自己写 |

#### 构造后发生了什么（install_handler = true 时）

```
ExceptionHandler 构造函数
        ↓
sigaction(SIGSEGV, &handler, NULL)   ← 安装对段错误的处理器
sigaction(SIGABRT, &handler, NULL)   ← 安装对 abort 的处理器
sigaction(SIGFPE,  &handler, NULL)   ← 安装对浮点异常的处理器
sigaction(SIGILL,  &handler, NULL)   ← 安装对非法指令的处理器
        ↓
对象构造完成，静默等待
        ↓ (某天信号触发)
handler 中调用 clone() 创建子进程
        ↓
子进程中：遍历 /proc/self/maps 收集加载模块信息
           ptrace 读取各线程寄存器
           调用 minidump writer 写入 .dmp 文件
           调用 callback(descriptor, context, success)
        ↓
父进程：子进程完成后，信号处理器返回，进程终止
```

#### 为什么要在子进程中写 dump

这是 Breakpad 的关键设计。崩溃时进程处于**信号处理器上下文**：

- 堆可能已被破坏（malloc 可能死锁）
- 线程可能持锁
- 栈可能被写坏

如果直接在崩溃线程里写文件，很可能**二次崩溃**。`clone()` 出一个干净的子进程，在隔离环境中安全写入 dump。

#### 生命周期

```cpp
int main(int argc, char *argv[]) {
#ifdef breakpad
    ExceptionHandler eh(...);   // ← 构造，激活崩溃监控
#endif
    // ... 整个程序运行期间都在监控下 ...

    return a.exec();
    // ← eh 离开作用域析构，卸载信号处理器
}
```

析构时自动调用 `sigaction()` 恢复原来的信号处理器，不会影响程序正常退出。

---

### 二者关系总结

```
MinidumpDescriptor          ExceptionHandler
     (地址条)                   (快递员)
        │                          │
        │  传入构造参数             │
        └─────────────────────────→│
                                   │
                           构造时注册信号处理器
                                   │
                           崩溃时：读 descriptor 里的路径
                                   │
                           把 .dmp 写到那个路径下
                                   │
                           路径填回 descriptor.path()
                                   │
                           调用 callback(descriptor, ...)
        │                          │
        │  回调中通过               │
        │  descriptor.path()        │
        │  拿到实际文件路径          │
        └──────────────────────────┘
```

`MinidumpDescriptor` = 告诉它**写到哪里**（配置信息）
`ExceptionHandler` = 负责**拦截崩溃、生成 dump、通知回调**（执行者）
