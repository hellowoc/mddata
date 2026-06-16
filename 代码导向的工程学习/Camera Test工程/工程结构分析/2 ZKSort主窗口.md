# ZKSort 主窗口详解

---

## 一、继承链

```
QObject
  └── QWidget                     ← 能画出来的控件
        └── BaseUi                ← 本项目自定义 UI 基类
              └── ZKSort          ← 主窗口
```

### BaseUi 做了什么[[QPainter详解]]

```cpp
class BaseUi : public QWidget
{
    Q_OBJECT
    LOG4QT_DECLARE_QCLASS_LOGGER   // 注入 logger() 成员

    virtual void retranslateUiWidget() = 0;   // 纯虚：子类必须实现语言切换
    virtual void showPage(bool) = 0;          // 纯虚：子类必须实现页面显示

protected:
    void changeEvent(QEvent *event)           // 监听事件
    {
        if (event->type() == QEvent::LanguageChange)
            retranslateUiWidget();            // 语言变了 → 调用子类的刷新
    }

    void paintEvent(QPaintEvent *)            // 让 QSS 样式表生效
    {
        QStyleOption opt;
        opt.init(this);
        QPainter p(this);
        style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
    }
};
```

BaseUi 是 ZKSort 和所有其他页面 widget 的共同父类，提供两个通用能力：语言切换时自动刷新界面 + QSS 样式渲染支持。[[QSS样式渲染详解]]

---

## 二、UI 布局（来自 .ui 文件）

```
┌─────────────────────────────────────────────┐
│  TopWidget (高度: 50px, 固定)               │  顶部状态栏
│  显示：当前页面标题、时间、状态图标           │
├─────────────────────────────────────────────┤
│                                             │
│  QStackedWidget (可变高度, 占满剩余空间)     │  中间页面区
│  ContainerStackedWidget                       │  所有业务页面在这里切换
│  第0页 → 第1页 → 第2页 → ...                 │
│                                             │
├─────────────────────────────────────────────┤
│  BottomWidget (高度: 60px, 固定)             │  底部导航栏
│  显示：按钮、阀状态、供料状态                 │
└─────────────────────────────────────────────┘
```

三个区域用 `QVBoxLayout` 垂直排列，间距 1px，边距全部为 0（全屏贴边）。`QStackedWidget` 是 Qt 的页面栈控件，同一时刻只显示一个页面，切换时只换中间区域，顶部和底部不变。
[[Qt绘图与窗口机制入门]]

---

## 三、构造函数逐项

```cpp
ZKSort::ZKSort(QWidget *parent) :
    BaseUi(parent),
    ui(new Ui::ZKSort)
{
    // ① 加载 UI 布局
    ui->setupUi(this);

    // ② 初始化底部导航栏状态（内部初始化后先隐藏）
    ui->bottomWidget->showPage(true);
    ui->bottomWidget->hide();

    // ③ 初始化顶部状态栏
    ui->topWidget->showPage(true);

    // ④ 无边框窗口（嵌入式设备不需要标题栏）
    setWindowFlags(Qt::FramelessWindowHint);

    // ⑤ 加载 QSS 皮肤样式
    LoadSkin();

    // ⑥ 把 QStackedWidget 的控制权交给 MiddleWidgetManager（页面管理器）
    g_MainManager().setStackedWidget(ui->ContainerStackedWidget, this);

    // ⑦ 记录开机次数（写入 QSettings INI 文件）
    QSettings setting(CFG_APPSet, QSettings::IniFormat);
    int onoff_cnts = setting.value("onoffcnts", 0).toInt();
    setting.setValue("onoffcnts", ++onoff_cnts);
    setting.sync();
    logger()->info("power on: %1 times", onoff_cnts);

    // ⑧ 启动加密校验定时器（100ms 后触发 dccryptCheck）
    mytimer = new QTimer(this);
    connect(mytimer, SIGNAL(timeout()), this, SLOT(dccryptCheck()));
    mytimer->start(100);

    // ⑨ 准备 FPGA 模式等待定时器（还没启动，等待需要时才 start）
    mytimerWaitFpgaMode = new QTimer(this);
    connect(mytimerWaitFpgaMode, SIGNAL(timeout()), this, SLOT(waitSetFpgaMode()));

    // ⑩ 配置哪些页面不需要显示底部/顶部导航栏
    m_hideBottomPageId.append(SM_MACHINE_DCCRYPT_WIDGET);   // 加密页面
    m_hideBottomPageId.append(SM_SELF_CHECK_WIDGET);        // 自检页面
    m_hideBottomPageId.append(SM_MAIN_PAGE_WIDGET_NEW);     // 新主页
    m_hideBottomPageId.append(SM_CREATE_MODE_WIDGET);       // 创建模式页
    m_hideBottomPageId.append(SM_MAIN_PAGE_WIDGET);         // 旧主页
    m_hideBottomPageId.append(SM_SVM_IMAGE_WIDGET);         // SVM图像页
    m_hideTopPageId.append(SM_SVM_IMAGE_WIDGET);            // SVM图像页也隐藏顶部
}
```
在6中将窗口的控制权交给了[[4 （界面管理）MiddleWidgetManager]]也就是说，在构造这类的时候，就已经完成了窗口的托管
[[代码导向的工程学习/QT对象/QSettings]]
---

