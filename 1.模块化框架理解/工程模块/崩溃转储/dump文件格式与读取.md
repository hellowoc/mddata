# Dump 文件格式与读取

## Dump 文件格式

生成的 `.dmp` 文件是 **Microsoft Minidump 格式**（这就是 `MinidumpDescriptor` 名字的由来）。Breakpad 借用了 Windows 的 minidump 标准格式，跨平台统一使用，它不是 Breakpad 自己发明的。

---

## 从 dump 里能读到什么

以下内容是 **Breakpad 自动写入的，你不需要写任何代码去生成**：

| 信息类别 | 具体内容 |
|---------|---------|
| 崩溃原因 | 哪个信号触发（`SIGSEGV`/`SIGABRT`/`SIGFPE`/`SIGILL`），崩溃时的内存地址 |
| 所有线程的调用栈 | 每个线程当时的函数调用链，直接定位到崩溃在哪个函数 |
| CPU 寄存器 | 每个线程的 PC、SP、LR 等寄存器快照 |
| 已加载模块列表 | 进程加载了哪些 .so/可执行文件，各自的加载基址、版本 |
| 部分栈内存 | 崩溃线程的栈原始内存数据 |
| 线程列表 | 崩溃时所有存活的线程 ID 和名称 |

示例——解析后能看到：

```
Crash reason:  SIGSEGV /SEGV_MAPERR
Crash address: 0x0                          ← 空指针解引用

Thread 0 (crashed) 名称: "Mainthread"       ← 主线程崩了
 0  libc.so + 0x1a234
 1  zksort!fpga::writeReg(int, int) + 0x3c  ← 崩在 fpga.cpp 的 writeReg 函数
 2  zksort!fpga::initHardware() + 0x15      ← 调用者是 initHardware
 3  zksort!GlobalFlow::initAll() + 0x88     ← 再上一层
 ...
```

---

## 读取 dump：Breakpad 已经提供了，不需要自己写

你 **不需要自己写解析器**。Breakpad 提供了两个现成的命令行工具：

| 工具 | 来源 | 作用 |
|------|------|------|
| `dump_syms` | Breakpad 自带，你编译产生 | 从编译好的二进制文件中提取调试符号 → `.sym` 文件 |
| `minidump_stackwalk` | Breakpad 自带，你编译产生 | 把 `.dmp` + `.sym` 合并 → 输出可读的调用栈文本 |

流程：

```bash
# 1. 提取符号（一次性，每个发布版本保留一份 .sym）
dump_syms zksort > zksort.sym

# 2. 解析 dump（每次拿到设备上的 .dmp 后执行）
minidump_stackwalk crash.dmp symbols/ > report.txt
```

这两个工具都是 Breakpad 源码的一部分，你编译 Breakpad 库的时候就会产生，不需要你写一行解析代码。

---

## 你只需要关注的三件事

| 你做的事 | 说明 |
|---------|------|
| 拿到 `.dmp` 文件 | 从设备 `/crash/` 目录拷出 |
| 保留未 strip 的二进制 | 编译时记得保留一份带符号的 zksort（不要 strip），否则 `.sym` 提取不出函数名 |
| 跑 `minidump_stackwalk` | 一行命令出结果 |

其他所有——dump 怎么生成、怎么格式化、怎么解析——**Breakpad 全部给你做好了**。
