# Qt 绘图与窗口机制 — 从零开始

---

## 一、窗口是怎么出来的

一个 Qt 窗口程序最简单的代码：

```cpp
int main(int argc, char *argv[]) {
    QApplication app(argc, argv);    // ① 创建应用
    QWidget w;                        // ② 创建窗口
    w.resize(400, 300);               // ③ 设大小
    w.show();                         // ④ 显示
    return app.exec();                // ⑤ 进入事件循环
}
```

这五行代码背后发生了什么：

```
① QApplication
   → 连接操作系统窗口系统（Windows 的 Win32 API / Linux 的 X11 或 QWS）
   → 初始化字体、样式、调色板等全局资源
   → 一个程序只有一个 QApplication

② QWidget w
   → 构造一个空的矩形区域（内存中的数据结构）
   → 此时还没有对应的操作系统窗口，只是一块内存

③ w.resize(400, 300)
   → 设定宽 400、高 300 像素

④ w.show()
   → ★ 关键一步：Qt 向操作系统请求创建一个真实的窗口
   → Windows: ::CreateWindowEx()
   → Linux QWS: 打开 framebuffer 设备
   → 操作系统分配一块屏幕矩形给这个窗口
   → Qt 发一个 paintEvent → 触发第一次绘制

⑤ app.exec()
   → 进入事件循环（前面讲过了）
   → 窗口保持显示，响应鼠标/键盘/定时器
```

---

## 二、创建窗口的三种方式

**方式 1：直接用 QWidget（最原始）**

```cpp
QWidget w;
w.resize(400, 300);
w.show();
// 结果：一个空的灰色矩形，没有标题栏
```

**方式 2：用 QMainWindow（标准桌面程序）**

```cpp
QMainWindow w;
w.resize(800, 600);
w.show();
// 结果：有菜单栏、工具栏、状态栏的标准窗口
```

**方式 3：继承 QWidget 自己写（本项目的方式）**

```cpp
class ZKSort : public BaseUi   // BaseUi : public QWidget
{
    ZKSort() {
        ui->setupUi(this);     // 加载 .ui 文件描述的布局
    }
};
// 结果：完全自定义的内容和外观
```

方式 3 是最灵活的方式——窗口里有什么、长什么样，全部自己控制。

---

## 三、.ui 文件和 setupUi

`.ui` 文件是 Qt Designer 拖控件生成的 XML 文件。uic 编译器把它转成 C++ 代码：

```cpp
// ui_zksort.h（自动生成，不用手写）
class Ui_ZKSort {
    QVBoxLayout *verticalLayout;
    TopWidget *topWidget;
    QStackedWidget *ContainerStackedWidget;
    BottomWidget *bottomWidget;

    void setupUi(QWidget *ZKSort) {
        // 1. 设置窗口大小
        ZKSort->resize(1024, 768);

        // 2. 创建垂直布局
        verticalLayout = new QVBoxLayout(ZKSort);

        // 3. 创建 topWidget 并加入布局
        topWidget = new TopWidget(ZKSort);
        topWidget->setMinimumHeight(50);
        verticalLayout->addWidget(topWidget);

        // 4. 创建 StackedWidget 并加入布局
        ContainerStackedWidget = new QStackedWidget(ZKSort);
        verticalLayout->addWidget(ContainerStackedWidget);

        // 5. 创建 bottomWidget 并加入布局
        bottomWidget = new BottomWidget(ZKSort);
        bottomWidget->setMinimumHeight(60);
        verticalLayout->addWidget(bottomWidget);
    }
};
```

所以 `ui->setupUi(this)` 本质就是**在构造函数里自动创建所有子控件并排好布局**。

---

## 四、Qt 的绘制机制

Qt 不是你想画就立刻画。它遵循**事件驱动的绘制模型**：

```
┌──────────────────────────────────────────────────┐
│                 Qt 绘制流水线                      │
│                                                   │
│  1. 需要一个重绘：                                  │
│     - 窗口首次 show()                              │
│     - 窗口被遮挡后暴露                             │
│     - 调用了 update() / repaint()                  │
│     - 窗口大小变了                                  │
│                                                   │
│  2. Qt 合并多个重绘请求（优化）                     │
│     → 连续两次 update() 只触发一次 paintEvent       │
│                                                   │
│  3. Qt 调用 paintEvent(QPaintEvent *event)         │
│     → event->rect() 告诉你哪块区域需要重绘          │
│                                                   │
│  4. paintEvent 中创建 QPainter                     │
│     → 调用 drawXxx() 画内容                        │
│                                                   │
│  5. Qt 把画好的内容刷到屏幕                         │
└──────────────────────────────────────────────────┘
```

**核心原则：绘制只发生在 paintEvent 里。** 你不能在别的地方画，想画就调 `update()` 触发 paintEvent。

---

## 五、关键类一览

### 绘图相关

