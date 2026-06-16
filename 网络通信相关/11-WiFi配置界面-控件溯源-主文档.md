# WiFi 配置界面 — 控件溯源

## 涉及的文件

| 文件 | 内容 |
|------|------|
| `CameraTest(aiuse)/zksort/systeminfo/wificonfigwidget.h` | WiFi 配置页类定义 |
| `CameraTest(aiuse)/zksort/systeminfo/wificonfigwidget.cpp` | 全部控件逻辑（309 行） |
| `CameraTest(aiuse)/zksort/systeminfo/wifiinterface.h` | WiFi 后端接口类定义 |
| `CameraTest(aiuse)/zksort/systeminfo/wifiinterface.cpp` | 通过 shell 脚本操作 wpa_supplicant/hostapd |
| `CameraTest(aiuse)/zksort/systeminfo/sysstatewidget.cpp:19` | 在 SysStateWidget 中注册为按钮 `btn_wifi_set` |

---

## 1. 这个页面是干什么的

WiFi 配置页管理 Linux 工控机的**无线网络**。有两种模式：

| 模式 | Linux 底层 | 用途 |
|------|-----------|------|
| **WiFi 模式**（客户端） | `wpa_supplicant` | 工控机作为客户端，连接到工厂的 WiFi 路由器 |
| **AP 模式**（热点） | `hostapd` | 工控机自己变成一个 WiFi 热点，手机/电脑连接它 |

页面是一个 **QTabWidget**，两个标签页各管理一种模式。

---

## 2. 页面生命周期

```
构造函数（wificonfigwidget.cpp:5-56）
  ├── 创建两个定时器：
  │   ├── m_FreshInfoTimer → updateWiFiConnState()   每4秒刷新 WiFi 连接状态
  │   └── m_updateConnDevTimer → updateConnDevice()  每4秒刷新已连接设备列表
  ├── 设置输入框类型（密码框、IP 输入框）
  ├── 获取 WifiInterface 单例 m_wifiInterface
  ├── 默认设为 WiFi 模式
  └── 隐藏模式切换按钮

showPage(true)
  → 根据当前模式启动对应定时器（4秒间隔）
    ├── WiFi模式 → m_FreshInfoTimer->start(4000)
    └── AP模式   → m_updateConnDevTimer->start(4000)

showPage(false)
  → 停止两个定时器
```

**线程**：全部在主线程。WiFi 操作通过 `QProcess` 执行 shell 命令，命令执行期间主线程会稍等（最多 5 秒）。

---

## 3. 控件逐一溯源

### 3.1 标签页切换 — tabWidget

**创建**：`wificonfigwidget.ui`（QTabWidget，两个标签页）

**切换→槽**：`on_tabWidget_currentChanged(int index)`（第 256-288 行）

```cpp
void WifiConfigWidget::on_tabWidget_currentChanged(int index)
{
    current_mode = index;

    if (current_mode == WifiInterface::WIFI)       // WiFi 模式标签
    {
        m_wifiInterface->setWifiMode(WifiInterface::WIFI);   // 切换底层模式
        updateWiFiConnState();                                // 刷新连接状态
        m_updateConnDevTimer->stop();                         // 停止 AP 定时器
        m_FreshInfoTimer->start(4000);                        // 启动 WiFi 定时器
    }
    else                                           // AP 模式标签
    {
        m_wifiInterface->setWifiMode(WifiInterface::AP);
        updateApIpAndSsidAndPsk();                            // 刷新 AP 配置显示
        updateConnDevice();                                   // 刷新已连接设备
        m_FreshInfoTimer->stop();                             // 停止 WiFi 定时器
        m_updateConnDevTimer->start(4000);                    // 启动 AP 定时器
    }
}
```

**入参**：`index` — 0=WiFi模式标签，1=AP模式标签。

---

### 3.2 WiFi 模式标签（Tab 0）

#### 扫描按钮 — updateBtn

**创建**：`wificonfigwidget.ui`

**点击→槽**：`on_updateBtn_clicked()`（第 106-108 行）

```cpp
void WifiConfigWidget::on_updateBtn_clicked()
{
    this->updateWifiList();   // 刷新 WiFi 列表
}

void WifiConfigWidget::updateWifiList()                    // 第153-158行
{
    QStringList wifiList = m_wifiInterface->getWifiScanResult();  // 调用底层扫描
    ui->ssidBox->clear();
    ui->ssidBox->addItems(wifiList);                      // 填入下拉框
}
```

**操作链路**：点击"刷新" → `getWifiScanResult()` → 底层执行 shell 脚本 → 调用 `wpa_cli scan` + `wpa_cli scan_results` → 解析返回的 WiFi 列表。

#### WiFi 列表下拉框 — ssidBox

**创建**：`wificonfigwidget.ui`

**数据来源**：`updateWifiList()` 填入扫描结果。

