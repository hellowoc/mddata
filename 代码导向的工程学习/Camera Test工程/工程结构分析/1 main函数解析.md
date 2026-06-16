d# main.cpp 逐段解析

整个 `main()` 函数按执行顺序可以分为 **8 个阶段**：

---

## 调试1：
[[dumpCallback解析]]

## 阶段 1：崩溃转储（仅 Linux + breakpad 编译宏）

```cpp
#ifdef breakpad
    setDumpPath("/crash");
    google_breakpad::MinidumpDescriptor descriptor("./crash");
    google_breakpad::ExceptionHandler eh(descriptor, NULL, dumpCallback, NULL, true, -1);
#endif
```

- [[setDumpPath解析]]
- 使用 Google Breakpad [[Google Breakpad库介绍]]，程序崩溃时自动生成 minidump 文件到 `./crash` 目录，方便事后分析
- [[Google Breakpad库介绍#^f286ed|MinidumpDescriptor]]
- [[Google Breakpad库介绍#^717801|ExceptionHandler]]
- 这是一个**编译时可选的调试功能**，只有定义了 `breakpad` 宏才生效

### 理解：[[dump文件格式与读取]]

```
setDumpPath()             设置或者说创建dump的书写地址 
dumpCallback()            写好dump之后的回调函数，告知已经写好，以及地址
MinidumpDescriptor(...)   指定地址
ExceptionHandler(...)     写dump
```


---

## 阶段 2：Qt 应用初始化 + 编码设置

```cpp
zkApplication a(argc, argv);                        // 自定义 QApplication 子类
QThread::currentThread()->setObjectName("Mainthread"); // 给主线程命名

QTextCodec *codec = QTextCodec::codecForName("UTF-8");
QTextCodec::setCodecForLocale(codec);     // 文件系统读写编码
QTextCodec::setCodecForTr(codec);         // tr() 翻译函数的默认编码
QTextCodec::setCodecForCStrings(codec);   // C字符串隐式转 QString 的编码
```

- `zkApplication` 继承自 `QApplication`，**重写了 `qwsEventFilter()`**，用于拦截 Qt 窗口系统的底层事件——主要是**屏幕背光管理**：每次有用户操作（触摸/按键）就重置背光倒计时[[zkApplication详解]]
- 四行编码设置确保整个程序在 **UTF-8** 下工作，避免中文乱码[[QTextCodec]]

---

## 阶段 3：USB 升级/维护触发[[g_Runtime解析]]

```cpp
QString usbPath = g_Runtime().getUsbPath();
if (usbPath != "") {
    // 检测4种触发文件，存在则执行对应操作：
    //   zksort_del_cnf.txt   → 删除 /sdcard/cnf 配置目录
    //   zksort_upgrade.txt   → 创建升级标记文件
    //   zksort_upgradep1.txt → 创建分区升级标记
    //   cmd.sh               → 拷贝到 /sdcard 并以 detached 方式执行
}
```

- 这是嵌入式设备常见的 **U 盘触发机制**：插入存有特定文件名的 U 盘，系统自动执行相应操作
- 检测完后立即删除触发文件（防止重启后重复执行）
- `cmd.sh` 会先 `chmod a+x` 赋予执行权限，再以 `startDetached`（脱离进程）方式运行

---

## 阶段 4：启动画面[[QFileInfo 用法]]

```cpp
QFileInfo bootImg("/sdcard/logo/boot.bmp");
if (bootImg.exists()) {
    QSplashScreen *splash = new QSplashScreen;
    splash->setPixmap(QPixmap("/sdcard/logo/boot.bmp"));
    splash->show();
}
```

- 如果 `/sdcard/logo/boot.bmp` 存在则显示启动画面
- 嵌入式设备上可以放**定制品牌 logo**，没有则不显示（splash 保持 NULL）

---

## 阶段 5：日志文件管理[[嵌入式工业日志方案]]

这段逻辑分两步：

**5a. 日志数量控制（最多保留 30 个归档）**
```cpp
// 列出 log.txt.* 文件，超过 30 个则删除最老的那个
if (fileInfo.size() >= 30) {
    QFile::remove(fileInfo.at(0).absoluteFilePath());
}
```

**5b. 跨天日志迁移**
```cpp
// 读取 log.txt 第一条日志的时间戳
// 如果日期 != 今天，说明上次运行跨天了
// 就把 log.txt 重命名为 log.txt.2026-05-19 归档
```

---

## 阶段 6：Log4Qt 日志框架初始化

```cpp
Log4Qt::Properties qtlogProper;
qtlogProper.setProperty("log4j.rootLogger", "DEBUG, A1");
qtlogProper.setProperty("log4j.appender.A1", "org.apache.log4j.DailyRollingFileAppender");
// ...
qtlogProper.setProperty("log4j.appender.A1.layout.ConversionPattern",
    "%d [%t] %p %c%x - %m%n");
Log4Qt::PropertyConfigurator::configure(qtlogProper);
```

- 用代码方式配置 Log4Qt（等效于 Java log4j 的 `log4j.properties` 文件）
- 使用 `DailyRollingFileAppender`，**按天滚动**日志文件
- 输出格式：`日期 [线程名] 级别 类名 - 消息`

---

## 阶段 7：资源加载 + 全局初始化[[QDesktopWidget详解]]

```cpp
QDesktopWidget* desktop = QApplication::desktop();
int screenW = desktop->width();   // 获取屏幕宽高
int screenH = desktop->height();

QResource::registerResource(..."/resource.rcc");  // 注册资源包（图片/翻译文件等）

myFlow.initAll();                 // 整机开机初始化（加载配置、初始化共享参数、语言词根等）
struCnfg.screenH = screenH;       // 记录屏幕高度到全局配置

paramDelayCode.getUiLanguage();   // 读取用户上次选择的语言
g_myLan().ChangeLanguage();       // 应用语言切换
```

- `myFlow` 是全局的 `GlobalFlow` 对象，`initAll()` 做了整机上电的核心初始化工作

---

## 阶段 8：显示主窗口 + 进入事件循环[[exec事件循环详解]]

```cpp
// 将信息提示框居中
g_infoWidget().move(..., (screenH - g_infoWidget().height()) / 2);

ZKSort w;
w.setFixedSize(screenW, screenH);  // 全屏固定尺寸
w.move(0, 0);                       // 贴屏幕左上角
w.show();

int execResult = a.exec();   // ★ 进入 Qt 事件循环，程序在此阻塞运行

// --- 以下在程序退出时执行 ---
if (splash) {
    splash->finish(&w);      // 关闭启动画面
    delete splash;
}
logger->removeAllAppenders();                       // 清理日志
logger->loggerRepository()->shutdown();
return execResult;
```

---

## 整体执行时序图

```
崩溃转储设置  →  QtApp+编码  →  USB触发检查  →  启动画面显示
                                                    ↓
    日志滚动管理  ←  跨天日志迁移  ←  旧日志清理
          ↓
    Log4Qt配置  →  资源包注册  →  myFlow全局初始化  →  语言加载
          ↓
    屏幕尺寸获取  →  创建ZKSort主窗口(全屏)  →  show()
          ↓
    进入事件循环 a.exec() ← 程序在此长时间运行
          ↓  (用户关闭窗口后)
    关闭splash  →  清理日志  →  return
```

---

## 关键设计思想

1. **全屏嵌入式应用** — 窗口固定屏幕大小，无标题栏，`move(0,0)` 贴紧左上角
2. **U 盘即插即用触发** — 通过特定文件名实现免登录的系统维护操作
3. **日志按天归档** — 自动检测跨天并归档，最多保留 30 个日志文件防止磁盘写满
4. **可选崩溃诊断** — breakpad 只在调试版本开启，量产版本不发版
