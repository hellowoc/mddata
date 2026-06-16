# QSettings — 应用程序配置存储

## 它是干什么的

```cpp
QSettings setting(CFG_APPSet, QSettings::IniFormat);
int onoff_cnts = setting.value("onoffcnts", 0).toInt();
setting.setValue("onoffcnts", ++onoff_cnts);
setting.sync();
```

这段代码做的事：

```
setting.value("onoffcnts", 0)   → 从配置文件里读 "onoffcnts" 的值
                                  第一次开机时文件不存在，返回默认值 0
setting.setValue(...)           → 把新值写回内存
setting.sync()                  → 把内存中的值刷到磁盘文件
```

---

## 类比：不同平台的配置存储

| 平台/框架 | 配置存储方式 |
|-----------|-------------|
| Windows 原生 | 注册表（RegOpenKeyEx / RegSetValueEx） |
| .NET | app.config / web.config |
| Qt | QSettings |
| 纯 Linux | 手动写文件到 /etc 或 ~/.config |

QSettings 就是 Qt 给你的"不需要写文件 I/O 代码的配置读写器"。

---

## 存储格式

QSettings 支持两种后端：

| 格式 | 示例 |
|------|------|
| `QSettings::IniFormat` | 经典 INI 文件，人类可读 |
| `QSettings::NativeFormat` | Windows → 注册表；Linux → `~/.config/...` |

这个项目用 `IniFormat`，产物是一个 `.ini` 文件：

```ini
; /sdcard/cnf/xxx.ini
[General]
onoffcnts=1523
vendorIndex=0
layertotal=1
```

---

## 关键 API

| 方法 | 作用 |
|------|------|
| `value("key", 默认值)` | 读配置，如果 key 不存在返回默认值 |
| `setValue("key", 值)` | 写配置（先存内存） |
| `sync()` | 把内存中的变更刷到磁盘文件 |
| `allKeys()` | 列出所有配置项 |
| `remove("key")` | 删除某个配置项 |
| `clear()` | 清空所有配置 |
| `contains("key")` | 检查是否存在某配置 |

---

## 在这个项目中的用途

记录设备的**开机次数**：

```cpp
QSettings setting(CFG_APPSet, QSettings::IniFormat);
int onoff_cnts = setting.value("onoffcnts", 0).toInt();  // 上次是第几次
setting.setValue("onoffcnts", ++onoff_cnts);              // 这次 +1
setting.sync();                                            // 写入磁盘
logger()->info("power on: %1 times", onoff_cnts);          // 记日志
```

这个数字不参与任何业务逻辑，纯粹是**运维统计**——售后工程师可以通过开机次数判断设备的使用强度，辅助判断磨损情况。

---

## 写入的完整链路

```
setValue("onoffcnts", 1524)
    │
    ▼
QSettings 内部 QHash 缓存
    │  (此时还没写到磁盘)
    │
    ▼
sync()
    │
    ▼
QFile 打开 CFG_APPSet 对应的 .ini 文件
    → 把 [General]\nonoffcnts=1524 写入
    → ::write() 系统调用
    → ::close()
    ↓
磁盘上的 INI 文件已更新
```

---

## 为什么需要 sync()

如果没有 `sync()`：

```
程序运行中 → setValue → 内存中改了
                     → 突然断电（工业设备常见情况）
                     → 磁盘上的开机次数没更新
                     → 下次开机读到的还是旧值，丢失了这一次的记录
```

所以 `sync()` 在工业设备上是必须的——每写一次关键配置就立刻刷盘。

---

## 一句话

QSettings 就是 Qt 的"自动读写配置文件"工具——你给它 key 和默认值，它负责去磁盘文件（或注册表）里读回来；你 `setValue` 再 `sync`，它负责写到磁盘。省去了手动 open/write/close 的繁琐。
