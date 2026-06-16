new一个process，判断有没有weifi配置文件

没有的话就给出提示[[提示与调试方法汇总]]

执行对应的命令行给出返回值，如果失败的话给出报错

deleteLater等待程序跑完后面的事情再删除
新建一个

```
QByteArray WifiInterface::execCmd(const QString& cmd)
```
这个函数是用来执行命令的
```
WifiInterface* WifiInterface::instance(QObject *parent)
```
这个函数创造一个单例，因为wifi板卡只有一个，全局用的时候都是用这一个
```
bool WifiInterface::setWifiSsidAndPwd(const QString& ssid, const QString& pwd)
```
用shell指令来添加wifi的名称和密码
```
bool WifiInterface::connectWifi(const QString& ssid)
```
用shell来链接wifi

后面关于wifi的诸多信息也是同样通过shell来获得
```
bool WifiInterface::setDhcpState(bool state)
```
这个是分配IP端口，就不需要自己再配置了，
wifi是一个局域网，路由器来给这个网段下面的各个设备分配地址，然后通信，只要进入这个wifi，各个设备之间就能够通信，dhcp自动给本机配置一个再路由器中注册过的地址，我以这个地址向其它的ip发消息

在wifi中，各种链接过程基本上都是通过shell指令完成

本工程支持本机当成路由器使用（ap模式），第三方设备链接后，用于调试