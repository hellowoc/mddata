# QDir 用法详解

## 概述

`QDir` 是 Qt 框架中用于**操作目录（文件夹）**的核心类。它提供了目录的创建、删除、遍历、路径解析等功能，相当于文件系统的“导航员”。

## 核心功能

| 类别 | 常用方法 | 说明 |
|------|---------|------|
| 静态特殊路径 | `homePath()` | 用户主目录 |
| | `tempPath()` | 系统临时目录 |
| | `currentPath()` | 当前工作目录 |
| | `rootPath()` | 根目录 |
| | `drives()` | Windows 驱动器列表 |
| 路径操作 | `absolutePath()` | 返回绝对路径 |
| | `canonicalPath()` | 规范化路径（解析符号链接） |
| | `relativeFilePath()` | 计算相对路径 |
| 存在性判断 | `exists()` | 目录是否存在 |
| | `isAbsolutePath()` | 是否为绝对路径 |
| | `isRelativePath()` | 是否为相对路径 |
| 目录管理 | `mkdir()` | 创建单级目录 |
| | `mkpath()` | 创建多级目录（类似 `mkdir -p`） |
| | `rmdir()` | 删除空目录 |
| | `removeRecursively()` | 递归删除目录及所有内容 |
| 目录遍历 | `entryList()` | 返回文件名列表 |
| | `entryInfoList()` | 返回 `QFileInfo` 列表 |
| 过滤与排序 | `setNameFilters()` | 设置文件名过滤器 |
| | `setFilter()` | 设置条目类型过滤器 |
| | `setSorting()` | 设置排序方式 |

## 基础用法示例

```cpp
#include <QDir>
#include <QDebug>

// 1. 获取特殊目录
QString home = QDir::homePath();          // 用户主目录
QString temp = QDir::tempPath();          // 临时目录
QString current = QDir::currentPath();    // 当前工作目录
QString root = QDir::rootPath();          // 根目录

// 2. 创建 QDir 对象
QDir dir("/home/user/projects");
// 或使用相对路径
QDir relDir("../data");

// 3. 检查目录是否存在
if (dir.exists()) {
    qDebug() << "目录存在";
} else {
    qDebug() << "目录不存在";
}

// 4. 获取绝对路径
qDebug() << dir.absolutePath();  // "/home/user/projects"

// 5. 创建目录
dir.mkdir("new_folder");          // 创建单级目录
dir.mkpath("a/b/c");              // 创建多级目录（自动创建不存在的父目录）

// 6. 删除目录
dir.rmdir("new_folder");          // 只能删除空目录
dir.removeRecursively();          // 递归删除，慎用！