## 四、成员函数分类

| 类别        | 函数                                | 作用                      |
| --------- | --------------------------------- | ----------------------- |
| **启动流程**  | `communication()`                 | 开机通信握手（检查加密、发起 FPGA 通信） |
|           | `startEnterMainPage()`            | 初始化参数下发完毕，进入主页          |
|           | `dccryptCheck()`                  | 加密授权校验（定时器触发）           |
|           | `waitSetFpgaMode()`               | 等待 FPGA 模式切换完成          |
| **页面控制**  | `PageChanged(pageId)`             | 页面切换时调整顶部/底部栏显隐         |
|           | `setBottomEnable(bool)`           | 控制底部阀门按钮是否可用            |
|           | `refreshBottomStatus()`           | 刷新底部供料/阀门状态             |
| **报警/事件** | `alarmTurnOffFeedSlt()`           | 气压低 → 停止供料、停止分选、关阀      |
|           | `alarmTurnOnFeedSlt()`            | 气压恢复 → 恢复供料、开始分选、开阀     |
|           | `alarmColorVoiceOnSlt()`          | 颜色报警开启                  |
|           | `alarmColorVoiceOffSlt()`         | 颜色报警关闭                  |
|           | `lostFpgaPowerOffSlt()`           | FPGA 掉电 → 关机流程          |
|           | `warnDccryptSlt()`                | 加密到期 → 弹窗提醒关机           |
| **擦拭/维护** | `myWipeStartSlt()`                | 自动擦拭流程                  |
|           | `infoStartValve()`                | 阀门测试                    |
|           | `infoStartIpcFilterCottonClean()` | IPC 滤棉清理                |
| **远程升级**  | `infoUpdateZKSort(QVariant)`      | 远程升级 zksort 主程序         |
|           | `infoUpdateIpcModel(QVariant)`    | 远程升级 IPC 模型             |
|           | `infoUploadIpcInfo()`             | 上传 IPC 信息               |
| **其他**    | `LoadSkin()`                      | 加载 QSS 皮肤文件             |
|           | `keyPressEvent()`                 | F8 键 → 重新加载皮肤           |
|           | `retranslateUiWidget()`           | 语言切换回调                  |

---

## 五、ZKSort 的角色定位

ZKSort 是这个系统的 **"总接线员"**：

```
    后台线程                  ZKSort                    UI 组件
    ─────────               ────────                  ────────
updateStatusThread  ──信号──→ alarmTurnOffFeedSlt() ──→ bottomWidget 刷新
updateStatusThread  ──信号──→ lostFpgaPowerOffSlt() ──→ 关机
updateStatusThread  ──信号──→ myWipeStartSlt()      ──→ 擦拭流程
myMqttMsgParaseThread─信号──→ infoUpdateZKSort()     ──→ 下载弹窗
    QTimer           ──信号──→ dccryptCheck()         ──→ communication()
                          │
    g_MainManager()  ←──PageChanged()  ← 页面切换时调整顶部/底部栏
                          │
    ZKSort 自己不做业务计算，它负责：
    1. 连接后台线程的信号到槽函数（信号槽接线）
    2. 协调顶部栏/底部栏的显隐（页面切换调度）
    3. 管理启动流程（communication → startEnterMainPage）
    4. 处理全局按键（F8 换肤）
```

在zksort构造的时候，创建了一个100ms的定时器，主函数中exec()在等待，
所以exec()获得的第一个需要处理的操作就是定时器结束后的加密器校验也就是[[3 dccryptCheck()]]