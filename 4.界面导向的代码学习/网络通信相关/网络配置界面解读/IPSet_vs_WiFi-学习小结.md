# IPSetWidget vs WifiConfigWidget — 学习小结

## 涉及文件

| 页面 | 文件 |
|------|------|
| 网络配置（有线） | `systeminfo/ipsetwidget.h` + `ipsetwidget.cpp`(484行) |
| WiFi 配置（无线） | `systeminfo/wificonfigwidget.h` + `wificonfigwidget.cpp`(308行) |

## 两个页面对比

| | IPSetWidget | WifiConfigWidget |
|------|------------|------------------|
| 在哪 | 远程控制 → Tab 0 | 左侧栏 "WiFi" |
| 管什么 | eth0/eth1 有线网卡 + VPN | WiFi 无线网卡 |
| 结构 | 单页，两栏布局 | QTabWidget（WiFi 模式 / AP 模式） |
| IP 来源 | **静态 IP**，手动写死在 INI 文件 | **DHCP 自动获取** 或手动填 |
| 连接对象 | 固定设备（FPGA/IPC），IP 已知 | 路由器，自动分配 IP |
| 额外功能 | VPN 连接 + ping 保活 | AP 热点模式 + 已连接设备列表 |
| 编辑锁 | `m_bModify`，必须先勾"修改" | WiFi：连接/断开直接可用；AP：直接修改 |

## 两个页面共同用到的基础概念

### Qt 机制

| 概念 | IPset 中 | WiFi 中 |
|------|---------|---------|
| `.ui` + `setupUi` | ✓ | ✓ |
| `ui->xxx` 控件指针 | ✓ | ✓ |
| 命名槽自动连接 `on_xxx_clicked` | ✓ | ✓ |
| 手动 `connect`（线程、定时器） | `pingThread::pingFailSig` | `m_FreshInfoTimer::timeout` |
| `QTimer` 定时刷新 | 无（只读一次配置） | ✓ 4 秒间隔 |
| `QSettings` 读写 INI | ✓ 读写 eth0/eth1 配置 | 无（WiFi 配置由 shell 脚本管） |
| `showPage(bool)` | ✓ 重读配置 | ✓ 启动/停止定时器 |
| 编辑锁 `m_bModify` | ✓ | 无 |
| `delayShow` + `processEvents` | ✓ | 无 |

### 系统操作

| 概念 | IPset 中 | WiFi 中 |
|------|---------|---------|
| `QProcess` 执行命令 | `ping` 网络测试 | 封装成 `execCmd()` 统一调用 |
| `system()` 执行命令 | `openvpn`、`killall` | 无 |
| 阻塞等结果 | `waitForFinished()` | `waitForFinished()` (在 execCmd 内) |
| 超时处理 | 手动 `sleep(1)` × 30 次 | `MAX_EXEC_TIME = 5000ms` |
| 命令出错恢复 | 无 | `deleteLater()` + `new QProcess` |

### C++ 概念

| 概念 | IPset 中 | WiFi 中 |
|------|---------|---------|
| 构造函数初始化列表 | ✓ | ✓ |
| 析构函数 `delete ui` | ✓ | ✓ |
| 成员变量 `m_` 前缀 | `m_bModify`、`m_thread` | `m_FreshInfoTimer`、`m_pwd`、`m_wifiInterface` |
| 静态成员 | 无 | 无 |
| 单例模式 | 依赖 `g_infoWidget()`、`myFlow` | 依赖 `WifiInterface::instance()` |
| 自定义线程类 | `pingThread`（定义在同一 `.h`） | 无 |

### 线程

| 概念 | IPset 中 | WiFi 中 |
|------|---------|---------|
| 子线程 | `pingThread`（每20秒ping一次） | 无子线程 |
| 子线程何时启动 | VPN 连接成功后 `m_thread->start()` | — |
| 子线程何时停止 | `stopPing()` | — |
| 子线程 emit → 主线程槽 | `pingFailSig()` → `on_btn_disconnect_clicked()` | — |

## 学习的知识点链对比

```
IPSetWidget 学习链：
  .ui机制 → 控件类型 → 编辑锁 → QSettings → sync/系统命令
    → QProcess/processEvents → tun0/VPN → pingThread线程
      → emit/connect → sleep机制 → 权限控制

WifiConfigWidget 学习链：
  .ui机制 → 双定时器 → showPage生命周期 → 密码遮盖
    → WiFi单例WifiInterface → execCmd封装QProcess
      → DHCP → STA/AP双模式 → 单按钮切换文字状态
        → 8秒后收尾检查 → MAC地址列表
```

## 两个页面都遵循的规律

1. **界面 = .ui拖 + .cpp写逻辑**，通过 `ui->xxx` 操控
2. **showPage(true/false)** 管后台任务启停，不是管显示
3. **底层用 shell 脚本**（wifiCtrol.sh 或 openvpn/system 命令），Qt 只做上层调用
4. **所有操作在主线程**，不阻塞的方式用 `processEvents` 版 sleep
5. **多语言支持**通过 `retranslateUiWidget()` + `g_myLan()` 实现
6. **在 SysStateWidget 左侧栏中**，作为 QStackedWidget 的两个子页面存在
