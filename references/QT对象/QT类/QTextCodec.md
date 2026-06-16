# QTextCodec 用法详解

## 概述

`QTextCodec` 是 Qt 框架中用于**文本编码转换**的核心类。Qt 内部使用 Unicode 来存储和处理字符串，而外部数据（如文件、网络传输）可能使用其他编码（如 GBK、Shift-JIS）。`QTextCodec` 的作用就是在 Unicode 和各种本地编码之间进行双向转换 [citation:1][citation:4]。

## 核心功能

| 类别 | 常用方法 | 说明 |
|------|---------|------|
| 获取编解码器 | `codecForName(name)` | 按编码名称获取 `QTextCodec` 对象 |
| | `codecForLocale()` | 获取本地系统默认编码的 `QTextCodec` |
| 解码（→ Unicode） | `toUnicode(QByteArray)` | 将本地编码转换为 Unicode（QString） |
| | `makeDecoder()` | 创建用于分块解码的 `QTextDecoder` |
| 编码（Unicode →） | `fromUnicode(QString)` | 将 Unicode 转换为本地编码（QByteArray） |
| | `makeEncoder()` | 创建用于分块编码的 `QTextEncoder` |
| 可用编码列表 | `availableCodecs()` | 获取系统支持的所有编码名称 |

## 基础用法示例

```cpp
#include <QTextCodec>
#include <QDebug>

// 1. 按编码名称获取编解码器
QTextCodec *gbkCodec = QTextCodec::codecForName("GBK");
QTextCodec *utf8Codec = QTextCodec::codecForName("UTF-8");
QTextCodec *shiftJisCodec = QTextCodec::codecForName("Shift-JIS");

// 2. 获取本地系统编码
QTextCodec *localeCodec = QTextCodec::codecForLocale();

// 3. 解码：本地编码 → Unicode
QByteArray encodedData = "\xba\xc3";  // GBK 编码的 "好"
QString unicodeStr = gbkCodec->toUnicode(encodedData);
qDebug() << unicodeStr;  // "好"

// 4. 编码：Unicode → 本地编码
QString text = "Hello 世界";
QByteArray gbkData = gbkCodec->fromUnicode(text);   // 转为 GBK
QByteArray utf8Data = utf8Codec->fromUnicode(text);  // 转为 UTF-8