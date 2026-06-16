头文件中完成了对HTTP应答状态代码的宏定义

声明了两个类，一个是MyHttpFileClient一个是httpDownloadManager

进入到cpp文件：

构造函数中，初始化资源定位符和token令牌

URL是资源定位符号，用来在网络上找到某个文件的完整地址
```
bool MyHttpFileClient::downLoadFile(QString url, QString path)
```
下载文件到固定地址

配置http的文件格式，设置URL和数据头

手动将异步变成同步，管理器发出完成信号的时候事件回环才退出（此处只是链接信号）
manage.get(request时开始接收，然后再eventLoop.exec，停在这里，直到接受结束。

```
bool MyHttpFileClient::canWebsiteVisit(QString url)
```
测试网络是否正常

timer.setSingleShot，定时器只触发一次

http通信过程：
用manage来管理首发操作，用request来包装数据格式，然后manage返回reply信息

```
void MyHttpFileClient::requestToken(QString url)
bool MyHttpFileClient::upLoadFile(QString url, QString path)
```
`requestToken` 是**注册**——设备还没拿到自己的 Token，用一个硬编码的 app 凭证（`client_id` + `client_secret`）去认证，相当于"我是色选机固件，请给我一个 Token"。云端返回 Token + 这个 Token 绑定的设备编号。

`requestTokenUpdate` 是**续期**——Token 过期了，用设备编号 `ZKGDDEV47` 告诉云端"我是上次那个设备，给我续一下"。不需要再传 `client_id`，因为 Token 绑定的设备身份已经确定了。

```
bool MyHttpFileClient::upLoadOssFile(QString fileName)
```
http的数据传输格式，首先是头，然后是数据，其格式是
头
头
头
空行（告诉对方头结束了，下面是数据）
数据
数据
数据

而数据的格式可以有三种[[HTTP普通请求体-QByteArray]]
```
    ├── QByteArray data  → 普通文本（JSON、表单）
    ├── QHttpMultiPart   → 多部件（文本字段 + 文件混传）
    └── QFile *file      → 直接传文件（本工程没用）
```
封装好数据之后用appand将QHttpPart结构体写入QHttpMultiPart
[[HTTP多部件格式-QHttpMultiPart]]

# 第二部分 httpdownload manager

第一部分是小文件的传输和解析以及相关的函数支持
第二部分主要是用来做大文件的解析工作，支持断点续传

```
void httpDownloadManager::startDownload()
```
先检查是不是断点续传，做好对应的处理工作
是：从上次下载的位置开始继续下载
不是：删除临时文件（如果有的话）然后继续下载

配置好http的头
发出get
连接各个槽函数
{
槽函数在TCP传输层触发[[HTTP大文件下载-断点续传]]
```
Linux 内核 TCP 栈
  → 数据到达 → 标记 socket 可读
    → Qt 事件分发器 (QAbstractEventDispatcher)
      → 检测到 socket 可读事件
        → QNetworkReply 内部调用 QAbstractSocket::read()
          → 数据写入 QNetworkReply 的内部缓冲区
            → 发射 readyRead() 信号
              → downloadReadyRead() 槽执行

    全部接收完毕：
      → 发射 finished() 信号
        → downloadFinished() 槽执行

    传输过程中周期性更新计数器：
      → 发射 downloadProgress(bytesReceived, bytesTotal)
        → downloadProgress() 槽执行
```

| 信号                 | 底层触发条件                               |
| ------------------ | ------------------------------------ |
| `readyRead`        | TCP 收到新数据，放入 socket 接收缓冲区，Qt 从缓冲区读出后 |
| `downloadProgress` | Qt 内部更新已接收字节计数器时                     |
| `finished`         | TCP 连接收到 FIN 包或所有数据已接收完毕             |
}

## 小结

1、手动同步，用一个定时器的槽函数来将http的收发进行同步

2、多次重复出现的http协议初始化的代码，就是在拼装数据格式，用get和post来上传和下载数据

3、http返回的数据是字节流，由客户端自行按照格式处理[[HTTP响应数据处理]]