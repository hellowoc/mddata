```
IPSetWidget::IPSetWidget(QWidget *parent)
IPSetWidget::~IPSetWidget()
```

构造这个类，然后初始化会用到的参数
这里的析构函数只析构了ui，但是没有让ping停止

其中initui就是给各个控件设置默认值

```
void IPSetWidget::initUi()
```

```
void IPSetWidget::showPage(bool bshow)
```
[[references/QSettings|QSettings]]
给两个eth写入初始值

当ipstr1为空时，隐藏eth2的相关参数框，不为空就写入余下的初始值

当不是工程师模式的时候，设置一些控件可见

```
void IPSetWidget::retranslateUiWidget()
```
本工程有语言系统[[语言系统]]这里是在适配

```
void IPSetWidget::on_okBtn_clicked()
```
设置对应的槽函数其中的g_infoWidget().delayShow();是因为
这个函数本身就是通过事件循环调用进来的，当前的事件循环没有结束，所以无法处理接下来的事件循环，而此处用了g_infoWidget().delayShow();，在这个函数中设置了10ms的处理时间供后面的操作和刷盘，同时用
```
QCoreApplication::processEvents(QEventLoop::AllEvents,100);
```
里面的processevents来实现先把后续的任务完成这一操作。

也就是说，原本得等按键触发之后的处理函数跑完才能处理后续操作，但是现在可以先处理后续的操作，不用等待on_okBtn_clicked（）跑完

读取用户输入的内容，然后刷入磁盘

之后用全局变量[[5 （通信）GlobalFlow|myFlow.sync]]来更新状态，再把窗口隐藏，然后设置不可修改，之后更新
```
void IPSetWidget::updateUi()
```
就是把所有控件的修改都禁止
void IPSetWidget::on_checkBox_clicked(bool checked)控制ip能否修改的框


```
void IPSetWidget::on_btn_nettest_clicked() //按下网络测试按钮之后
```
同样的弹出来提示框，g_infoWidget()是一个全局可用的提示框，在不同的时候，给提示框输入不同的内容[[提示与调试方法汇总]]
这里的process相当于是用来执行linux的指令的，其中，system也可以二者区别如下

|     | `system()` | `QProcess`    |
| --- | ---------- | ------------- |
| 返回值 | 只有退出码      | 能拿到命令的输出      |
| 控制  | 发了就不管了     | 可以等、可以杀、可以读输出 |
如果是用system（），就是system（"ping -w 5 ntp.ntsc.ac.cn"）

获得了ping的返回值之后，检查里面有无TTL，从而判断是否链接成功

```
void IPSetWidget::on_btn_connect_clicked()
```
连接按钮，通过[[g_Runtime解析]]来获得vpn的ip，如果没有连上vpn就连openvpn

同样的弹出提示框，然后delayshow

通过connetToServer来判断是否链接成功
```
bool IPSetWidget::connetToServer()
```
该函数中用到getlocalvirtualaddr来获得本机虚拟地址
```
bool IPSetWidget::getLocalVirtualAddr()
```
用一个qnetworkinterface来装载所有的网卡信息，系统有一个网卡，这个list就会有一个参数，随后遍历，看有没有tun0这个网卡
再用QNetworkAddressEntry来查找这个网卡上的所有ip地址，遍历找到ipv4地址，显示这个ip
```cpp
void GlobalFlow::sleep(int secs)
{
    QTime dieTime = QTime::currentTime().addSecs(secs);   // 计算终点时间
    while (QTime::currentTime() < dieTime)                 // 循环直到时间到
    {
        QCoreApplication::processEvents(QEventLoop::AllEvents, 100);
    }
}
```

此处同样的使用sleep来做一个假休眠。等待链接成功，在延迟的时间内，每100ms轮询一次队列中的其他线程


链接成功之后，设置各个控件的使能，如果链接失败，则给出提示框

如果有vpn地址，则直接获得getLocalVirtualAddr

本文档的学习过程在[[学习路径（以IPSetWidget）为例]]中