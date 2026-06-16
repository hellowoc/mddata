# QResource::registerResource 详解

## 调用位置

```cpp
// main.cpp
QResource::registerResource(QApplication::applicationDirPath()+"/resource.rcc");
```

传入的是一个路径字符串，返回 `bool`（注册成功/失败），返回值在这里被忽略了。

---

## QResource 是什么

```
QResource
  继承: 无（独立工具类，不继承 QObject）
  定位: Qt 资源系统的运行时加载器
```

它不是在屏幕上的控件，而是一个**后台的文件系统抽象层**——让程序像读文件一样读取内嵌资源，无论资源来自磁盘上的 `.rcc` 文件还是编译进可执行文件内部。

---

## `.rcc` 文件是什么

`.rcc` 是 Qt Resource Collection 的缩写，由 Qt 自带的 `rcc` 工具编译产生：

```
编译时:
  resource.qrc (XML描述文件，列出哪些文件要打包)
      ↓ rcc 编译器
  resource.rcc  (二进制归档文件)

运行时:
  QResource::registerResource("resource.rcc")
      ↓ 加载到内存
  程序通过 ":/skin/qss/skin.qss" 路径访问
```

本质就是一个**二进制压缩归档文件**——类似于 `.zip` 但没有压缩，结构更简单，专为 Qt 优化。把从 PNG 图片到 QSS 样式表到翻译文件，全部打进一个二进制包。

---

## 注册前后的对比

```
注册前:
  QFile(":/white/skin/qss/skin.qss")
  → Qt 在内部资源表中查找 -> 找不到 -> 失败

QResource::registerResource("/path/to/resource.rcc");
  → 打开.rcc文件 → 解析目录树 → 注册到全局资源表

注册后:
  QFile(":/white/skin/qss/skin.qss")
  → Qt 在内部资源表中查找 -> 找到了 -> 返回文件数据
```

`:` 前缀是 Qt 资源系统的虚拟根目录，后面的路径是 `.qrc` 文件中定义的资源别名，**不是实际文件系统路径**。

---

## registerResource 内部流程

```cpp
// Qt 源码中 registerResource 的简化逻辑

bool QResource::registerResource(const QString &file)
{
    // 1. 打开 .rcc 文件
    QFile rccFile(file);
    if (!rccFile.open(QIODevice::ReadOnly))
        return false;

    // 2. 读取 .rcc 文件头，验证魔数
    //    文件开头四个字节: 'q' 'r' 'e' 's' (qresource)
    quint32 magic;
    rccFile.read((char*)&magic, 4);
    if (magic != 0x73657271)  // "qres" 的小端编码
        return false;

    // 3. 读取版本号、目录偏移量、文件树
    //    .rcc 内部结构:
    //    ┌──────────────┐
    //    │ 文件头(magic, version, timestamp, ...)    │
    //    ├──────────────┤
    //    │ 目录树(文件名、偏移、大小)                 │
    //    │  ├── white/                               │
    //    │  │   ├── skin/                            │
    //    │  │   │   └── qss/skin.qss (offset=0x1234, size=5432) │
    //    │  │   └── ...                              │
    //    │  └── ...                                  │
    //    ├──────────────┤
    //    │ 文件数据区                                 │
    //    │  └── skin.qss 的原始字节                   │
    //    └──────────────┘

    // 4. 将目录树映射到全局资源注册表
    //    注册后，Qt 的文件系统代理层知道:
    //    ":/white/skin/qss/skin.qss" → .rcc文件偏移0x1234, 大小5432字节

    // 5. （可选）mmap 映射整个 .rcc 文件，后续读取零拷贝

    return true;
}
```

---

## 注册后如何读取

当业务代码请求一个资源时：

```cpp
// zksort.cpp LoadSkin()
QFile file(":/white/skin/qss/skin.qss");
file.open(QIODevice::ReadOnly);
QString styleSheet = file.readAll();
```

Qt 内部执行路径：[[Qt资源系统与文件系统]]

```
QFile(":/white/skin/qss/skin.qss")
    ↓
检测到路径以 ":" 开头 → 走资源系统而非真实文件系统
    ↓
在全局资源注册表中查找路径 "/white/skin/qss/skin.qss"
    ↓
找到注册项 → 指向 resource.rcc 中偏移 0x1234
    ↓
file.readAll()
    ↓
直接定位到 .rcc 文件第 0x1234 字节 → 读取 5432 字节
    ↓
返回原始 QSS 文本数据
```

**关键：整个过程不经过操作系统文件 API**——`open()` 不需要磁盘寻道（除了首次打开 `.rcc`），`read()` 直接定位 `.rcc` 内部偏移，比读散落的小文件快得多。

---

## 为什么嵌入式设备喜欢 .rcc

| 对比项 | 散落文件（100个图片 + 20个翻译 + 5个QSS） | 单文件 .rcc |
|--------|------------------------------------------|------------|
| 文件数量 | 125 个文件 | 1 个文件 |
| 磁盘 I/O | 每次打开一个文件都要 open + read + close | 一次 mmap，后续零拷贝 |
| 空间效率 | 每个文件占用一个扇区（最小 4KB） | 连续存储，无浪费 |
| 部署 | 125 个文件容易缺失 | 1 个文件，缺了就全缺 |
| 防篡改 | 客户可以替换单个 PNG 图片 | 整包替换才生效 |

嵌入式设备 I/O 性能差（SD 卡随机读写很慢），文件数量多是性能杀手。把几百个小文件打成一个大文件，**寻道次数从几百次降到 1 次**。

---

## registerResource 的返回值和错误处理

```cpp
bool success = QResource::registerResource("/path/to/resource.rcc");
if (!success) {
    qDebug() << "Failed to load resource file";
}
```

返回 `false` 的情况：

| 原因 | 说明 |
|------|------|
| 文件不存在 | 路径错误、文件被删除 |
| 文件格式错误 | 不是有效的 .rcc 文件（魔数不匹配） |
| 版本不兼容 | .rcc 版本号和当前 Qt 版本不匹配 |
| 内存不足 | 无法分配内部数据结构 |

这个项目里没有检查返回值（直接调用不接收返回值），说明这是一个**不需要容错的操作**——`resource.rcc` 和设备固件同时部署，不存在缺失的可能。

---

## 在这个项目中的作用

```
resource.rcc 注册后，以下资源都通过 ":" 路径访问：

":/white/skin/qss/skin.qss"          → ZKSort::LoadSkin() 加载界面样式
":/white/skin/qss/skin_1024_600.qss" → 适配 1024×600 屏幕
":/white/skin/qss/skin_1024_768.qss" → 适配 1024×768 屏幕
":/resource/languages/Chinese_lan.ts"→ 中文翻译
":/resource/languages/English_lan.ts"→ 英文翻译
... (其他 23 种语言的翻译)
... (界面图片 PNG/BMP)
```

## 理解
rcc文件是一个QT内置的编译工具产生的，他把散落的各个资源类似压缩一般，形成了一个更容易检索和获得的rcc文件，从而在寻找资源的时候可以更快。

此处的registerResource[[QResource及rcc]]就是在把rcc文件的地址注册上，方便后面查找