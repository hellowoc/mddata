# 📋 Qt 框架与工程全景 — 导航索引

> 对应学习路径：**第一阶段 — 粗览工程全貌**
>
> 总计 47 篇文档，涵盖 Qt 核心机制、工程架构全景、模块速览。

---

## 🏗️ 一、工程架构全景

### 入口与启动流程
- [[2026.1前/zkApplication详解|zkApplication 启动]] — 应用对象初始化
- [[2026.1前/MiddleWidgetManager详解|MiddleWidgetManager]] — QStackedWidget 页面管理器
- [[2026.1前/exec之后运行流程|exec() 之后运行流程]] — 进入事件循环后的启动时序

### 全局基础设施
- [[2026.1前/global全局单例详解|全局单例]] — 全局数据访问模式
- [[2026.1前/Runtime详解|Runtime 运行时]] — 全局状态管理
- [[2026.1前/g_Runtime解析|g_Runtime 解析]] — 全局运行时结构体

### 模块化了解工程（7 篇深度分析）
- [[2026.1前/模块化了解工程/工程完整业务流程总结|完整业务流程]] — 大米分选全链路
- [[2026.1前/模块化了解工程/系统时序与进程线程分析|时序与线程模型]] — 7-9 线程协作
- [[2026.1前/模块化了解工程/异常路径与故障恢复分析|异常与故障恢复]] — 故障场景处理
- [[2026.1前/模块化了解工程/bus通信层详解|bus 通信层]]
- [[2026.1前/模块化了解工程/arithprofilewidget算法参数详解|arithprofilewidget 算法参数]]
- [[2026.1前/模块化了解工程/语言系统详解|语言系统（多语言切换）]]

---

## 📦 二、功能模块速览

### 业务页面
- [[2026.1前/mainpage详解|mainpage 主页面群]] — 30 个业务页面（用户操作、灵敏度、供料等）
- [[2026.1前/engineerset工程师调试详解|engineerset 工程师调试]] — 16 个调试/训练页面
- [[2026.1前/factoryset工厂设置详解|factoryset 工厂设置]] — 25 个配置页面
- [[2026.1前/systeminfo系统信息详解|systeminfo 系统信息]] — 23 个监控/测试页面

### 数据持久化
- [[2026.1前/data数据层详解|data 数据层总览]]
- [[2026.1前/myjson详解|myjson — JSON 配置读写]]
- [[2026.1前/MySqlite与SQL数据库详解|MySqlite — SQLite 统计]]
- [[2026.1前/JsonDataConvert详解|JsonDataConvert — JSON↔C结构体映射]]

### 崩溃处理
- [[2026.1前/Google Breakpad库介绍|Google Breakpad 库]]
- [[2026.1前/dump文件格式与读取|dump 文件格式与读取]]
- [[2026.1前/dumpCallback解析|dumpCallback 回调解析]]
- [[2026.1前/setDumpPath解析|setDumpPath 路径设置]]

### 串口与缓冲
- [[2026.1前/环形缓冲区详解|环形缓冲区（MyQueue）]]
- [[2026.1前/锁机制详解|锁机制（QMutex/ReadWriteLock）]]

### 显示组件
- [[2026.1前/ShowWidget详解|ShowWidget 显示组件]]
- [[2026.1前/waitSetFpgaMode详解|waitSetFpgaMode 等待界面]]

### 日志
- [[2026.1前/嵌入式工业日志方案详解|嵌入式工业日志方案]] — Log4Qt 在 ARM 上的实战

---

## 🎨 三、Qt 核心机制

### 信号槽与事件
- [[2026.1前/connect信号槽详解|connect 信号槽]] — Qt 核心通信机制
- [[2026.1前/Qt伙伴机制|Qt 伙伴机制]] — QBuddy 标签-控件关联
- [[2026.1前/exec事件循环详解|exec 事件循环]] — 为什么 Qt 程序不会退出
- [[2026.1前/qwsEventFilter工作机制|qwsEventFilter]] — Qt 嵌入式事件过滤器

### 绘图与 UI
- [[2026.1前/Qt绘图与窗口机制入门|Qt 绘图与窗口机制入门]]
- [[2026.1前/BaseUi_paintEvent详解|BaseUi::paintEvent 详解]]
- [[2026.1前/QPainter详解|QPainter 详解]]
- [[2026.1前/QSS样式渲染详解|QSS 样式渲染详解]]
- [[2026.1前/QDesktopWidget详解|QDesktopWidget 详解]]

### 数据与 IO
- [[2026.1前/QSettings详解|QSettings 详解]]
- [[2026.1前/QDir|QDir 目录操作]]
- [[2026.1前/QFileInfo 用法详解|QFileInfo 文件信息]]
- [[2026.1前/QResource详解|QResource 资源系统]]
- [[2026.1前/Qt资源系统与文件系统|Qt 资源系统与文件系统]]
- [[2026.1前/QTimer详解|QTimer 定时器]]
- [[2026.1前/setProperty详解|setProperty 动态属性]]

### 对象体系
- [[2026.1前/QObject及其衍生类族谱|QObject 类族谱]]

---

## 🛠️ 四、工具与配置
- [[2026.1前/Git与GitHub使用指南|Git 与 GitHub 使用指南]]
- [[编译过程全链路详解-主文档|编译过程全链路详解]]（根目录独立文档）

---

*MOC 文件，关联目录: `2026.1前/`*
