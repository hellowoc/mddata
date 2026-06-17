# QSplashScreen 用法详解

## 概述

`QSplashScreen` 是 Qt 框架中用于在应用程序启动时显示**启动画面**的控件。它继承自 `QWidget`，本质上是一个显示指定图片的窗口。对于启动耗时的应用程序（例如需要连接数据库、加载大量模块或插件），它能在主界面显示之前向用户提供一个视觉反馈，表示程序正在加载中，有效提升用户体验。

## 核心功能

| 类别 | 常用方法 | 说明 |
|------|---------|------|
| 构造与显示 | `QSplashScreen(const QPixmap &pixmap)` | 使用指定图片构造启动画面 |
| | `show()` | 显示启动画面 |
| 消息显示 | `showMessage(const QString &message)` | 在启动画面上显示文字信息 |
| 关闭画面 | `finish(QWidget *mainWin)` | 在主窗口显示后关闭启动画面 |
| 用户交互 | `mousePressEvent()` (重写) | 处理鼠标点击（默认点击画面可关闭） |

## 基础用法示例

```cpp
int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    // 1. 加载启动画面图片
    QPixmap pixmap(":/images/splash.png");

    // 2. 创建并显示启动画面
    QSplashScreen splash(pixmap);
    splash.show();

    // 3. 在启动画面上显示加载信息，并处理事件让画面实时更新
    splash.showMessage("正在加载模块...", Qt::AlignLeft | Qt::AlignBottom, Qt::white);
    app.processEvents();

    // ... 这里模拟执行耗时的初始化任务 ...

    splash.showMessage("正在连接数据库...", Qt::AlignLeft | Qt::AlignBottom, Qt::white);
    app.processEvents();

    // ... 更多初始化工作 ...

    // 4. 创建并显示主窗口
    MainWindow mainWindow;
    mainWindow.show();

    // 5. 关闭启动画面（finish 内部调用 close）
    splash.finish(&mainWindow);

    return app.exec();
}