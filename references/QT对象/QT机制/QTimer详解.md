# QTimer — Qt 的定时器

## 它是什么

```cpp
QTimer *mytimer = new QTimer(this);
connect(mytimer, SIGNAL(timeout()), this, SLOT(dccryptCheck()));
mytimer->start(100);
```

三行代码完成一件事：**100 毫秒后，调用一次 `dccryptCheck()`。**

---

## 工作机制

```
mytimer->start(100)
    │
    ▼
Qt 向操作系统申请一个定时器："100ms 后叫醒我"
    │
    │  (100ms 内程序正常做别的事，定时器不占 CPU)
    │
    ▼
100ms 到了 → 操作系统发出 timer 事件
    │
    ▼
a.exec() 事件循环拿到 timer 事件
    │
    ▼
QTimer 发射 timeout() 信号
    │
    ▼
connect → dccryptCheck() 被调用
    │
    ▼
dccryptCheck() 里的第一行：mytimer->stop();
    → 停止定时器，只触发这一次
```

---

## 两种模式

| 模式 | 写法 | 行为 |
|------|------|------|
| 一次性 | `start(100)` + 槽里 `stop()` | 100ms 后触发一次 |
| 重复 | `start(100)` 不 stop | 每隔 100ms 触发一次 |

本项目 `mytimer` 用的是一次性模式——它在 `dccryptCheck()` 第一行就 `stop()` 了自己，目的是 **"延迟 100ms 后再执行加密校验"**。

为什么延迟 100ms？因为构造函数还没有执行完，主窗口还没有完全初始化好，此时直接做加密校验太早。100ms 的延迟让构造函数先跑完、事件循环先启动起来。

---

## 核心 API

| 方法 | 作用 |
|------|------|
| `start(ms)` | 启动定时器，`ms` 毫秒后触发 timeout |
| `stop()` | 停止定时器 |
| `setSingleShot(true)` | 设为单次触发（自动 stop） |
| `isActive()` | 定时器是否在运行 |
| `setInterval(ms)` | 修改时间间隔 |
| `remainingTime()` | 距离下次触发还有多少毫秒 |

---

## signal/slot 连接

```cpp
connect(mytimer,       // 信号源
        SIGNAL(timeout()),  // 信号：时间到了
        this,          // 接收者
        SLOT(dccryptCheck())); // 槽函数：时间到了就调这个
```

`QTimer` 只发射一个信号：`timeout()`。唯一的作用就是**时间到了通知你**。

---

## 本项目中两个定时器的区别

| 定时器 | 用途 | 启动时机 | 停止时机 |
|--------|------|---------|---------|
| `mytimer` | 延迟 100ms 后执行加密校验 | 构造函数末尾 | dccryptCheck() 第一行 |
| `mytimerWaitFpgaMode` | 等待 FPGA 模式切换（20 秒超时） | FPGA 模式异常时 | waitSetFpgaMode() 第一行 |

`mytimerWaitFpgaMode` 在构造函数里只创建不启动：
```cpp
mytimerWaitFpgaMode = new QTimer(this);
connect(mytimerWaitFpgaMode, SIGNAL(timeout()), this, SLOT(waitSetFpgaMode()));
// 没有 start() — 只在需要时才启动
```

---

## 底层实现

```
QTimer::start(100)
    │
    ▼
Qt 内部维护一个定时器列表，按到期时间排序
    │
    ▼
a.exec() 事件循环内部：
    while(true) {
        int nextTimeout = findNextTimer();    // 最近一个定时器多久后到期
        int timeout = min(nextTimeout, 其他事件等待时间);
        poll(fds, timeout);                   // 阻塞等待，最多等 timeout 毫秒
        // 定时器到期 → 发射 timeout() 信号
    }
```

Linux 底层用的是 `poll()` / `epoll()` 的 timeout 参数，不占用 CPU 轮询。

---

## 一句话

QTimer = "定时闹钟"。设好时间，时间到了自动通过信号槽通知你去做事。你可以设它只响一次还是反复响。
