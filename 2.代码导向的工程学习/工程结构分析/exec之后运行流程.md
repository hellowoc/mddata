# exec() 之后的运行流程

## 一、exec() 刚进入时

```cpp
int execResult = a.exec();  // ← 进入事件循环

// exec() 内部：
while (true) {
    QEvent *event = getNextEvent();   // 阻塞等待事件
    dispatch(event);                   // 分发给对应对象处理
    if (quit) break;
}
```

此时屏幕上主窗口已显示，信息提示框居中，但一切是**静止**的——唯一即将到来的事件是：**100ms 定时器**。

---

## 二、100ms 后 — 第一个事件

```
exec() 内部:
  getNextEvent() → 定时器到期
    ↓
  dispatch → QTimer::timerEvent()
    ↓
  emit timeout()
    ↓
  connect 链 → ZKSort::dccryptCheck()   ← ★ 第一个业务代码
```

`dccryptCheck()` 是 exec() 之后第一个被调用的业务函数。

---

## 三、dccryptCheck() → communication() — 硬件握手

```cpp
void ZKSort::dccryptCheck()
{
    mytimer->stop();   // 定时器只触发一次
    // ... 加密校验 ...
    communication();   // 校验通过 → 开始通信
}
```

```cpp
void ZKSort::communication()
{
    if (struGsh.serialPortOpenState == -1) return;
    if (myFlow.startWithoutCnf == 1) {
        startEnterMainPage();   // 无配置启动 → 弹窗确认 → 进主页
        return;
    }

    g_infoWidget().setLabelText(g_myLan().msg_communicating); // "通信中..."
    g_infoWidget().delayShow();

    int ret2 = myFlow.resetCommunication();  // ★ 和 FPGA/接口板硬件握手

    if (ret2 == 0) {
        startEnterMainPage();         // 成功 → 进主页
    } else {
        g_MainManager().ShowWidget(SM_SELF_CHECK_WIDGET);  // 失败 → 自检页
    }
}
```

---

## 四、startEnterMainPage() — 进主页

```cpp
void ZKSort::startEnterMainPage()
{
    g_infoWidget().setLabelText(g_myLan().msg_system_init);  // "系统初始化..."
    g_infoWidget().delayShow();

    myFlow.initSendAllParams();   // ★ 把所有配置参数下发到 FPGA/传感器

    for (int j = 0; j < 15; j++) {     // 等 IPC 相机模型加载（最多 15 秒）
        myFlow.sleep(1);
        if (struIpcShare.allIpcModelLoadOk == 2) break;
    }

    g_MainManager().CreateWidgets();   // ★ 创建所有页面 widget

    g_infoWidget().hide();

    g_MainManager().ShowWidget(SM_MAIN_PAGE_WIDGET_NEW); // 显示主页
}
```

---

## 五、UI 变化

```
启动前:
┌──────────────────────────┐
│  TopWidget               │
├──────────────────────────┤
│  QStackedWidget (空白)   │
├──────────────────────────┤
│  BottomWidget (隐藏)     │
└──────────────────────────┘

启动后:
┌──────────────────────────┐
│  TopWidget               │
├──────────────────────────┤
│  NewMainPage             │  ← 用户看到的主页
│  [开始][停止][模式][设置]  │
├──────────────────────────┤
│  BottomWidget (显示)     │  ← 底部按钮出现
└──────────────────────────┘
```

---

## 六、正常运行状态

dccryptCheck() 末尾十几个 connect 全部接好：

```cpp
connect(myUpdateStatusThread, SIGNAL(sAlarmTurnOffSig()), this, SLOT(alarmTurnOffFeedSlt()));
connect(myUpdateStatusThread, SIGNAL(sWipeStartSig()),    this, SLOT(myWipeStartSlt()));
connect(myUpdateStatusThread, SIGNAL(sLostFpgaPowerOffSig()), this, SLOT(lostFpgaPowerOffSlt()));
// ... 十几个 connect
```

之后 exec() 循环中永续运转：

```
exec() 事件循环中：
  ├── 定时器超时 → 查询硬件状态 → 有变化 emit 信号 → 槽函数处理
  ├── 串口数据到达 → 解析 → 更新界面
  ├── 用户触摸屏幕 → 背光重置 → 事件分发 → 页面切换/参数修改
  ├── 网络数据 → 远程控制 → 升级/查询
  └── ... 无穷无尽 ...
```

---

## 七、完整时序图

```
a.exec()
  │
  ├── (100ms 静默)
  │
  ├── QTimer::timeout → dccryptCheck()
  │   │                   ├── 加密校验通过
  │   │                   └── communication()
  │   │                         ├── resetCommunication() ← 硬件握手
  │   │                         ├── 成功 → startEnterMainPage()
  │   │                         └── 失败 → 自检页
  │   │
  │   └── dccryptCheck() 结尾: connect × 15 ← 接好所有信号槽
  │
  ├── startEnterMainPage()
  │   ├── initSendAllParams()     ← 下发配置
  │   ├── 等 IPC 加载 (最多 15s)
  │   ├── CreateWidgets()         ← 创建所有页面
  │   └── ShowWidget(主页)        ← 用户看到主界面
  │
  └── 正常运行
      ├── 串口事件 → 更新数据
      ├── 触摸事件 → 用户操作
      ├── 定时器 → 后台轮询
      └── 报警信号 → 应急处理
```

---

## 一句话

`a.exec()` 进入后程序不是空闲等待，而是进入了**事件驱动的状态机**。100ms 定时器是第一个多米诺骨牌——它触发 dccryptCheck → communication → startEnterMainPage，一步步把页面创建出来、配置下发、信号槽全接好。之后在事件循环中永续运转，直到窗口关闭。
