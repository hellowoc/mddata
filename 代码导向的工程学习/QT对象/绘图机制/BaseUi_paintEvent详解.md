# `BaseUi::paintEvent` — 让 QSS 样式生效

## 问题背景

Qt 的样式表（QSS，类似于 CSS）可以给控件设背景色、边框、圆角等：

```css
/* skin.qss 中的样式 */
ZKSort {
    background-color: #1a1a2e;
    border: 2px solid #333;
}
```

但如果你直接继承 `QWidget` 写一个新的类，这个背景色**不会生效**。因为 Qt 在绘制 QWidget 时，如果发现这是自定义子类，就会走"只绘制子控件，不绘制自身背景"的路径——Qt 不知道这个自定义 widget 长什么样，让你自己去画。

---

## 解决办法

```cpp
void BaseUi::paintEvent(QPaintEvent *)
{
    QStyleOption opt;                                    // ① 创建样式选项对象
    opt.init(this);                                      // ② 从当前 widget 提取状态
    QPainter p(this);                                    // ③ 创建画笔
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this); // ④ 画背景
}
```

---

## 逐行执行链

### ① `QStyleOption opt;`

```
QStyleOption
  继承: 无（轻量级数据结构）
  作用: 携带控件的状态信息，传给样式引擎

opt 构造后是空的
```

### ② `opt.init(this);`

```
opt.init(this)
  → 从 this (即 ZKSort 或子类 widget) 身上提取：
    ├── 控件的启用/禁用状态
    ├── 是否有焦点
    ├── 是否被按下
    ├── 控件矩形大小
    └── 字体、文字方向等

这些状态决定了 QSS 中 :hover :pressed :disabled 等选择器是否匹配
```

### ③ `QPainter p(this);`

```
QPainter p(this)
  继承: QPainter（纯工具类，不继承 QObject）
  作用: Qt 的绘图引擎

p(this) → 创建绑定到 this widget 的画笔
  后续所有绘制操作都画在这个 widget 表面
```

### ④ `style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);`

```
style()  → 返回当前使用的 QStyle 对象（可能被 QSS 代理）
          如果设置了样式表，返回 QStyleSheetStyle（QSS 代理）
          如果没设，返回系统原生样式（如 WindowsStyle）

drawPrimitive(QStyle::PE_Widget, ...)
  → PE_Widget = "绘制一个普通 widget 的背景"
  → 样式引擎读取 opt 中的状态
  → 匹配 QSS 规则
  → 绘制背景色、边框、圆角等
```

---

## 完整链路

```
QSS 规则: ZKSort { background-color: #1a1a2e; border: 2px solid #333; }
                ↓
paintEvent() 被 Qt 调用（widget 需要重绘时）
    ↓
opt.init(this)     ← 提取状态（启用、尺寸、焦点...）
    ↓
style()            ← 拿到 QSS 样式引擎
    ↓
drawPrimitive(PE_Widget, &opt, &p, this)
    ↓
QSS 引擎: opt 里的状态匹配了 ZKSort 选择器吗？
    → 匹配 → 画 #1a1a2e 背景 → 画 2px #333 边框
    ↓
输出到屏幕
```

---

## 没有这个 paintEvent 会怎样

```cpp
// BaseUi 中不写 paintEvent
// → 用的是 QWidget::paintEvent() 默认实现
// → QSS 中的 background-color 不生效
// → 界面是灰蒙蒙的默认色，完全没有自定义皮肤
```

这就是为什么 `BaseUi` 要写这个函数——它让所有继承 `BaseUi` 的类（包括 `ZKSort` 和所有业务页面）都能通过 QSS 文件统一控制外观，而不需要在代码里手动 `setStyleSheet()`。

---

## 一句话

这个 `paintEvent` 就是**打开 QSS 背景渲染的开关**。没有它，QSS 样式表只对按钮、输入框等原生控件生效；有了它，自定义 widget 也能被 QSS 控制背景和边框。
