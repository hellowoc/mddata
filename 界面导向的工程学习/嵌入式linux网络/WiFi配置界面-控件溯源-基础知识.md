# WiFi 配置界面控件溯源 — 基础知识清单

按在 [[WiFi配置界面-控件溯源-主文档|主文档]] 中首次出现的顺序排列。

---

## 第 1 章出现的

### wpa_supplicant（§1）

Linux 下的 WiFi 客户端守护进程。负责扫描 WiFi 热点、连接认证、加密密钥协商。

当你点"连接"时，底层调用 `wpa_cli`（wpa_supplicant 的命令行客户端）发送连接命令，wpa_supplicant 完成与路由器的四次握手。

### hostapd（§1）

Host Access Point Daemon。Linux 下的软热点守护进程。将无线网卡从"客户端模式"切换为"热点模式"，让其他设备可以连接过来。

### STA vs AP（§1）

- STA = Station（站点），即**客户端模式**——连接到别人的 WiFi
- AP = Access Point（接入点），即**热点模式**——自己变成 WiFi 信号源

---

## 第 2 章出现的

### WifiInterface::instance()（§2）

单例模式（Singleton）。`instance()` 保证全局只有一个 `WifiInterface` 对象，避免多个对象同时操作 WiFi 导致冲突。

```cpp
static WifiInterface* instance(QObject *parent = 0);
```

### QProcess（§2）

Qt 的执行外部进程类。可以启动一个新的进程（如 `sh`），执行命令行程序，等待其完成并读取输出。

```cpp
QProcess process;
process.start("sh", QStringList() << "-c" << "some_command");
process.waitForFinished(5000);              // 最多等 5 秒
QByteArray output = process.readAll();      // 读命令输出
```

`WifiInterface::execCmd()` 内部封装了 `QProcess`。

---

## 第 3.1 节出现的

### QTabWidget::currentChanged(int)（§3.1）

用户切换标签页时发射的信号。参数 `index` 是被选中的标签页索引（从 0 开始）。

---

## 第 3.2 节出现的

### wpa_cli（§3.2）

wpa_supplicant 的命令行接口。`wpa_cli scan` 触发 WiFi 扫描，`wpa_cli scan_results` 获取扫描结果。

### 密码遮盖（§3.2）

`setText("****")` 的手动实现。不是 Qt 标准的 `QLineEdit::Password` 模式。`setType(textType)` 是工程自定义的输入框类型（可能在 `myinputmethod` 中定义），`textType` 可能就是一种密码输入模式。

### QTimer::singleShot(8000, ...)（§3.2）

`QTimer` 的静态方法。设置一个**一次性定时器**——8 秒后调用指定的槽函数。和 `setSingleShot(true) + start()` 等价，但语法更简洁。

这 8 秒不是精确的——它只是"给 WiFi 连接 8 秒时间去完成"，实际连接可能在 2 秒完成（多余的等待无害），也可能超过 8 秒（到时候检查仍是"未连接"）。

### DHCP（§3.2）

Dynamic Host Configuration Protocol。自动从路由器获取 IP 地址、子网掩码、网关、DNS。

当 DHCP 开启时，IP 输入框全部禁用——因为不需要手动配，路由器会自动分配。当 DHCP 关闭时，需要手动输入静态 IP 配置。

### 自定义滑动开关（§3.2）

`ui->dhcpBtn` 类型不是标准 `QCheckBox`，而是工程自定义的 `SwitchBtn`（`mysortwidget/switchbtn.cpp`）。`setToggle(bool)` 和 `isToggled()` 是其自定义方法（标准 QCheckBox 用的是 `setChecked` 和 `isChecked`）。

---

## 第 3.3 节出现的

### SSID（§3.3）

Service Set Identifier。WiFi 热点的名称——就是你搜 WiFi 时看到的那个名字（如 "TP-LINK_ABC"）。

### PSK（§3.3）

Pre-Shared Key。WiFi 密码。WPA/WPA2 加密中使用的预共享密钥。8 位以上才有效（WiFi 协议的要求）。

### MAC 地址（§3.3）

Media Access Control 地址。每个网络设备（手机、笔记本电脑）的硬件唯一标识。

`getApConnDevice()` 返回的是连接到热点的设备的 MAC 地址列表（不是 IP 列表），因为 MAC 地址是最可靠的身份标识——IP 可能会变，MAC 不会。

---

## 第 4 章出现的

### 单例模式（§4）

保证一个类只有一个实例，并提供全局访问点。本工程中 `WifiInterface` 使用单例，因为 WiFi 硬件只有一块，不应该同时有两个对象去操作它。

### execCmd()（§4）

`WifiInterface` 的私有方法（`wifiinterface.h:177`）。用 `QProcess` 执行 shell 脚本并读取输出。如果命令超时（超过 5 秒），认为失败。

### 脚本控制 WiFi（§4）

为什么用 shell 脚本而不是直接调用 C API？因为 Linux WiFi 的配置涉及多个步骤（停止 wpa_supplicant → 修改配置文件 → 重启 wpa_supplicant → 触发重新连接），封装在 shell 脚本中更容易调试和修改。嵌入式开发中常用的模式——脚本做底层操作，C++ 界面做上层调用。

---

## 第 5 章出现的

### wpa_supplicant.conf（§5）

wpa_supplicant 的配置文件。存储 WiFi 网络列表和对应的密码：

```
network={
    ssid="MyWiFi"
    psk="mypassword"
    key_mgmt=WPA-PSK
}
```

`setWifiSsidAndPwd()` 修改的就是这个文件。

### 阻塞调用与 UI 卡顿（§5）

`QProcess::waitForFinished()` 是阻塞的——在子进程完成前，调用它的线程卡住。因为是主线程调用，如果 shell 脚本执行 3 秒，UI 会卡 3 秒。但 WiFi 连接是用户手动操作（低频），短暂卡顿可以接受。

对比：如果这是个每秒执行几十次的操作，就绝对不能阻塞主线程——需要把 QProcess 放到子线程。
