# QSettings — 配置文件读写

## 理解

`QSettings` 是 Qt 提供的配置文件工具类。**指定一个 INI 文件 → 进入某个节 → 读写其中的键值对。**

不需要自己 `fopen`、`fscanf`、`fprintf`，Qt 帮你做了文件 I/O。

## 三步操作

```
1. QSettings setting(文件路径, IniFormat)     ← 指定操作哪个 INI 文件
2. beginGroup("节名")                         ← 进入哪个节（[eth0]）
3. value("键名") / setValue("键名", 值)       ← 读写键值
```

## 所有方法（本工程用到的）

### 构造

| 写法 | 含义 |
|------|------|
| `QSettings setting(CFG_APPSet, QSettings::IniFormat)` | 打开指定路径的 INI 文件 |

`CFG_APPSet` 是宏，展开为配置文件路径字符串。

### 读

| 方法 | 作用 | 例子 |
|------|------|------|
| `beginGroup("节名")` | 进入 `[节名]` 节 | `setting.beginGroup("eth0")` |
| `value("键名")` | 读键值，返回 `QVariant` | `setting.value("ipStr")` |
| `value("键名", 默认值)` | 读键值，不存在则返回默认值 | `setting.value("ipStr", "192.168.0.10")` |
| `endGroup()` | 退出当前节 | `setting.endGroup()` |

`QVariant` 需要通过 `.toString()` / `.toInt()` / `.toDouble()` 转为具体类型。

### 写

| 方法 | 作用 | 例子 |
|------|------|------|
| `setValue("键名", 值)` | 写入键值 | `setting.setValue("ipStr", "192.168.1.100")` |
| `sync()` | 立即写入磁盘 | `setting.sync()` |

`setValue` 只是改内存中的值，`sync()` 才真正落盘。

## INI 文件格式

```ini
[eth0]
ipStr=162.254.129.100
maskStr=255.255.255.0
gatewayStr=162.254.129.1

[eth1]
ipStr=192.168.1.100
maskStr=255.255.255.0
gatewayStr=192.168.1.1
```

## 本工程中的典型用法

### 读取（IPSetWidget::showPage）

```cpp
QSettings setting(CFG_APPSet, QSettings::IniFormat);
setting.beginGroup("eth0");
QString ipStr0 = setting.value("ipStr", "192.168.0.10").toString();
QString maskStr0 = setting.value("maskStr", "255.255.255.0").toString();
setting.endGroup();

ui->IP_LineEdit->setText(ipStr0);     // 显示到界面
ui->Mask_LineEdit->setText(maskStr0);
```

### 写入（IPSetWidget::on_okBtn_clicked）

```cpp
QSettings setting(CFG_APPSet, QSettings::IniFormat);
setting.beginGroup("eth0");
setting.setValue("ipStr", ui->IP_LineEdit->text());     // 从界面读 → 写入配置
setting.setValue("maskStr", ui->Mask_LineEdit->text());
setting.endGroup();
setting.sync();    // 立即落盘

myFlow.getSetting();  // 通知其他模块重新加载配置
```

# 理解

在创建的时候绑定一个ini文件，之后进入这个文件的不同节中，完成对对应键值对的读写和编辑