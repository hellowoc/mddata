```
WifiConfigWidget::WifiConfigWidget(QWidget* parent):
WifiConfigWidget::~WifiConfigWidget()
```
构造函数，首先初始化继承的父类，然后new一个ui界面，该界面也是通过qtdesigner创建的，随后用setupui将designer中拖拽的界面创建出来，放到内存中等待show（）

析构函数依旧只删除这个ui界面

QTimer(this)，这里把this作为入参，从而当this被销毁的时候，qtimer创建的新对象也被销毁，不需要额外去管理内存
定时器在这里只是创建，然后绑定了一个槽函数，具体的配置在后面启动的时候才会配好

随后一长段是对各个控件的输入配置和使能

[[WifiInterface]]

```
void WifiConfigWidget::showPage(bool bshow)
```
在不同模式下，启动不同的定时器，在wifi的模式下，刷新链接信息；在ap模式下，刷新谁链接了我

showpage在各个界面的cpp中都是用来控制资源的启动和停止，show用来控制界面的显示

```
void WifiConfigWidget::retranslateUiWidget()
```
[[语言系统]]

```
void WifiConfigWidget::on_pwdEdit_textChanged(const QString& pwd)
```

当这个框中的文字发生改变的时候就会触发，这个入口参数就是这个触发信号自带的参数[[提示与调试方法汇总]]

```
void WifiConfigWidget::singleShotFunc()
```
当点击链接按钮的时候，会在定时器的超时槽函数中绑定这个函数，其作用就是链接成功了，就通过dhcp自动拿ip，失败了就等4秒，然后使能两个按钮