#### 密码输入框 — pwdEdit

**创建**：`wificonfigwidget.ui`，设为密码模式（`setType(textType)`）

**文本变化→槽**：`on_pwdEdit_textChanged()`（第 112-117 行）

```cpp
void WifiConfigWidget::on_pwdEdit_textChanged(const QString& pwd)
{
    if (pwd == "****" || pwd.isEmpty())   // 跳过占位符和空输入
        return;
    m_pwd = pwd;                           // 保存真实密码
    ui->pwdEdit->setText("****");          // 用 **** 遮盖显示
}
```

**密码遮盖的实现**：不是用标准 `QLineEdit::setEchoMode(Password)`，而是手动替换为 `****`。真实密码存在 `m_pwd` 成员变量中。

#### 连接/断开按钮 — connect

**创建**：`wificonfigwidget.ui`

**点击→槽**：`on_connect_clicked()`（第 290-308 行）

```cpp
void WifiConfigWidget::on_connect_clicked()
{
    if (按钮当前文字 == "连接")
    {
        // === 连接 WiFi ===
        ui->connectState->setText("连接中...");     // 状态提示
        m_FreshInfoTimer->stop();                    // 暂停定时器

        m_wifiInterface->setWifiSsidAndPwd(          // 1. 设置 SSID 和密码
            ui->ssidBox->currentText(), m_pwd);
        m_wifiInterface->connectWifi(                // 2. 发起连接
            ui->ssidBox->currentText());

        QTimer::singleShot(8000, this, SLOT(singleShotFunc()));
        //                     ↑ 8秒后调用 singleShotFunc() 检查连接结果
    }
    else
    {
        // === 断开 WiFi ===
        ui->connectState->setText("断开");
        m_FreshInfoTimer->stop();

        m_wifiInterface->disconnectWifi();           // 断开连接

        QTimer::singleShot(3000, this, SLOT(singleShotFunc()));
        //                     ↑ 3秒后恢复
    }
    ui->connect->setEnabled(false);                  // 操作期间禁用按钮
    ui->updateBtn->setEnabled(false);
}
```

**为什么用 8 秒延迟**：WiFi 连接需要时间（扫描→认证→关联→DHCP 获取 IP）。8 秒后调用 `singleShotFunc()` 检查连接状态。

**`singleShotFunc()`**（第 119-131 行）：

```cpp
void WifiConfigWidget::singleShotFunc()
{
    this->updateWiFiConnState();              // 检查连接状态

    if (连接成功)
    {
        ui->dhcpBtn->setToggle(true);         // 自动开启 DHCP
        this->on_dhcpBtn_toggled(true);
    }

    m_FreshInfoTimer->start(4000);            // 恢复定时器
    ui->connect->setEnabled(true);            // 恢复按钮
    ui->updateBtn->setEnabled(true);
}
```

#### DHCP 开关 — dhcpBtn

**创建**：`wificonfigwidget.ui`（自定义 SwitchBtn 滑动开关）

**切换→槽**：`on_dhcpBtn_toggled(bool checked)`（第 168-192 行）

```cpp
void WifiConfigWidget::on_dhcpBtn_toggled(bool checked)
{
    if (checked)
    {
        // DHCP 开启 → 禁用手动 IP 输入框
        ui->ipEdit->setEnabled(false);
        ui->maskEdit->setEnabled(false);
        ui->gateEdit->setEnabled(false);
        ui->dnsEdit->setEnabled(false);
        ui->confirmBtn->setEnabled(false);

        m_wifiInterface->setDhcpState(true);   // 通知底层开启 DHCP
    }
    else
    {
        // DHCP 关闭 → 允许手动输入 IP
        ui->ipEdit->setEnabled(true);
        // ...

        m_wifiInterface->setDhcpState(false);  // 通知底层关闭 DHCP
    }
    this->updateWifiNetInfo();                 // 刷新显示当前 IP 配置
}
```

#### 手动 IP 配置 — ipEdit / maskEdit / gateEdit / dnsEdit + 应用按钮

**创建**：`wificonfigwidget.ui`

**应用点击→槽**：`on_confirmBtn_clicked()`（第 194-202 行）

```cpp
void WifiConfigWidget::on_confirmBtn_clicked()
{
    m_wifiInterface->setIpCfg(WIFI::IpCfg(
        ui->ipEdit->text(),     // IP 地址
        ui->maskEdit->text(),   // 子网掩码
        ui->gateEdit->text(),   // 默认网关
        ui->dnsEdit->text()     // DNS 服务器
    ));
    this->updateWifiNetInfo();  // 刷新显示
}
```

**数据读取**：`updateWifiNetInfo()`（第 160-166 行）从底层读取当前 IP 配置并显示在输入框中。

---

### 3.3 AP 模式标签（Tab 1）

#### SSID / IP / 密码 — apSsidEdit / apIpEdit / apPskEdit

