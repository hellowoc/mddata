# `qwsEventFilter` 工作机制

## 观察到的矛盾

```cpp
bool zkApplication::qwsEventFilter(QWSEvent *event)
//                                  ^^^^^^^^^^^^^^^^ 
//                                  声明了参数
{
    struGsh.nBacklightCounter = 0;    // ← 但函数体里完全没用到 event
    if (struGsh.nBacklightStat == 0) {
        // ...
    }
    return TRUE;
}
```

为什么声明了参数却不用？答案是——**参数的用途不由你决定，由调用方决定。**

---

## 工作原理类比

可以把它理解成一个**保安值班表上的签名栏**：

```
保安姓名栏（参数）    值班记录（函数体）

"不管谁来值班，我只看他来没来签到，不在乎他是谁"
```

`event` 参数就是这个"保安姓名"——Qt 框架硬性要求这个签名，它调用你的时候一定会传一个 `QWSEvent*` 进来，但你的逻辑（背光管理）**只需要知道"有人来了"这个事实，不关心来的是谁**。

---

## 调用链路

```
硬件：用户手指触摸屏幕
        ↓
Linux 内核：input 子系统读取触摸坐标
        ↓
Qt QWS Server：封装为 QWSEvent（类型=鼠标按下, 坐标=x,y）
        ↓
QApplication 内部：在分发事件到具体窗口之前，
    调用 qwsEventFilter(event) ← 把 QWSEvent 对象传给这个函数
        ↓
zkApplication::qwsEventFilter(event)
    ├── nBacklightCounter = 0     ← 只关心"有事发生"
    ├── 如果背光关了就打开
    └── return TRUE
        ↓
Qt 继续把 event 分发给目标窗口（按钮/列表/页面...）
```

**不是"有输入就触发"，而是"Qt 框架在有输入时主动调用它"**。对 `qwsEventFilter` 来说，它是**被动回调**，不是主动监听。

---

## 为什么参数不用却不能去掉

因为这是一个**虚函数重写**，函数签名必须和父类一致：

```cpp
// QApplication 中声明（父类）
virtual bool qwsEventFilter(QWSEvent *event);

// zkApplication 中重写（子类）
bool qwsEventFilter(QWSEvent *event);  // 签名必须一模一样
```

如果把参数删掉：

```cpp
bool qwsEventFilter();   // ❌ 签名不匹配，这不是重写，这是新函数
```

Qt 框架依然会调用父类 `QApplication::qwsEventFilter(QWSEvent*)`，你写的这个函数永远不会被调用——子类的新函数和父类的虚函数之间没有任何关联。

---

## 参数什么时候真正会被用到

如果这个项目不只是做背光管理，还想**区分事件类型**来做不同处理，就会用到 `event` 了。比如：

```cpp
bool zkApplication::qwsEventFilter(QWSEvent *event)
{
    // 现在要区分事件类型了，所以得用 event
    switch (event->type) {
    case QWSEvent::Mouse:
        // 触摸事件 → 重置背光
        nBacklightCounter = 0;
        break;
    case QWSEvent::Key:
        // 按键事件 → 也重置背光
        nBacklightCounter = 0;
        break;
    case QWSEvent::NoOp:
        // 空事件 → 不处理，不重置背光
        break;
    }
    return QApplication::qwsEventFilter(event);
}
```

这个项目没这么做的原因是：**对背光管理来说，所有类型的用户事件都需要重置计时器**，所以不需要区分事件类型，参数自然也就用不上了。

---

## 触发范围

`qwsEventFilter` 会被触发的 QWS 事件包括：

| 事件类型 | 场景 |
|---------|------|
| `QWSEvent::Mouse` | 触摸屏点击、移动、释放 |
| `QWSEvent::Key` | 物理按键、键盘 |
| `QWSEvent::Wheel` | 鼠标滚轮（如果接了鼠标） |
| `QWSEvent::Focus` | 窗口焦点切换 |
| `QWSEvent::PropertyNotify` | 窗口属性变化 |

每一个事件都会经过这个函数——这就是为什么它适合做背光管理：**只要用户在操作设备，这个函数就一定会被调到**。

---

## 工作流程图

```
触摸屏/按键/输入事件
        │
        ▼
QWS Server 封装为 QWSEvent 对象
        │
        ▼
qwsEventFilter(QWSEvent *event) 被调用
        │
        ├── Qt 保证: event 指向一个有效的 QWSEvent 对象
        │
        ├── 本项目的实现:
        │     只看"事件发生了"这个事实
        │     不看 event 内部的具体内容
        │     → 重置背光计时器
        │
        └── 返回后:
             Qt 继续将 event 分发给目标 widget 处理
```

---

## 一句话总结

`event` 参数是 Qt 框架签发的"通行证"——它证明了"用户输入事件已发生"。这个项目的背光逻辑不需要检查通行证上的详细信息（是触摸还是按键），只需确认"有人来过"，所以参数虽然接收了但没有读取。
