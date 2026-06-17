# QSS 样式渲染详解

## QSS 是什么

```
QSS = Qt Style Sheets
     = Qt 的 "CSS"
     = 用文本文件控制 Qt 程序外观的一套语法
```

浏览器用 CSS 控制 HTML 的外观，Qt 用 QSS 控制 Widget 的外观。理念完全一致。

---

## 在这个项目中的位置

```cpp
// ZKSort::LoadSkin()
QString skinPath = ":/white/skin/qss/skin.qss";  // 从 .rcc 加载 QSS 文件
QFile file(skinPath);
file.open(QIODevice::ReadOnly);
QString styleSheet = file.readAll();   // 读取出纯文本
qApp->setStyleSheet(styleSheet);       // ★ 全局应用
```

设置后，QSS 中对所有 widget 的样式规则**全程序生效**，不需要在每个地方单独写。

---

## QSS 语法的简单示例

```css
/* 所有 QPushButton */
QPushButton {
    background-color: #4a90d9;
    color: white;
    border: 1px solid #2a6db0;
    border-radius: 4px;
    padding: 8px 16px;
    font-size: 14px;
}

/* 鼠标悬停时 */
QPushButton:hover {
    background-color: #5ba0e9;
}

/* 按下时 */
QPushButton:pressed {
    background-color: #3a80c9;
}

/* 特定名称的控件 */
#alarmButton {
    background-color: red;
}

/* 输入框 */
QLineEdit {
    border: 1px solid #ccc;
    background-color: white;
}
```

---

## QSS 是怎么生效的（"样式渲染"）

`qApp->setStyleSheet(styleSheet)` 调用后：

```
setStyleSheet("QPushButton { background: #4a90d9; ... }")
    │
    ▼
Qt 内部创建一个 QStyleSheetStyle 对象
    │ （这是 QStyle 的子类，专门负责 QSS 的解析和渲染）
    │
    ▼
每次任何 widget 需要重绘(paintEvent)时：
    │
    ├── widget->style() 返回 QStyleSheetStyle
    │
    ├── style()->drawPrimitive(PE_Widget, &opt, &painter, widget)
    │       │
    │       ├── 1. 检查是否有 QSS 规则匹配此 widget
    │       │       → 看 widget 的类名（如 "QPushButton"）
    │       │       → 看 widget 的对象名（如 "alarmButton"）
    │       │       → 看 widget 的状态（hover / pressed / disabled）
    │       │
    │       ├── 2. 找到匹配的规则 → 提取样式属性
    │       │       → background-color → 用 QPainter 填充背景色
    │       │       → border → 用 QPainter 画边框
    │       │       → border-radius → 裁剪为圆角
    │       │
    │       └── 3. 调用 QPainter 的实际绘图函数
    │               → painter.fillRect()
    │               → painter.drawLine()
    │               → ...
    │
    └── 绘制完成，widget 表面呈现 QSS 定义的外观
```

---

## 这也是为什么 paintEvent 里需要那段代码

```cpp
void BaseUi::paintEvent(QPaintEvent *)
{
    QStyleOption opt;
    opt.init(this);
    QPainter p(this);
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
    //     ^^^^^^^^^^^^^
    //     如果设置了 QSS，这里的 style() 返回的就是 QStyleSheetStyle
    //     drawPrimitive 会走上面那条 QSS 匹配+渲染的路径
    //     如果不写这个 paintEvent，QWidget 默认实现不调用 drawPrimitive
    //     QSS 的背景色和边框就不会被画出来
}
```

---

## 样式渲染的完整链路

```
1. skin.qss 被 LoadSkin() 读入，字符串传给 setStyleSheet()
2. Qt 创建 QStyleSheetStyle 对象，替代默认的系统样式
3. 任何 widget 需要重绘 → paintEvent 被调用
4. paintEvent 中调用 style()->drawPrimitive()
5. QStyleSheetStyle 解析 QSS 规则，匹配 widget，提取属性值
6. 用 QPainter 把背景色/边框/圆角画到 widget 上
7. 用户看到的是 QSS 定义的外观
```

步 4-7 每一次重绘都会发生。所以当你改了 QSS 文件后，触发一次重绘，外观就变了——不需要重启程序。

---

## 本质理解

```
QSS 文件 = 规则仓库
重绘时 → 查仓库 → 有规则就照着画 → 没规则就用默认样式
换皮肤 = 换一个 QSS 文件 → setStyleSheet() → 所有 widget 自动重绘
```

---

## 为什么工业设备要用 QSS

| 方式 | 效果 |
|------|------|
| 不写 QSS | 灰框白底默认样式，各平台不一样 |
| 每个 widget 单独 `setStyleSheet()` | 代码里散落样式，改一个颜色要搜几十个文件 |
| 一个统一的 skin.qss 文件 + LoadSkin() | **改一个文件就换一套皮肤**，换品牌色/换字体/适配不同屏幕，全部集中在 QSS 里 |

这个项目有两套 QSS 文件：`skin.qss`、`skin_1024_600.qss`、`skin_1024_768.qss`——不同屏幕分辨率用不同的样式文件，`LoadSkin()` 根据 `screenH` 自动选。
