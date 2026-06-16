# QDir::entryInfoList() 用法详解

## 概述

`QDir::entryInfoList()` 是 `QDir` 类中用于**获取目录条目详细信息**的核心方法。它返回一个 `QFileInfoList`（即 `QList<QFileInfo>`），列表中每个 `QFileInfo` 对象都包含了对应文件或子目录的完整元信息，如文件名、大小、修改时间、权限等。

与 `entryList()` 只返回文件名列表不同，`entryInfoList()` 返回的是 `QFileInfo` 对象列表，可以直接获取每个条目的各种属性，无需再次构造 `QFileInfo`。

## 函数原型

```cpp
QFileInfoList QDir::entryInfoList(const QStringList &nameFilters,
                                   QDir::Filters filters = NoFilter,
                                   QDir::SortFlags sort = NoSort) const;

QFileInfoList QDir::entryInfoList(QDir::Filters filters = NoFilter,
                                   QDir::SortFlags sort = NoSort) const;
```

## QDir::Filter`常量表格
| 常量 | 说明 |
|------|------|
| `QDir::Dirs` | 只列出目录 |
| `QDir::Files` | 只列出文件 |
| `QDir::NoDotAndDotDot` | 不包含 `.` 和 `..` 这两个特殊目录 |
| `QDir::Hidden` | 包含隐藏文件（名称以 `.` 开头的文件） |
| `QDir::System` | 包含系统文件 |
| `QDir::Readable` | 只列出可读文件 |
| `QDir::Writable` | 只列出可写文件 |
| `QDir::Executable` | 只列出可执行文件 |
| `QDir::AllEntries` | 列出所有条目（默认值） |
| `QDir::NoFilter` | 不过滤，等同于 `AllEntries` |
## QDir::SortFlag常量表格
| 常量 | 说明 |
|------|------|
| `QDir::Name` | 按文件名排序 |
| `QDir::Time` | 按修改时间排序 |
| `QDir::Size` | 按文件大小排序 |
| `QDir::Type` | 按文件类型（后缀名）排序 |
| `QDir::Unsorted` | 不排序 |
| `QDir::DirsFirst` | 目录排在文件前面 |
| `QDir::DirsLast` | 目录排在文件后面 |
| `QDir::Reversed` | 逆序排列 |
| `QDir::IgnoreCase` | 不区分大小写排序 |
| `QDir::LocaleAware` | 使用本地化排序规则 |
