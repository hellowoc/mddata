# `setDumpPath` 函数解析

这个函数出现在 `main.cpp:28-44`：

```cpp
void setDumpPath(const QString& dirName)
{
#ifdef Q_OS_LINUX
    QDir dir;
    bool bExists = dir.exists("./" + dirName);
    if (!bExists)
    {
        bool ok = false;
        ok = dir.mkdir("./" + dirName);
        qDebug() << "create dir " << dirName << " success";
    }
    else
    {
        qDebug() << dirName << " exist";
    }
#endif
}
```

---

## 它做了什么

一句话：**在程序当前目录下创建一个用于存放崩溃 dump 文件的子目录。**

---

## 逐行解读

### `#ifdef Q_OS_LINUX`

```cpp
#ifdef Q_OS_LINUX
    // ... 函数体 ...
#endif
```

整个函数体被 `Q_OS_LINUX` 宏包裹。这意味着：
- **Linux 平台** → 函数正常执行，创建目录
- **Windows 平台** → 函数体为空，什么都不做

理由很简单：这个项目在实际产品中跑在 ARM Linux 上，Breakpad 也只链接 Linux 版本，Windows 桌面只是调试用的模拟环境，不需要崩溃转储。

### `dir.exists("./" + dirName)`[[QDir]]

```cpp
QDir dir;
bool bExists = dir.exists("./" + dirName);
```

检查**程序当前工作目录**下是否已存在同名子目录。`"./"` 表示当前路径（对于嵌入式设备，通常是程序所在目录）。

在 `main()` 中调用时传入的是 `"/crash"`，但注意拼接方式：

```cpp
setDumpPath("/crash");   // dirName = "/crash"
                         // 实际检查 "./" + "/crash" = ".//crash"
```

这里其实有个小瑕疵：`"/crash"` 已经以 `/` 开头，拼接 `"./" + "/crash"` 得到 `".//crash"`，虽然 Qt 的路径处理能容忍双斜杠，但逻辑上不够严谨。更干净的写法应该是 `"./crash"` 或者直接 `"crash"`。

### 互斥分支[[QDebug 用法]]

```cpp
if (!bExists) {
    // 目录不存在 → 创建
    bool ok = false;
    ok = dir.mkdir("./" + dirName);
    qDebug() << "create dir " << dirName << " success";
} else {
    // 目录已存在 → 只打印
    qDebug() << dirName << " exist";
}
```

两个细节值得注意：
1. **`ok` 变量没有被使用** — `mkdir()` 的返回值虽然赋给了 `ok`，但创建失败也没有报错，代码只管打印 "success"。这是个不严谨但无伤大雅的地方。
2. **只创建不清理** — 目录已存在时不做任何操作，旧的 dump 文件会保留，方便累积多个崩溃记录而不是被覆盖。

---

## 和 `dumpCallback` 的关系

两者配合形成完整的崩溃记录链路：

```
setDumpPath("/crash")         ← 阶段1: main() 启动时，创建 ./crash 目录
        ↓
ExceptionHandler(./crash)     ← 阶段1: 注册 Breakpad，指向同一目录
        ↓
    ... 程序运行 ...
        ↓ (某天崩溃)
Breakpad 写 minidump 到 ./crash/xxx.dmp
        ↓
dumpCallback()                ← 阶段2: 回调，打印 dump 路径到日志
```

---

## 为什么要单独抽出成函数

看似只有 3 行有效代码，抽成独立函数有两个原因：
1. **语义清晰** — `setDumpPath("/crash")` 比内联写一串 `QDir` 代码可读性好
2. **预留扩展点** — 后期如果需要在创建目录的同时做权限设置（如 `chmod`）、检查磁盘空间、清理旧 dump 等，只需改这一个函数
