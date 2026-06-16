# QDebug 用法详解

## 概述

`QDebug` 是 Qt 框架中用于**输出调试和日志信息**的核心类。它提供了一个类似标准输出流的接口，可以将各种类型的数据格式化输出到控制台、文件或其他设备。`qDebug()` 是最常用的宏，它返回一个 `QDebug` 对象。

## 核心功能

| 功能 | 常用方法/宏 | 说明 |
|------|------------|------|
| 调试输出 | `qDebug()` | 输出调试信息 |
| 警告输出 | `qWarning()` | 输出警告信息 |
| 错误输出 | `qCritical()` | 输出严重错误信息 |
| 致命错误 | `qFatal()` | 输出信息后终止程序 |
| 流式操作 | `<<` | 将数据追加到输出流 |
| 空格控制 | `nospace()`, `space()` | 控制是否自动添加空格 |
| 引号控制 | `quote()`, `noquote()` | 控制字符串是否带引号 |
| 刷新缓冲 | `flush()` | 强制刷新输出缓冲区 |

## 基础用法示例

```cpp
#include <QDebug>

// 1. 基本输出
qDebug() << "Hello, Qt!";
qDebug() << "数值：" << 42;
qDebug() << "浮点数：" << 3.14159;

// 2. 不同级别的输出
qWarning() << "这是一个警告信息";
qCritical() << "这是一个严重错误信息";
// qFatal() << "致命错误，程序将终止";  // 注意：这会终止程序！

// 3. 输出 Qt 类型
QString name = "张三";
QDateTime now = QDateTime::currentDateTime();
qDebug() << "用户：" << name << "登录时间：" << now;

// 4. 输出容器类型
QStringList list = {"apple", "banana", "cherry"};
qDebug() << "水果列表：" << list;

QMap<QString, int> scores;
scores["张三"] = 90;
scores["李四"] = 85;
qDebug() << "成绩：" << scores;
```


相当于printf，用来打印信息的


