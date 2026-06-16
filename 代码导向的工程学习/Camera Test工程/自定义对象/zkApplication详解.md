# `zkApplication` 对象详解

## 定义位置

定义在 `zksort.h:56-61`，实现在 `zksort.cpp:8-27`。

---

## 继承链

```
QObject                      ← Qt 元对象系统根基
  └── QCoreApplication       ← 非 GUI 应用的事件循环
        └── QApplication      ← GUI 应用（窗口系统、样式、字体等）
              └── zkApplication ← 本项目自定义（加了 16 行代码）
```

继承链很短，只在 `QApplication` 基础上做了一件事。

---

## 构造 — 什么也不做

```cpp
zkApplication::zkApplication(int &argc, char **argv):
    QApplication(argc, argv)
{
}
```

构造函数体为**空** —— 所有初始化工作直接交给父类 `QApplication`。这就是为什么在 `main()` 里写 `zkApplication a(argc, argv)` 就能正常启动 Qt 应用。

---

## 核心重写 — `qwsEventFilter()`

```cpp
bool zkApplication::qwsEventFilter(QWSEvent *event)
{
    struGsh.nBacklightCounter = 0;          // ① 重置背光倒计时

    if (struGsh.nBacklightStat == 0) {       // ② 如果背光当前是关闭状态
        if (struGsh.bFlagPowerOff)           // ③ 但系统正在关机
            return TRUE;                     //    不处理，别再开背光了
        myFlow.setTsBackLight(1);            // ④ 否则立刻打开背光
        return TRUE;
    }

#if defined(Q_WS_QWS)
    return QApplication::qwsEventFilter(event);  // ⑤ 正常走默认流程
#endif
    return TRUE;
}
```

struGsh是全局变量对象，包含了关于机器的各种信息

这段代码的本质是用 **QWS 事件过滤器** 做了一个**屏幕背光自动管理**：

```
用户触摸屏幕 / 按键盘 / 任何输入事件
        ↓
qwsEventFilter() 被调用
        ↓
nBacklightCounter = 0    ← 重置空闲计时器
        ↓
背光当前关了？ ──是──→ 立即点亮屏幕
        │
        否
        ↓
正常传递给应用处理
```

### QWS 是什么[[qwsEventFilter工作机制]]

QWS（Qt Window System）是 **Qt for Embedded Linux**（Qt4 时代叫 Qtopia）自带的轻量级窗口系统。它直接运行在 Linux framebuffer 之上，不需要 X11。

`qwsEventFilter()` 的特殊之处在于：它是**事件到达任何窗口之前**的第一道关卡：

```
硬件输入（触摸屏/键盘/鼠标）
    ↓
QWS Server 进程接收
    ↓
qwsEventFilter(event)   ← ★ zkApplication 在这里拦截，比任何窗口都先拿到事件
    ↓ (返回 false 则继续传递)
具体窗口的 event() / keyPressEvent()
```

所以这 16 行代码实现的是**全局性的无侵入背光管理**——不需要在每个窗口/控件里写背光代码，只要用户操作了设备，背光自动重置。

---

## 为什么不用 `eventFilter` 而用 `qwsEventFilter`

| 方式                     | 安装方式      | 作用范围          | 时机          |
| ---------------------- | --------- | ------------- | ----------- |
| `installEventFilter()` | 需要挂到具体对象上 | 只拦截该对象的事件     | Qt 事件循环内    |
| `qwsEventFilter()`     | 重写这一个方法即可 | **所有 QWS 事件** | 比 Qt 事件循环更早 |

`qwsEventFilter` 是全局的，不需要安装，天生覆盖整个应用的所有输入——正是背光管理需要的行为。

---

## 完整类层次职责对照

```
QObject
  └── 信号槽、对象树、元对象
       │
       └── QCoreApplication
             └── 事件循环 (exec())
                  │
                  └── QApplication
                        ├── GUI 初始化（窗口系统、字体、样式）
                        ├── 桌面感知（desktop()->width/height）
                        └── 事件分发到窗口
                             │
                             └── zkApplication
                                   └── qwsEventFilter(): 在所有事件前
                                       插入背光管理逻辑
```

---

## 和 `ZKSort` 主窗口的关系

```
zkApplication a(argc, argv);    ← 应用对象（事件循环管理者 + 背光监控）
        │
ZKSort w;                       ← 主窗口对象（UI 容器）
    w.show();
        │
a.exec();                        ← zkApplication 启动事件循环
```

两者各司其职：
- `zkApplication` — **全局后台服务**（背光管理），整个程序生命周期只有它一个实例
- `ZKSort` — **前台 UI**（页面布局、用户交互）

按 Qt 惯例，`QApplication` 子类只放"跟窗口无关但跟整个应用有关"的逻辑。背光管理正是这种逻辑——它不关心当前显示哪个页面，只管"用户有没有在操作"。

---

## 一个小细节 — 关机信号

```cpp
if (struGsh.bFlagPowerOff)
    return TRUE;    // 关机中，不点亮屏幕
```

这是防止关机过程中的用户触碰导致屏幕又亮起来——关机流程走到最后阶段时 `bFlagPowerOff` 被置 1，此后再来任何输入事件都会被静默吞掉，屏幕保持熄灭直到断电。
