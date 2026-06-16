# QPainter — Qt 的 2D 绘图引擎

## 它是什么

```
QPainter
  继承: 无（纯工具类，不继承 QObject）
  所属: QtGui 模块
  定位: Qt 所有 2D 绘制的统一入口
```

它是 Qt 中**所有"画出来"操作的执行者**——画文字、画矩形、画图片、画渐变、画线条，全部通过 QPainter。

---

## 在这个项目中的使用

```cpp
void BaseUi::paintEvent(QPaintEvent *)
{
    QStyleOption opt;
    opt.init(this);
    QPainter p(this);                                    // 创建绑定到此 widget 的画笔
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
}
```

这里的 `QPainter p(this)` 只做了**最小化的存在**——它创建了一个绑定到当前 widget 的绘图上下文，然后被传给了 `drawPrimitive()`。QPainter 本身在这里不主动画任何东西，它只是提供"画布"，真正的绘制由 `style()->drawPrimitive()` 完成。

---

## 继承链

```
QPainter（独立类，不继承任何基类）
```

它不继承 QObject，不能用信号槽，不能用对象树管理。它就是一把"笔"，用完就丢。

---

## 核心工作原理

```
QPainter p(widget);
        │
        ├── 构造函数内部:
        │     → 调用 widget->paintEngine() 获取绘图后端
        │     → 后端可能是:
        │         ├── QWidget  → 光栅绘图引擎（软件渲染，画到像素缓冲区）
        │         ├── QImage   → 光栅绘图引擎（画到内存图像）
        │         ├── QPrinter → 打印引擎（生成 PostScript/PDF）
        │         └── QPicture → 记录引擎（录制绘图指令，稍后回放）
        │     → 初始化画笔、画刷、字体等默认状态
        │     → 设置裁剪区 = widget 的 rect（超出不画）
        │     → 开启抗锯齿
        │
        └── 后续的 drawXxx() 调用:
              → 光栅引擎计算像素 → 写入 widget 的像素缓冲区
              → Qt 事件循环在合适时机把缓冲区刷到屏幕
```

---

## 核心 API

| 类别 | API | 作用 |
|------|-----|------|
| **几何图形** | `drawRect()` `drawEllipse()` `drawLine()` `drawPolygon()` | 画矩形、圆、线、多边形 |
| **文字** | `drawText()` | 画文字 |
| **图片** | `drawPixmap()` `drawImage()` | 画位图、图像 |
| **填充** | `fillRect()` `eraseRect()` | 填充矩形区域 |
| **变换** | `translate()` `rotate()` `scale()` | 平移、旋转、缩放坐标系 |
| **裁剪** | `setClipRect()` `setClipRegion()` | 限制绘制区域 |
| **状态** | `save()` `restore()` | 保存/恢复画笔状态（压栈/出栈） |
| **属性设置** | `setPen()` `setBrush()` `setFont()` | 设置线条样式、填充样式、字体 |

---

## 三个核心属性

```cpp
QPainter p(this);

p.setPen(QPen(Qt::red, 2));           // 画笔 = 轮廓线（颜色、粗细、虚实）
p.setBrush(QBrush(Qt::yellow));        // 画刷 = 填充（颜色、渐变、纹理）

p.drawRect(10, 10, 100, 50);
//  → 画一个矩形：红色 2px 边框 + 黄色填充

p.setFont(QFont("Arial", 16, QFont::Bold));
p.drawText(20, 40, "Hello Qt");
//  → 用 Arial 16号粗体写 "Hello Qt"
```

**Pen（笔）= 描边，Brush（刷）= 填色。** 类比画画：先用笔画轮廓 (pen)，再用刷子涂颜色 (brush)。

---

## save() / restore() — 状态管理

```cpp
p.save();           // 把当前所有状态压入栈
p.setPen(Qt::red);
p.drawText(10, 10, "红字");
p.restore();        // 恢复之前的状态（蓝色笔又回来了）
p.drawText(10, 30, "又变回蓝色了");
```

这在复杂绘制中很关键——临时改变属性后能恢复原样，不影响后续绘制。

---

## 这个项目中还有哪里用了 QPainter

虽然不是直接 new QPainter，但项目中以下控件都间接依赖 QPainter：

- `MyWaveWidget` —— 画实时波形曲线
- `barchart.cpp` —— 画柱状图
- `plotcurve.cpp` —— 画折线图
- `mywid.h` —— 各种自定义控件的绘制

它们的 `paintEvent` 里都会创建 QPainter 然后调用 `drawLine`、`drawRect`、`drawText` 等。

---

## 安全性约束

```
QPainter 只能在 paintEvent() 中构造
         ↓
不能在 paintEvent 之外创建 QPainter(widget) 然后画
         ↓
违反 → Qt 警告
```

这是 Qt 的硬性规定——widget 的绘制必须发生在 `paintEvent` 调用栈内，确保双缓冲和裁剪正确工作。

---

## 一句话

QPainter 是 Qt 所有 2D 绘制的**统一抽象层**——不管目标是屏幕窗口、内存图片还是打印机，都用同一套 `drawXxx()` API 来画。`BaseUi::paintEvent` 里创建它是为了把"画布"交给 `style()->drawPrimitive()`，让 QSS 样式引擎在上面绘制背景和边框。