| 类        | 作用                                |
| -------- | --------------------------------- |
| QPainter | 画笔，所有 drawXxx() 的入口[[QPainter详解]] |
| QPen     | 笔（轮廓线）：颜色、粗细、虚实                   |
| QBrush   | 刷（填充）：颜色、渐变、纹理                    |
| QFont    | 字体                                |
| QColor   | 颜色                                |
| QPixmap  | 位图（适合显示，存在显存）                     |
| QImage   | 图像（适合处理，存在内存）                     |
| QStyle   | 样式引擎（QSS 背后就是它）                   |

### 窗口相关

| 类 | 作用 |
|----|------|
| QWidget | 所有控件的基类 |
| QApplication | 应用主控（管理事件循环、全局样式） |
| QMainWindow | 标准桌面窗口（菜单栏+工具栏+状态栏） |
| QDialog | 对话框 |
| QLayout | 布局管理器（QVBoxLayout/QHBoxLayout/QGridLayout） |
| QStackedWidget | 页面栈（同一时刻只显示一页） |
| QSplitter | 可拖拽分割面板 |

---

## 六、三种绘制场景

**场景 1：只做布局，不自己画（最常见）**

```cpp
// 用 .ui 文件拖拽控件 + QSS 控制外观
// 不需要写一行 paintEvent
// ZKSort 的主体布局就是这种
```

**场景 2：QSS + paintEvent 让样式生效**[[QSS样式渲染详解]]

```cpp
void paintEvent(QPaintEvent *) {
    QStyleOption opt;
    opt.init(this);
    QPainter p(this);
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
}
// BaseUi 的做法——自己不画，委托给 QSS 样式引擎
```

**场景 3：完全手绘**

```cpp
void paintEvent(QPaintEvent *) {
    QPainter p(this);
    p.setPen(QPen(Qt::red, 2));
    p.drawLine(0, 0, 100, 100);          // 画线
    p.setBrush(Qt::blue);
    p.drawRect(50, 50, 30, 30);          // 画矩形
    p.setFont(QFont("Arial", 14));
    p.drawText(10, 10, "Hello");         // 画文字
}
// MyWaveWidget、barchart 等控件就是这种
```

---

## 七、从代码到屏幕的完整创建流程（本项目为例）

```
main()
  │
  ├─ zkApplication a(argc, argv)
  │   → 初始化 Qt 内部（字体、样式、窗口系统连接）
  │
  ├─ QDesktopWidget* desktop = QApplication::desktop()
  │   → 查询屏幕尺寸（1024×768）
  │
  ├─ ZKSort w;                                          ← 构造函数开始
  │   │
  │   ├─ ui->setupUi(this);
  │   │   → new TopWidget        (创建顶部栏对象——内存分配)
  │   │   → new QStackedWidget   (创建页面栈——内存分配)
  │   │   → new BottomWidget     (创建底部栏对象——内存分配)
  │   │   → QVBoxLayout 排列三者
  │   │
  │   ├─ setWindowFlags(Qt::FramelessWindowHint)
  │   │   → 告诉 Qt："这个窗口不要标题栏"
  │   │
  │   ├─ LoadSkin()
  │   │   → 读取 QSS 文件 → qApp->setStyleSheet(...)
  │   │
  │   ├─ g_MainManager().setStackedWidget(...)
  │   │   → MiddleWidgetManager 接管 QStackedWidget 的控制
  │   │
  │   └─ new QTimer + connect(...)                       ← 构造函数结束
  │
  ├─ w.setFixedSize(1024, 768)
  │   → 固定窗口大小，不可拖拽缩放
  │
  ├─ w.move(0, 0)
  │   → 指定窗口在屏幕上的位置（左上角）
  │
  ├─ w.show()
  │   → Qt 向操作系统请求创建真实窗口
  │   → 操作系统分配屏幕区域
  │   → 触发 paintEvent → QPainter 绘制 → 用户看到界面
  │
  └─ app.exec()
      → 进入事件循环
      → 鼠标/触摸/键盘事件 → paintEvent → 绘制更新
```

---

## 八、什么时候会触发重绘

| 场景 | 触发方式 |
|------|---------|
| 窗口首次显示 | Qt 自动 |
| 窗口从最小化恢复 / 从遮挡中暴露 | 操作系统通知 Qt → Qt 自动 |
| 窗口大小改变 | Qt 自动 |
| 控件内容变化（如 QLabel::setText） | 控件内部调 update() |
| 你的代码想要重绘 | 调 `update()`（推荐）或 `repaint()`（立即） |
| QSS 规则变化 | 调 `update()` |

**关键规则：你永远不直接调用 `paintEvent()`，你调 `update()`，Qt 在合适的时机调用 `paintEvent()`。**

---

## 九、一句话总结

```
窗口创建: QApplication → new Widget → show() → exec()
           (初始化环境)  (内存对象)  (申请屏幕区域) (事件循环)

绘制流程: update() → paintEvent → QPainter → drawXxx() → 屏幕
           (请求)    (回调)     (画笔)      (画)      (输出)
```

具体怎么画，Qt 给了三条路：**直接用现成控件 + QSS 配色**（最省事）、**paintEvent 委托 QSS**（本项目 BaseUi 的做法）、**paintEvent 纯手绘**（波形图、柱状图等定制控件）。
