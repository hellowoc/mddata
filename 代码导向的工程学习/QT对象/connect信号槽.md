# `connect()` — Qt 信号槽连接

## 语法拆解

```cpp
connect(mytimer,              // ① 信号源（谁发信号）
       SIGNAL(timeout()),     // ② 信号（发生了什么）
       this,                  // ③ 接收者（谁处理）
       SLOT(dccryptCheck())); // ④ 槽函数（怎么处理）
```

四个参数构成一句话：「**① mytimer** 的 **② timeout 信号** 被触发时，调用 **③ this** 的 **④ dccryptCheck 方法**」

---

## 类比

```
connect(门铃,   SIGNAL(被按下了()),
        我,     SLOT(去开门()));
//     ↑               ↑
//   信号源           接收者
//   (门铃)          (我)
//
// 门铃被按下 → 我收到通知 → 我去开门
```

代码不会被通知"卡住"——调用 `connect` 后就继续往下走了，信号触发时才异步回调槽函数。

---

## 工作机制

```
mytimer->start(100)
    │
    │  (100ms 内程序做别的事)
    │
    ▼
100ms 到期 → QTimer 内部发射信号:
    emit timeout();
    │
    ▼
Qt 查找所有 connect 到此信号的连接
    → 找到: connect(mytimer, SIGNAL(timeout()), this, SLOT(dccryptCheck()))
    │
    ▼
Qt 调用 this->dccryptCheck()

整个过程调用栈:

  事件循环 → QTimer::timerEvent()
           → emit timeout()
           → Qt元对象系统查找连接
           → ZKSort::dccryptCheck()   ← ★ 你的代码在这里
```

---

## 本质：发布-订阅模式

```
connect 做的事:

1. 建立一条连接记录:
   { 信号源: mytimer,  信号: "timeout()",  接收者: this,  槽: "dccryptCheck()" }

2. 把这个记录存到 Qt 内部的连接表里

3. 将来信号源发射信号时，Qt 查表 → 找到记录 → 调接收者的槽函数
```

---

## SIGNAL() 和 SLOT() 宏

```cpp
SIGNAL(timeout())      → 展开为 "2timeout()"      (QMetaObject 的信号签名)
SLOT(dccryptCheck())   → 展开为 "1dccryptCheck()"  (QMetaObject 的槽签名)
```

这两个宏是 Qt4 时代的写法，本质是把函数签名转换成字符串，Qt 内部通过字符串匹配来查找。需要继承 `QObject` 且声明 `Q_OBJECT` 宏才能用。

---

## 这个项目中的 connect 调用汇总

ZKSort 构造函数结束时只有一个 connect。但在 `dccryptCheck()` 末尾有一大波 connect：

```cpp
// dccryptCheck() 最后几行 —— 连接后台线程信号到 ZKSort 槽函数
connect(myUpdateStatusThread, SIGNAL(sWipeStartSig()),           this, SLOT(myWipeStartSlt()));
connect(myUpdateStatusThread, SIGNAL(sLostFpgaPowerOffSig()),     this, SLOT(lostFpgaPowerOffSlt()));
connect(myUpdateStatusThread, SIGNAL(sAlarmTurnOffSig()),         this, SLOT(alarmTurnOffFeedSlt()));
connect(myUpdateStatusThread, SIGNAL(sAlarmTurnOnSig()),          this, SLOT(alarmTurnOnFeedSlt()));
connect(myUpdateStatusThread, SIGNAL(sDccryptWarningSig()),       this, SLOT(warnDccryptSlt()));

// 每个 connect 都是一条规则：
//   后台线程的某个信号 → ZKSort 的某个处理方法
```

---

## 为什么分成两拨连接

```
构造函数里:
  connect(mytimer, SIGNAL(timeout()), this, SLOT(dccryptCheck()));
  → 只连一个定时器，100ms 后触发加密校验

dccryptCheck() 末尾:
  connect(myUpdateStatusThread, SIGNAL(sWipeStartSig()), ...);
  connect(myUpdateStatusThread, SIGNAL(sLostFpgaPowerOffSig()), ...);
  ... (十几个 connect)
  → 加密校验通过后，才把所有后台线程的信号接上来
```

**加密没通过的话，后面这些功能根本不需要启动。** 先验证权限再接线——没授权就别想用设备。

---

## 一句话

`connect(A, SIGNAL(xx), B, SLOT(yy))` = "当 A 发生了 xx 事件，就调 B 的 yy 方法"。一条 connect 就是一条"事件到处理函数"的绑定规则，多个 connect 可以并存，多个信号可以接同一个槽，一个信号也可以接多个槽。
