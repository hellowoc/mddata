# IPSetWidget 学习路径

从零开始，通过逐行阅读 `IPSetWidget` 的代码，边问边学，最终完全理解这个页面。

## 学习的知识点链

### 第一层：Qt 界面机制

**问**：`.ui` 文件和 `.cpp` 文件是什么关系 → **答**：uic 编译 `.ui` 生成 `ui_xxx.h`，里面有一个类，包含所有控件的指针和一个 `setupUi()` 函数。

**问**：`ui->xxx` 是什么 → **答**：`ui` 是 `Ui::IPSetWidget*` 类型的指针，指向自动生成的对象。`ui->xxx` 是该对象中名为 `xxx` 的控件指针（如 `QPushButton*`）。

**问**：`namespace Ui { class IPSetWidget; }` 是什么 → **答**：C++ 的命名空间，用于前向声明，避免 `.h` 文件包含自动生成的大文件，只让 `.cpp` 去包含。

### 第二层：C++ 构造函数

**问**：`IPSetWidget(QWidget *parent) : basewidget(parent), ui(new Ui::IPSetWidget)` 是什么语法 → **答**：构造函数初始化列表。冒号后面先初始化父类和成员变量，再执行 `{}` 里的函数体。

**问**：构造函数做什么 → **答**：只做三件事——`setupUi()` 创建控件、设初始值（`m_bModify = false`）、连接信号（`connect`）。不触发任何动作。

**问**：析构函数做什么 → **答**：`delete ui`。Qt 的 parent/child 机制会自动销毁子控件。

### 第三层：控件类型和操作

**问**：这个页面有几种控件类型 → **答**：四种——`myLineEdit`（输入框）、`QPushButton`（按钮）、`QLabel`（标签）、`QCheckBox`（复选框）。

**问**：各自有什么操作函数 → **答**：
- `myLineEdit`：`setType()`、`setText()`、`text()`、`setEnabled()`、`hide()`
- `QPushButton`：`setVisible()`、`setEnabled()`、`setText()`、`hide()` + 命名槽 `on_xxx_clicked()`
- `QLabel`：`setText()`、`setAlignment()`、`hide()`
- `QCheckBox`：`setChecked()`、`setVisible()`、`setText()`

**问**：`setType(textType)` 里的 `textType` 是什么 → **答**：`inputType` 枚举值，决定点输入框时弹出什么类型的软键盘（数字键盘/全键盘/密码键盘）。

### 第四层：编辑锁机制

**问**：`m_bModify` 是什么 → **答**：`m_` 前缀表示 member（成员变量）。控制 IP 输入框是否可编辑——`false` 时灰色锁定，`true` 时解锁。防止误触修改关键网络配置。

**问**：流程是怎样的 → **答**：初始 `false` → 勾选"修改"复选框 → `true` → 编辑 → 点"应用"保存 → 恢复 `false` 重新锁定。

### 第五层：配置读写

**问**：`QSettings setting(CFG_APPSet, QSettings::IniFormat)` 在做什么 → **答**：创建一个操作指定 INI 文件的工具对象。

**问**：怎么读写 → **答**：三步——`beginGroup("节名")` 进入节 → `value("键名")` 读 / `setValue("键名", 值)` 写 → `endGroup()` 退出。写完后 `sync()` 立即落盘。

### 第六层：Linux 命令执行

**问**：`QProcess m_cmd; m_cmd.start("ping ...")` 为什么不用 `system()` → **答**：`QProcess` 能读到命令的输出文本（`readAll()`），`system()` 只能拿到退出码。此处需要搜输出中有没有 `"ttl"` 来判断 ping 通没通。

**问**：`system()` 和 `QProcess` 的使用场景 → **答**：不需要返回值用 `system()`（如 `system("killall openvpn")`）。需要读输出用 `QProcess` + `waitForFinished()` + `readAll()`。后台长时间跑用 `system("命令 &")`。

### 第七层：事件循环与 delayShow

**问**：`delayShow()` 里的 `processEvents` 是干什么的 → **答**：事件循环一次只能做一件事。当前槽函数没返回，`show()` 排进队列的绘制事件不会被处理。`processEvents` 强制事件循环立刻去处理队列里的积压事件——把窗口真正画出来。

**问**：为什么不用 `sleep()` → **答**：`sleep()` 完全卡死线程，什么都做不了。`processEvents` 版等法让窗口在等的时候还能画出来。

### 第八层：网络概念

**问**：`tun0` 是什么 → **答**：OpenVPN 创建的虚拟网卡，没有物理硬件，但对程序和操作系统来说跟 eth0 一样可以用。

**问**：eth0、eth1、lo、tun0 的关系 → **答**：
- eth0/eth1：物理网口，插网线连接 FPGA/IPC
- lo (127.0.0.1)：本地回环，自己跟自己通信，天生就有
- tun0：虚拟网卡，OpenVPN 创建，走加密隧道连云端

