```
MyNetWorkThread::MyNetWorkThread(QThread *parent) : QThread(parent)
```
构造函数中初始化各个参数，然后写内存

该函数主要是用来进行图像的数据传输，由于fpga是纯硬件逻辑，不支持TCP/IP协议，所以只能用udp协议直接传输