# QDesktopWidget 详解

## 一句话

**`QDesktopWidget` 是 Qt 中代表物理屏幕的对象，用来查询屏幕的尺寸、数量、分辨率等信息。**

---

## 在这个项目中的使用

```cpp
// main.cpp
QDesktopWidget* desktop = QApplication::desktop();
int screenW = desktop->width();    // 屏幕宽度（像素）
int screenH = desktop->height();   // 屏幕高度（像素）
```

---

## 继承链

```
QObject
  └── QWidget                      ← 所有可视控件的基类
        └── QDesktopWidget         ← 代表屏幕本身
```

它本质上是一个"虚拟的 Widget"——不对应屏幕上某个按钮或窗口，而是代表**整个物理屏幕**这个抽象概念。

---

## 核心方法

| 方法 | 返回值 | 含义 |
|------|--------|------|
| `width()` | int | 屏幕总宽度（像素） |
| `height()` | int | 屏幕总高度（像素） |
| `screenCount()` | int | 连接的屏幕数量 |
| `screenGeometry(n)` | QRect | 第 n 块屏幕的矩形区域（坐标+宽高） |
| `availableGeometry(n)` | QRect | 第 n 块屏幕的可用区域（去掉任务栏等保留区） |
| `primaryScreen()` | int | 主屏幕的编号 |
| `screenNumber(widget)` | int | 某个窗口当前显示在第几块屏幕上 |

---

## 单屏 vs 多屏

```
单屏（这个项目）:
  screenCount() = 1
  width() = 1024
  height() = 768
  → 只有一块嵌入式屏幕

多屏（PC）:
  screenCount() = 2
  screenGeometry(0) = {0, 0, 1920, 1080}    左屏
  screenGeometry(1) = {1920, 0, 1920, 1080} 右屏
  → 可以把窗口拖到不同屏幕
```

---

## 在这个项目中的完整使用

```cpp
QDesktopWidget* desktop = QApplication::desktop();
int screenW = desktop->width();   // 如 1024
int screenH = desktop->height();  // 如 768

// ① 记录屏幕高度到全局配置（界面排版需要根据分辨率适配）
struCnfg.screenH = screenH;

// ② 将全屏提示框居中放置
g_infoWidget().move(
    (desktop->width() - g_infoWidget().width()) / 2,      // 水平居中
    (screenH - g_infoWidget().height()) / 2                // 垂直居中
);

// ③ 创建主窗口，固定为全屏尺寸
ZKSort w;
w.setFixedSize(screenW, screenH);  // 窗口 = 屏幕大小 → 全屏
w.move(0, 0);                       // 贴紧屏幕左上角

// ④ 记录窗口的绝对位置（虽然 move(0,0)，但记录以备不时之需）
g_Runtime().appPositionX = (desktop->width() - w.width()) / 2;
g_Runtime().appPositionY = (screenH - w.height()) / 2;
```

在这个项目里，`QDesktopWidget` 的作用就是**拿到屏幕的物理分辨率**，然后让窗口全屏铺满。嵌入式设备只有一个屏幕，所以不存在多屏管理的复杂度。

---

## 和 `QApplication::desktop()` 的关系

```cpp
QDesktopWidget* desktop = QApplication::desktop();
//                        ^^^^^^^^^^^^^^^^^^^^^^^^
//                        静态方法，返回全局唯一的 QDesktopWidget 指针
```

`QApplication::desktop()` 返回的 `QDesktopWidget*` 是一个**单例**——整个程序只有一个桌面，不需要也不可能创建多个 `QDesktopWidget`。底层通过 X11/Wayland/QWS 等窗口系统协议获取屏幕的真实物理尺寸。
