```
TopWidget::TopWidget(QWidget *parent) :
TopWidget::~TopWidget()

```

构造函数和析构函数：
m_levelPageId做了一个等级容器，不同的操作等级有不同的权限。
ipcstatus做了一个数组，方便用一个循环配置9个状态灯的内容，后续在初始化获得status之后会根据实际情况点亮对应的灯

# 调用到updateDateTime函数


```
void TopWidget::updateDateTime()
```


## 调用到updateIpcStatus
```
void TopWidget::updateIpcStatus()
```
确认这个机型有无ipc，检查ipc工作状态

0-3分别是：未加载，直接berak
加载中，break
加载成功，显示物料名称，
加载失败

如果没有ipc相机，则直接显示物料的名称
最后根据是否有ipc来控制状态栏的可见性


回到updateDateTime
通过QDateTime::currentDateTime()向操作系统拿时间
秒和日期混在一起，所以取前两位获得秒
在特定时间完成日志的撰写和坏点计数的清零

ui->alarmLbl->setVisible(struGsh.bAlarmStatus && (struGsh.nCounter%2 == 0));用奇偶判断来实现灯闪烁

检查每一层，每一个视野，每一个相机是否工作，只要有一个工作异常，就亮红，否则亮绿灯


回到构造函数：
链接槽函数到[[mythread.c]]，每秒触发一次

创建一个定时器，连点的时候这个定时器启动，如果1s内点击超过了5次，那么进入隐藏的界面

```
void TopWidget::reloadTitleMap()
```
相当于是语言切换功能，这里不直接显示是因为不同的界面，显示的控件不一样，所以不能直接全部显示出来，后面要显示谁，直接调用就好了

```
void TopWidget::PageChanged(int pageId)
```
通过id来实现，不同界面显示不同的控件的功能
函数的最后用QWidget::update()来立刻刷新

```
void TopWidget::showPage(bool bshow)
```
此处的showpage由于界面一直存在，所以不需要写false的情况

```
void TopWidget::mousePressEvent(QMouseEvent *event)
```
鼠标按下 -> 通过事件来触发定时器 -> 定时器timeout通过绑定槽函数检查点击次数 -> 达到条件 -> 显示界面

```
void TopWidget::paintEvent(QPaintEvent *e)
```
这里需要绘制顶部的图像，所以自定义绘制了


```
void TopWidget::on_SysStateWidgetBtn_clicked()
```
这里就和g_MainManager[[4 （界面管理）MiddleWidgetManager]]文件中对应上了，根据pageid来显示对应界面

```
void TopWidget::on_screenshotBtn_clicked()
```
