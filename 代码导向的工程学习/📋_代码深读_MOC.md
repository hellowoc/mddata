# 📋 代码深度阅读 — 导航索引

> 对应学习路径：**第三阶段 — 代码深读**
>
> 从 `main.cpp` 入口开始，逐级向上溯源。8 篇文档串联起完整启动流程。

---

## 🔗 按启动顺序阅读

```
main.cpp 程序入口
  │
  ├─ [1] [[代码导向的工程学习/Camera Test工程/工程结构分析/1 main函数解析|main 函数解析]]
  │     日志初始化 → 编码设置 → USB触发 → 启动画面 →
  │     资源加载 → 全局初始化 → ZKSort 创建 → exec()
  │
  ├─ [2] [[代码导向的工程学习/Camera Test工程/工程结构分析/2 ZKSort主窗口|ZKSort 主窗口]]
  │     全屏 1024×768 → BaseUi 基类 → 键盘事件 → 顶部/底部栏
  │
  ├─ [3] [[代码导向的工程学习/Camera Test工程/工程结构分析/3 dccryptCheck()|dccryptCheck() 加密校验]]
  │     授权检查 → 时间加密验证 → 过期处理
  │
  ├─ [4] MiddleWidgetManager 页面管理器
  │     → [[2026.1前/MiddleWidgetManager详解|MiddleWidgetManager 详解]]
  │     QStackedWidget 页面栈 → 所有页面的创建/切换/消息路由
  │
  └─ [5] [[代码导向的工程学习/Camera Test工程/工程结构分析/5 （通信）GlobalFlow|GlobalFlow 全局流程]]
        串口/UDP 协议解析 → 参数发送 → IPC 通信协调
```

---

## 🏗️ 工程全景

- [[代码导向的工程学习/Camera Test工程/工程结构分析/0 CameraTest工程框架分析|工程框架分析]] — 三层架构全景图（solution.pro → 3rdparty/uikits/zksort）

---

## 🔧 深入专题

- [[代码导向的工程学习/Camera Test工程/工程结构分析/日志系统|日志系统]] — Log4Qt 在嵌入式环境的完整用法
- [[代码导向的工程学习/Camera Test工程/工程结构分析/（自定义控件）mysortwidget控件库|mysortwidget 控件库]] — BaseUi 基类体系 + 40+ 控件

---

## 📝 总结

- [[代码导向的工程学习/学习总结与分析|本阶段学习总结]] — 16 天学习时间线 + 成果可视化

---

## 🔗 关联资源

- Qt 框架基础：[[2026.1前/📋_Qt与工程全景_MOC|Qt 与工程全景 MOC]]
- 从另一个角度理解工程：[[2026.1前/模块化了解工程/工程完整业务流程总结|完整业务流程]]

---

*MOC 文件，关联目录: `代码导向的工程学习/`*
