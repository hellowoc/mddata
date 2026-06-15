# Qt 资源系统 vs 真实文件系统

## 本质关系

它们是**两套平行的"文件查找机制"**，Qt 内部通过路径前缀自动分流：

```cpp
QFile file(":/white/skin/qss/skin.qss");   // ":" → 走资源系统
QFile file("/sdcard/logs/log.txt");         // "/" → 走真实文件系统
QFile file("../resource/skin.qss");         // "." → 走真实文件系统
```

两者的切换机制在 QFile 构造时：

```
QFile::QFile("/sdcard/logs/log.txt")
    → 路径不以 ":" 开头 → 走真实文件系统
    → 最终用 ::open() 调用 Linux 内核 VFS

QFile::QFile(":/white/skin/qss/skin.qss")
    → 路径以 ":" 开头 → 走 Qt 资源系统
    → 在全局资源注册表中查找
    → 如果有 → 返回资源数据
    → 如果无 → 失败（不会降级去查真实文件系统）
```

---

## 对比

```
真实文件系统                          Qt 资源系统
───────────────                      ──────────────
数据来源: 磁盘上的实际文件            数据来源: .rcc 文件或编译进 exe 的内部数据
路径示例: /sdcard/logs/log.txt       路径示例: :/white/skin/qss/skin.qss
寻址方式: inode → 扇区 → 磁盘读取    寻址方式: 资源表查找 → .rcc内部偏移 → 内存读取
是否可写: 是（open + write）         否（只读）
修改持久: 是                         否（.rcc 是静态归档）
文件列表: ls / opendir / readdir     只有注册的资源，无法"列举"
存储开销: 每个文件一个 inode + 至少一个扇区   连续存储，零碎片
```

---

## 它们不是"上下级"关系

资源系统**不是**架在真实文件系统之上的虚拟层。正确理解：

```
QFile::QFile(path)
        │
        ├── path 以 ":" 开头 ──→ Qt 资源系统 (纯内存查表，不碰磁盘)
        │                           ├── 资源注册表[0]
        │                           ├── 资源注册表[1]  (可以有多个 .rcc 同时注册)
        │                           └── ...
        │
        └── path 以 "/" 或 "." 开头 ──→ 真实文件系统 (系统调用 open/read/write)
                                        ↓
                                    Linux VFS
                                        ├── ext4 (eMMC)
                                        ├── vfat (SD卡/U盘)
                                        └── proc (/proc 虚拟文件系统)
```

它们是**并列的两条路径**，Qt 在构造 QFile 时做一次 `if (path.startsWith(':'))` 的判断，然后分别走完全不同的代码分支。

---

## 资源系统内部的细节

资源注册表是一个全局单例：

```cpp
// Qt 源码中资源系统的简化结构
class QResourceGlobalData {
    QList<QResource> registeredResources;   // 已注册的 .rcc 列表
    QHash<QString, ResourceEntry> entryMap; // 路径 → 资源数据的映射
};

// 每次 registerResource() 调用:
//   1. 打开 .rcc 文件
//   2. 解析目录树
//   3. 把每条路径插入 entryMap
//      "white/skin/qss/skin.qss" → {rccFileOffset: 0x1234, size: 5432}
//      "white/skin/qss/skin_1024_600.qss" → {rccFileOffset: 0x6789, size: 3456}
//      ...
```

访问时：

```cpp
// QFile(":/white/skin/qss/skin.qss").readAll()
//   ↓ 去掉 ":" 前缀 → "white/skin/qss/skin.qss"
//   ↓ entryMap["white/skin/qss/skin.qss"]
//   ↓ 命中 → {offset: 0x1234, size: 5432}
//   ↓ 直接 .rcc 文件内定位读取 5432 字节
//   ↓ 返回数据
```

---

## 真实文件系统路径在这个项目中的使用

```
// 配置文件 (运行时读写，不能用 .rcc)
"/sdcard/cnf/"                    ← JSON 配置
"/sdcard/logs/log.txt"            ← 日志文件
"/sdcard/logo/boot.bmp"           ← 启动画面
"/sdcard/ts/corndata.csv"         ← 统计数据

// 系统文件
"/proc/mounts"                    ← 内核挂载信息
"/proc/scsi/usb-storage"          ← USB 状态

// 资源文件 (静态不变，适合打包)
":/white/skin/qss/skin.qss"       ← 界面样式
":/resource/languages/..."        ← 翻译文件
```

**判断标准很简单：运行时会变的数据走真实文件系统；编译完就不变的静态数据走资源系统。**

---

## 为什么两者不能混用

你**不能**在资源路径上做真实文件系统的操作：

```cpp
// ❌ 以下全部失败
QFileInfo::exists(":/skin.qss");  // Qt 内部对资源路径有特殊处理
QDir dir(":/white");              // 不能列举资源目录
QFile::rename(":/old.qss", ":/new.qss");  // 资源是只读的

// ❌ 反过来也不行——不能在真实文件系统路径上用资源路径的方式
QResource::registerResource("/sdcard/logs/log.txt");  // 魔数不对，失败
```

---

## 一句话总结

Qt 资源系统和真实文件系统是**同一套 QFile API 下的两条独立实现**。路径开头有没有 `:` 决定了走哪条路——就像网络请求中 `http://` 和 `file://` 的区别，前缀决定协议，后面才是路径。
