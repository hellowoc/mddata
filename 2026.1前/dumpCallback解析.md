# `dumpCallback` 函数解析

这个函数出现在 `main.cpp:22-26`：

```cpp
static bool dumpCallback(const google_breakpad::MinidumpDescriptor& descriptor, 
                         void* context, bool succeeded)
{
    qDebug() << "Dump path: " << descriptor.path() << " succeeded:" << succeeded;
    return succeeded;
}
```

---

## 它是什么

这是 **Google Breakpad** 库的**崩溃回调函数**。当程序崩溃时，Breakpad 会在生成 minidump 文件后调用它，通知上层"崩溃信息已经写好了"。

---

## 谁调用了它

在 `main.cpp:51` 注册：

```cpp
google_breakpad::MinidumpDescriptor descriptor("./crash");
google_breakpad::ExceptionHandler eh(descriptor, NULL, dumpCallback, NULL, true, -1);
//                                    ①        ②     ③             ④     ⑤     ⑥
```

| 参数 | 值 | 含义 |
|------|-----|------|
| ① `descriptor` | 指定 dump 文件输出到 `./crash` 目录 | 崩溃文件的存放路径 |
| ② `filter` | `NULL` | 不过滤，所有类型的崩溃都捕获 |
| ③ `callback` | `dumpCallback` | dump 写入完成后回调的函数 |
| ④ `context` | `NULL` | 传给回调的自定义数据，这里不需要 |
| ⑤ `install_handler` | `true` | 注册信号处理器（SIGSEGV/SIGABRT 等） |
| ⑥ `server_fd` | `-1` | 不使用外部 out-of-process dump 服务器 |

---

## 回调参数的逐项解释

| 参数 | 类型 | 含义 |
|------|------|------|
| `descriptor` | `MinidumpDescriptor` | 描述 dump 文件的对象，调用 `.path()` 可拿到完整路径，如 `./crash/abc123.dmp` |
| `context` | `void*` | 注册时传入的自定义上下文指针，这里传的 `NULL` 所以没用 |
| `succeeded` | `bool` | dump 是否写入成功。`true` = minidump 文件已完整写入磁盘；`false` = 写入过程中出错 |

---

## 返回值 `bool` 的含义

```
return succeeded;
```

- **`return true`** → 告诉 Breakpad："我处理完了，你继续走标准流程"（如果注册了的话，会弹出崩溃对话框等）
- **`return false`** → 告诉 Breakpad："直接退出程序，别做额外处理"

这里的策略是：**如果 dump 写成功了，就走默认流程；如果连 dump 都没写成功，也别折腾了，直接死掉。**

---

## 这个项目里的实现为什么这么"薄"

```cpp
// 这个项目只做了一件事：打印一行日志
qDebug() << "Dump path: " << descriptor.path() << " succeeded:" << succeeded;
```

对比 Breakpad 官方示例中的典型用法：

```cpp
// 官方示例通常会在回调里做更多事：
static bool dumpCallback(...) {
    // 1. 把 dump 文件重命名/移动到固定位置
    // 2. 弹出一个对话框告诉用户"程序崩溃了"
    // 3. 把 dump 路径写入配置文件，下次启动时自动上传
    // 4. 尝试重新启动程序
    return succeeded;
}
```

这个项目之所以只打了一行日志，原因：
- **嵌入式设备**没有桌面弹窗的概念
- **没有网络上传**的需求（量产设备离线运行）
- 日志已经写到了 `log.txt`，事后排查时翻日志就能知道 dump 文件路径
- 保持简洁，不引入额外复杂度

---

## 整体流程

```
程序运行时发生 SIGSEGV（段错误）
        ↓
Breakpad 信号处理器拦截
        ↓
生成 minidump 写入 ./crash/<uuid>.dmp
        ↓
调用 dumpCallback()
  ├── 打印路径和成功状态到 qDebug
  └── return succeeded
        ↓
程序终止
        ↓
下次开机后，开发人员查看 log.txt 找到 dump 路径
用 minidump_stackwalk 工具解析 .dmp 文件定位崩溃原因
```

---

## 注意

这个回调以及 Breakpad 整个功能都在 `#ifdef breakpad` 宏保护下，**只在开启 `breakpad` 编译选项的调试版本中生效**。量产固件通常不加这个宏，不会引入 Breakpad 的依赖和开销。