**问**：ping 通了就能通信吗 → **答**：对。ping 验证了 IP 层可达——网线对、IP 对、路由通。这三样确认了，UDP 就能通。不需要 I2C 那样的 start/stop/ACK 流程。

### 第九层：VPN 连接流程

**问**：点击"连接"按钮后发生了什么 → **答**：
1. 检查 `tun0` 是否已存在（避免重复启动 OpenVPN）
2. 不存在 → `system("openvpn --config zk.ovpn &")` 后台启动
3. `connetToServer()` 轮询 30 次，每次等 1 秒，看 `tun0` 是否出现
4. 出现 → VPN 连接成功 → 启动 `pingThread` 保活
5. 30 秒仍未出现 → 连接失败 → `killall openvpn`

**问**：为什么要 sleep(1) 等 → **答**：OpenVPN 不是瞬间启动的——握手、验证证书、获取 IP 需要几秒到十几秒。不等的话循环几微秒跑完 30 次，全部失败，而此时 OpenVPN 还在后台连接中。

### 第十层：pingThread 线程

**问**：`m_thread` 是什么 → **答**：`IPSetWidget` 的私有成员，类型 `pingThread*`。不是全局单例，`m_` 前缀表示 member。

**问**：什么时候启动 → **答**：VPN 连接成功后 `m_thread->start()`。构造函数只 `new` 了对象，没调 `start()`。

**问**：做什么 → **答**：每 20 秒 ping 一次 `10.18.0.1`（VPN 网关）。ping 不通 → `emit pingFailSig()` → Qt 跨线程调度 → 主线程 `on_btn_disconnect_clicked()` 自动断开 VPN。

### 第十一层：IP 查找流程

**问**：`QNetworkInterface::allInterfaces()` 在做什么 → **答**：遍历系统所有网卡，找名字叫 `"tun0"` 的。

**问**：找到网卡后怎么拿 IP → **答**：`interface.addressEntries()` 获取该网卡绑定的所有地址，遍历找 IPv4 的那个，读 `address.ip().toString()`。

**问**：为什么只取 IPv4 → **答**：VPN 服务器分配的是 IPv4。IPv6 地址（fe80::xxx）是系统自动生成的本机链路地址，跟 VPN 无关。

### 第十二层：确认对话框

**问**：`myMessageBox` 和 `g_infoWidget()` 有什么区别 → **答**：
- `g_infoWidget()`：底部提示条，几秒自动消失，不阻塞，不需要用户确认
- `myMessageBox`：模态对话框，`exec()` 阻塞等待用户点"确定"或"取消"

**问**：`exec()` 怎么做到的 → **答**：内部起一个临时 `QEventLoop`，显示对话框后卡住，等用户点击按钮 → `loop.quit()` → 返回结果。

### 第十三层：emit 信号

**问**：`emit pingFailSig()` 发射给谁 → **答**：给 `this`（IPSetWidget 自己）。子线程 pingThread emit → Qt 自动跨线程调度 → 主线程 IPSetWidget 的槽执行。

**问**：`connect` 怎么连的 → **答**：构造函数里手动 `connect(m_thread, SIGNAL(pingFailSig()), this, SLOT(on_btn_disconnect_clicked()))`。`.ui` 里的按钮则用命名槽自动连接（`on_xxx_clicked`）。

### 第十四层：myFlow.sleep 实现

**问**：`myFlow.sleep(1)` 怎么实现的 → **答**：跟 `delayShow` 一样——`while + processEvents` 忙等循环，不是真休眠。每次循环让事件循环处理积压事件，所以 UI 不会卡死。

**问**：`myFlow` 是什么 → **答**：`GlobalFlow` 的全局单例，整个工程的总调度中心。

### 第十五层：页面权限控制

**问**：`struGsh.bRunMode != RUNMODE_ENGINEER` 在干什么 → **答**：判断当前模式。工程师模式下所有操作按钮隐藏——只能查看网络配置，不能修改。

## 学习路径总结

```
.ui 文件机制
  → ui->xxx 访问控件
    → 控件有哪些类型、各有什么操作
      → 编辑锁 m_bModify
        → QSettings 读写 INI 配置
          → sync() 落盘 → getSetting() 使生效
            → system() 执行 Linux 命令
              → QProcess 读命令输出
                → processEvents 事件循环
                  → delayShow / sleep 忙等
                    → tun0 虚拟网卡 / VPN 加密隧道
                      → ping 连通性测试
                        → pingThread 保活线程
                          → emit 信号 / connect 槽
```

15 层，每一层在前一层的基础上追问，直到追到 Qt 的固有类或 Linux 系统调用为止。