**创建**：`wificonfigwidget.ui`

**应用点击→槽**：`on_apApplyBtn_clicked()`（第 231-237 行）

```cpp
void WifiConfigWidget::on_apApplyBtn_clicked()
{
    m_wifiInterface->setApSsid(ui->apSsidEdit->text());     // 设置热点名称
    m_wifiInterface->setApIp(ui->apIpEdit->text());         // 设置热点 IP
    if (ui->apPskEdit->text().size() >= 8)                   // 密码至少 8 位
        m_wifiInterface->setApPsk(ui->apPskEdit->text());
    this->updateApIpAndSsidAndPsk();                         // 刷新显示
}
```

**默认密码**：构造函数第 39 行写死 `"123456789"`。

#### 恢复默认按钮 — apRestoreBtn

**点击→槽**：`on_apRestoreBtn_clicked()`（第 239-242 行）

```cpp
void WifiConfigWidget::on_apRestoreBtn_clicked()
{
    m_wifiInterface->setWifiMode(WifiInterface::AP);  // 重新应用 AP 模式
    this->updateApIpAndSsidAndPsk();                   // 刷新显示默认值
}
```

#### 已连接设备列表 — apListWidget

**创建**：`wificonfigwidget.ui`（QListWidget）

**刷新函数**：`updateConnDevice()`（第 250-254 行）— 每 4 秒调用一次

```cpp
void WifiConfigWidget::updateConnDevice()
{
    QStringList devs = m_wifiInterface->getApConnDevice();  // 获取 MAC 列表
    ui->apListWidget->clear();
    ui->apListWidget->addItems(devs);
}
```

---

### 3.4 隐藏的模式切换按钮 — switchModePtn

**创建**：`wificonfigwidget.ui`，默认隐藏（第 53 行 `hide()`）

**点击→槽**：`on_switchModePtn_clicked()`（第 204-228 行），通过 `WifiInterface::MODE` 在 WiFi/AP 间切换。

按钮隐藏的原因是：页面本身已经是 QTabWidget 两个标签页，不需要额外的切换按钮。

---

## 4. 底层实现：WifiInterface

**文件**：`CameraTest(aiuse)/zksort/systeminfo/wifiinterface.h`

`WifiInterface` 是一个**单例**（`instance()` 静态方法）。所有 WiFi 操作最终通过执行 shell 脚本实现：

```cpp
class WifiInterface : public QObject
{
    QProcess *m_process;                              // Qt 进程类
    const QString WIFI_CONFIG_FILE = "/sdcard/wifi/wifiCtrol.sh";  // WiFi 控制脚本
    const int MAX_EXEC_TIME = 5000;                   // 命令超时 5 秒

    QByteArray execCmd(const QString& cmd);           // 执行 shell 命令并返回输出
};
```

**每个方法的底层实现**都是调用 shell 脚本，例如：

```
getWifiScanResult()  → execCmd("sh wifiCtrol.sh scan")
connectWifi(ssid)    → execCmd("sh wifiCtrol.sh connect " + ssid)
getWiFiState()       → execCmd("sh wifiCtrol.sh status")
setApSsid(ssid)      → execCmd("sh wifiCtrol.sh ap_ssid " + ssid)
```

`wifiCtrol.sh` 内部操作 Linux 的标准 WiFi 工具：
- WiFi 模式：`wpa_supplicant` + `wpa_cli`（WiFi 客户端守护进程）
- AP 模式：`hostapd` + `dnsmasq`（热点守护进程 + DHCP 服务器）

---

## 5. 数据流图

```
用户点击"连接"
    │
    ↓ 主线程
on_connect_clicked()
    ├── m_wifiInterface->setWifiSsidAndPwd(ssid, pwd)
    │     → execCmd("sh wifiCtrol.sh set_ssid " + ssid + " " + pwd)
    │       → 写入 wpa_supplicant.conf
    │
    ├── m_wifiInterface->connectWifi(ssid)
    │     → execCmd("sh wifiCtrol.sh connect " + ssid)
    │       → wpa_cli select_network / reassociate
    │
    └── QTimer::singleShot(8000, singleShotFunc)
          │ 8 秒后
          ↓
          updateWiFiConnState()
            → execCmd("sh wifiCtrol.sh status")
              → wpa_cli status (解析 CONNECTED/DISCONNECTED)
            → 更新 ui->connectState 标签
            → 连接成功 → 自动开 DHCP
```

---

## 6. 线程

全部在主线程。`QProcess::execute()` 是**阻塞调用**——shell 命令期间主线程卡住。但 WiFi 命令通常 1~3 秒完成，且 WiFi 设置不是高频操作（用户手动操作，频率极低），所以可以接受短暂卡顿。

`QTimer` 定时刷新也在主线程，通过 `m_FreshInfoTimer->start(4000)` 每 4 秒检查一次连接状态。
