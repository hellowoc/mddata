# `a.exec()` — Qt 事件循环

## 这行代码做了什么

```cpp
int execResult = a.exec();
```

**`exec()` 启动了一个死循环，程序从此"卡"在这一行。** 但这个"卡"不是卡死——它在不停地接收、分发、处理事件，直到窗口关闭才退出。

---

## 如果没有这个死循环会怎样

```cpp
int main() {
    zkApplication a(argc, argv);
    ZKSort w;
    w.show();
    return 0;   // ← main 直接结束，程序退出
}
// 窗口一闪而逝，什么都看不到
```

`w.show()` 只是把窗口画出来，画完就继续往下走。没有 `exec()` 兜底，`main` 立刻 return，进程结束，窗口瞬间消失。

---

## exec() 内部的死循环

```cpp
int QApplication::exec()
{
    // 简化版：
    while (true)
    {
        // 1. 从操作系统拿事件
        QEvent *event = getNextEvent();      // 触摸、按键、定时器、网络...
        
        // 2. 分发给目标对象
        QObject *receiver = findReceiver(event);
        receiver->event(event);              // → keyPressEvent()
                                             // → mousePressEvent()
                                             // → timerEvent()
                                             // → signal/slot 调用
        
        // 3. 如果收到退出事件（窗口关闭），跳出循环
        if (event->type() == QEvent::Quit)
            break;
        
        // 4. 回到第 1 步
    }
    return exitCode;  // 返回退出码
}
```

---

## 为什么叫"事件循环"

```
用户触摸屏幕
    → Linux kernel 产生 input event
    → Qt QWS Server 封装为 QWSEvent
    → exec() 循环中的 getNextEvent() 拿到这个事件
    → 分发给 ZKSort::keyPressEvent() 处理
    → 处理完，回到循环顶部，等下一个事件
    ↓
定时器超时
    → QTimer 产生 timer event
    → exec() 拿到事件
    → 分发给对应槽函数（如 dccryptCheck()）
    → 处理完，回到循环
    ↓
网络数据到达
    → QTcpSocket 产生 readyRead 信号
    → exec() 拿到事件
    → 信号槽链触发
    → 回到循环
    ↓
...无穷无尽，直到窗口关闭
```

**所有你在界面上能感知到的"程序在运行"——按钮响应、定时器触发、网络处理、页面切换——都是 `exec()` 这个死循环在背后驱动的。**

---

## 阻塞的位置

```
main()
  │
  ├── ...各种初始化...
  ├── w.show();                    ← 窗口显示
  │
  ├── int execResult = a.exec();   ← ★ 程序在这里"停下"
  │                                   实际上进入了死循环
  │                                   不停地拿事件、发事件、处理事件
  │                                   你看到的所有 UI 交互都发生在这个循环里
  │
  │   (用户关闭了主窗口)
  │   exec() 退出循环，返回退出码
  │
  ├── logger->removeAllAppenders(); ← 代码继续往下走
  ├── return execResult;            ← main 返回
  └── 进程退出
```

---

## 返回值 `int`

```cpp
int execResult = a.exec();  // 如 0 表示正常退出
return execResult;          // 传给操作系统
```

退出码的来源：
- 用户正常关闭窗口 → `0`
- 调用 `QApplication::exit(1)` → `1`
- 调用 `QApplication::quit()` → `0`

这个项目里 `execResult` 直接作为 `main()` 的返回值，就是告诉 Linux："程序正常结束了"。

---

## exec() 进入循环到返回

```
a.exec() 被调用
    │
    ▼
┌─────────────────────────────┐
│  事件循环（可能持续几天/几月）  │
│  ┌─ 触摸事件 → 切换页面       │
│  ├─ 定时器 → dccryptCheck()   │
│  ├─ 串口数据 → 更新状态       │
│  ├─ 网络包 → MQTT 回调        │
│  └─ ...永不停歇...            │
│         ↓                     │
│  收到退出事件                  │
└─────────────────────────────┘
    │
    ▼
a.exec() 返回，代码继续执行
```

中间可能间隔了几天甚至几个月——工业设备常年不关机，`exec()` 一旦进入就可能运行数万小时，只有当用户主动关机或系统崩溃时才会退出循环。

---

## 一行代码定义 `int` 和阻塞运行矛盾吗

不矛盾。这是 C++ 最基本的语法：

```cpp
int execResult = a.exec();
//               ^^^^^^^^
//    exec() 是函数调用，它不返回时，赋值操作就不会完成
//    exec() 阻塞了，整行代码就卡在函数调用处
//    exec() 什么时候返回，execResult 就什么时候被赋值
```

等同于：

```cpp
int foo() {
    while(true) { ... }  // 死循环
    return 0;            // 永远到这不了
}

int result = foo();  // ← 程序永远停在这一行
```

区别只是 `exec()` 的死循环**有出口**（窗口关闭），而 `foo()` 没有。

---

## 一句话

`a.exec()` = Qt 程序的心脏起搏器。进入之前所有代码都是"准备工作"，进入之后程序才真正"活过来"，退出之后只是"收尸"（清理资源）。那行 `int execResult = a.exec()` 卡住的不是程序，而是 `main()` 的执行流——程序活在了 `exec()` 内部的死循环里。
