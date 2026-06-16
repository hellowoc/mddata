# zksort Qt5 迁移 — 配置变更汇总

## 1. 编译环境

| 项目 | 原值 | 新值 |
|---|---|---|
| Qt 版本 | Qt 4.8.4 (ARM) | Qt 5.15.3 (x86_64) |
| 目标架构 | ARM 32-bit | x86_64 |
| 编译器 | arm-linux-gnueabihf-g++ | g++ 11 (x86_64) |
| C++ 标准 | (Qt4 默认) | C++11 (`CONFIG += c++11`) |

## 2. zksort.pro 修改

### 2.1 编译选项
```qmake
# 新增 -Wno-narrowing，避免 0xB9 等字节值在 char 数组初始化时触发 narrowing 错误
QMAKE_CXXFLAGS += -Wno-narrowing -Wno-unused-parameter \
    -Wno-unused-variable
```

### 2.2 mqtt 库切换
```qmake
# 原 (使用 ARM stub):
LIBS += -L$$PWD/../3rdparty/mqtt -lmosquittopp -lmosquitto

# 改 (使用系统 x86_64 库):
LIBS += -lmosquittopp -lmosquitto
```

## 3. 第三方库编译 (有源码)

### 3.1 log4qt → x86_64 Qt5
- 路径: `3rdparty/log4qt/`
- 原始: ARM 32-bit ELF, Qt4
- 操作: 修复源码后 `qmake && make` 重编译
- 输出: `3rdparty/log4qt/lib/unix/liblog4qt.so.1.0.0` (x86_64)

### 3.2 uikits → x86_64 Qt5
- 路径: `uikits/`
- 原始: ARM 32-bit ELF, Qt4
- 操作: `uikits.pro` 添加 `QT += widgets`, 修复 QPainterPath 头文件, `qmake && make`
- 输出: `uikits/lib/unix/libuikits.so.1.0.0` (x86_64)

### 3.3 qextserialport & jsoncpp
- 原始即为 x86_64，无需重新编译

## 4. ARM 预编译库 → x86_64 Stub

以下库无源码，为 ARM 32-bit，无法在 x86_64 PC 上使用。创建了空实现 stub：

| 原始库 | Stub 源文件 | 输出 |
|---|---|---|
| `3rdparty/svm/libsvmtool.a` (ARM) | `3rdparty/stubs/svm_stub.cpp` | `libsvmtool.a` (x86_64) |
| `3rdparty/svm/libWheat.a` (ARM) | `3rdparty/stubs/svm_stub.cpp` | `libWheat.a` (x86_64) |
| `3rdparty/encrypt/libtime_encrypt.a` (ARM) | `3rdparty/stubs/encrypt_stub.cpp` | `libtime_encrypt.a` (x86_64) |
| `3rdparty/fToFp/libTEA.a` (ARM) | `3rdparty/stubs/fToFp_stub.cpp` | `libTEA.a` (x86_64) |
| `3rdparty/qrencode/lib/unix/libqrencode.a` (ARM) | `3rdparty/stubs/qrencode_stub.c` | `libqrencode.a` (x86_64) |

> **注意**: mqtt 库 (`libmosquitto.so`, `libmosquittopp.so`) 的 stub 最终未使用，改为链接系统自带版本。在 `zksort.pro` 中已将 `-L../3rdparty/mqtt` 移除。

## 5. 本地开发目录

为替代嵌入式 `/sdcard/` 路径，创建了本地目录结构：

```
/home/zcy/Desktop/CameraTest/sdcard/
├── logo/
├── logs/       ← 实际日志写入 zksort/logs/
├── cnf/
├── bmp/
│   ├── back/
│   └── front/
├── bin/tempdata/
├── ts/
├── wifi/
└── cfg/
```

## 6. 运行时环境

```bash
# 运行 zksort 需要设置 LD_LIBRARY_PATH:
LD_LIBRARY_PATH=\
../3rdparty/log4qt/lib/unix:\
../3rdparty/qextserialport/lib/unix:\
../uikits/lib/unix:\
../3rdparty/jsoncpp/lib/unix:\
/lib/x86_64-linux-gnu:\
/usr/lib/x86_64-linux-gnu

# 需要设置 Qt 平台:
QT_QPA_PLATFORM=xcb
```

## 7. 备份

完整备份保存在: `/home/zcy/Desktop/noerror/`
