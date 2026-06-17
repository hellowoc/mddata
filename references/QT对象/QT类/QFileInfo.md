# QFileInfo 用法详解

## 概述

`QFileInfo` 是 Qt 框架中用于获取**文件系统元信息**的工具类。它不负责读写文件内容，而是提供与文件相关的各种属性查询功能，例如路径、大小、时间戳、权限等。

## 核心功能

`QFileInfo` 可以获取以下信息：

| 类别   | 常用方法                 | 说明                 |
| ---- | -------------------- | ------------------ |
| 路径信息 | `filePath()`         | 返回完整路径（构造时传入的路径）   |
|      | `absolutePath()`     | 返回绝对路径（不含文件名）      |
|      | `absoluteFilePath()` | 返回带文件名的绝对路径        |
| 文件信息 | `fileName()`         | 返回文件名（含后缀）         |
|      | `baseName()`         | 返回文件名（不含后缀）        |
|      | `suffix()`           | 返回文件后缀名            |
|      | `size()`             | 返回文件大小（字节）         |
| 时间信息 | `created()`          | 返回创建时间 `QDateTime` |
|      | `lastModified()`     | 返回最后修改时间           |
|      | `lastRead()`         | 返回最后读取时间           |
| 权限信息 | `isReadable()`       | 是否可读               |
|      | `isWritable()`       | 是否可写               |
|      | `isExecutable()`     | 是否可执行              |
| 类型判断 | `isFile()`           | 是否为普通文件            |
|      | `isDir()`            | 是否为目录              |
|      | `isSymLink()`        | 是否为符号链接/快捷方式       |
|      | `isHidden()`         | 是否为隐藏文件            |
| 存在性  | `exists()`           | 文件或目录是否存在          |
| 引用文件 | `symLinkTarget()`    | 返回符号链接指向的目标路径      |

## 基础用法示例

```cpp
#include <QFileInfo>
#include <QDebug>

QFileInfo info("/home/user/document.txt");

qDebug() << "文件名:" << info.fileName();          // "document.txt"
qDebug() << "绝对路径:" << info.absolutePath();    // "/home/user"
qDebug() << "后缀名:" << info.suffix();            // "txt"
qDebug() << "文件大小:" << info.size() << "字节";  // 文件字节数
qDebug() << "最后修改:" << info.lastModified();    // QDateTime 对象
qDebug() << "是否可读:" << info.isReadable();      // true/false
qDebug() << "是否为目录:" << info.isDir();         // true/false
qDebug() << "是否存在:" << info.exists();          // true/false

##两个重要特性
构造 QFileInfo 对象时，目标文件不必实际存在。你可以先构造对象，再通过 exists() 方法检查文件是否存在。
QFileInfo info("/path/to/nonexistent/file.txt");

if (!info.exists()) {
    qDebug() << "文件不存在，可以进行创建操作";
}

QFileInfo 会在首次访问时缓存文件信息以提高性能。如果文件在程序运行期间被外部程序修改，缓存的信息可能过时，需要调用 refresh() 手动刷新。
QFileInfo info("/home/user/data.txt");

// 首次获取文件大小
qDebug() << info.size();  // 例如：1024 字节

// ... 期间外部程序修改了文件 ...

// 刷新缓存后获取最新信息
info.refresh();
qDebug() << info.size();  // 例如：2048 字节（更新后的值）

##与QFile 的区别

| 类 | 职责 |
| ---------- | -------------------------------------------- |
|  QFile     | 读写文件内容（打开、读取、写入、关闭） |
|  QFileInfo | 查询文件属性（大小、时间、权限、类型等） |

两者通常配合使用：先用 QFileInfo 检查文件状态，再用 QFile 执行实际的读写操作。
QFileInfo info("data.txt");
if (info.exists() && info.isWritable()) {
    QFile file(info.absoluteFilePath());
    if (file.open(QIODevice::ReadWrite)) {
        // 读写操作...
        file.close();
    }
}
