```
void mqttsrv::on_subscribe(int mid, int qos_count, const int *granted_qos)
void mqttsrv::on_message(const struct mosquitto_message *message)
```

首先是订阅和有消息传进来的响应函数，当订阅之后，会在中断给出调试提示，当有消息时，会进行二次确认，保证收到的信息和订阅的主题是一致的。

通过[[锁机制]]来上锁，写入消息队列

```
mqttThread::mqttThread(QThread *parent) : QThread(parent)
```
构造函数
取设备编号，创建订阅主题
如果设备名不为空，则创建一个mqttsrv的对象test

run中：
```cpp
mosqpp::lib_init();                     // 初始化 mosquitto 全局状态
test->tls_opts_set(1, "tlsv1", NULL);  // 启用 TLSv1 加密
test->tls_insecure_set(true);          // 跳过服务器证书域名验证
rc = test->tls_set("./root.crt", NULL);// 加载 CA 根证书（验证服务器不是假冒的）
```
这一段是在做TLS配置

之后用setting读取配置文件中的云端地址
```
rc = test->connect(testip.toLocal8Bit().constData(),1884,50);
```
用test（构造函数最后构造出来的test）的connect来链接brocker
```cpp
rc = test->loop();    // ★ mosquitto 核心函数
        // loop() 每调用一次：
        //   - 从 socket 读数据（如果有）
        //   - 如有消息 → 调 on_message() 回调 → 消息放入 recvMsgList
        //   - 发送待发的 MQTT 包
        //   - 发送 keepalive PING 包（如果距上次超过 50 秒）
        //   - 返回值：0=正常，非 0=连接断开
```

```
mqttMsgParaseThread::mqttMsgParaseThread(QThread *parent) : QThread(parent)
```
先初始化各个参数，然后数据转换，之后链接四个信号和对应槽函数

```cpp
// 1. 远程控制锁屏信号
connect(this, SIGNAL(StartRemoteControl(QString)),
        this, SLOT(DealStartRemoteControl(QString)));
// 触发链：doParseMessage → onRemotecontrol → emit StartRemoteControl(phone)
//                           → DealStartRemoteControl(phone)
//                             → g_MainManager().ShowWidget(SM_REMOTE_CONTROL_WIDGET)
//                             → 主线程弹出锁屏确认页

// 2. FPGA 图像采集完成信号
connect(myFlow.myNetWork->udpthread, SIGNAL(SignalImageRecvFinished(int, QString)),
        this, SLOT(imgaeCaptureOver(int, QString)));
// 触发链：MyNetWorkThread 收到完整图像 → 发射信号 → imgaeCaptureOver(height, path)
//                           → n_uploadImageFlag = 1
//                           → 主循环检测到 → uploadCaptureImage()
//                           → HTTP 上传图像到云端

// 3. FPGA 波形采集完成信号
connect(myFlow.myNetWork->udpthread, SIGNAL(SignalWaveRecvFinished()),
        this, SLOT(waveCaptureOver()));
// 触发链：MyNetWorkThread 收到完整波形 → 发射信号 → waveCaptureOver()
//                           → n_uploadWaveFlag = 1
//                           → 主循环检测到 → uploadCaptureWave()

// 4. FPGA 喷阀计数采集完成信号
connect(myFlow.myNetWork->udpthread, SIGNAL(SignalBlowCountsRecvFinished()),
        this, SLOT(blowCountsCaptureOver()));
// 触发链：MyNetWorkThread 收到喷阀计数 → 发射信号 → blowCountsCaptureOver()
//                           → n_uploadBlowCountsFlag = 1
//                           → 主循环检测到 → uploadBlowCounts()
```

```
void mqttMsgParaseThread::run()
```
先等待token认证

然后上报数据

死循环1s一次，更新token
轮询15个标志位:
把当前的状态同步到云端

如果消息队列不为空，取队列第一个数据，然后删除队头

```
void mqttMsgParaseThread::doParseMessage()
```
解析收到的json数据，
检查是否被远程操控，通过电话号码检查是不是同一个人
回传ack信号
三种危险情况不执行
8个界面不执行
switch结构分发指令

收到命令之后的主要流程是进入循环中对应的分支，然后进入到分支函数执行对应的操作：参数格式匹配，验证参数格式，执行全局变量修改，myflow[[全局单例]]更新全局参数或是实现对应操作[[MQTT命令执行-CMD_Feed详解]]

```
int mqttMsgParaseThread::onParaUpload(QString cmd_value)
```

这个函数对应的操作是 HTTP 数据上传（耗时长、可能阻塞），不适合在 `doParseMessage` 中同步执行。所以只返回 SUCCESS + 设标志位 `n_uploadRealTimeParaFlag = 1`，让主循环在下个周期异步执行


```
void mqttMsgParaseThread::onUpdateZKSort(string cmd_value)
```
MQTT 收到"去这个 URL 下载文件"的通知 → mqttMsgParaseThread 只做参数解析 → 把下载任务丢给主线程 → 主线程的 `httpDownloadManager` 用异步信号完成实际下载。[[HTTP大文件下载-断点续传]]

# 总结

首先构造函数中new一个test然后用test来链接brocker，mqttMsgParaseThread创建线程，随后run()获得信息队列中的第一个数据，doParseMessage()解析各个数据，分发给对应的函数，函数中通过全局参数和全局单例来实现远程控制，对于下载和上传任务交给http处理

```
1. 构造函数（主线程调用 globalflow.cpp）
   mqttThread:
     → 读取设备编号 → new mqttsrv("ZKGDDEV47") → 这就是 test
     → topic = "/ZKGDDEV47"

   mqttMsgParaseThread:
     → 构建固件版本字符串
     → 连接采图完成信号（MyNetWorkThread → imgaeCaptureOver 等）
     → 连接远程控制信号（StartRemoteControl → DealStartRemoteControl）

2. start() → 两个线程同时启动

3. mqttThread::run()
   → ping 外网 → TLS 配置 → connect("cloud.chinaamd.cn", 1884, 50)
   → subscribe("/ZKGDDEV47") → while(1) loop() 死循环收消息
     → 收到消息 → on_message() → 加锁放入 recvMsgList

4. mqttMsgParaseThread::run()（同时进行）
   → 等 10 秒让 mqttThread 先连上
   → 等 networkConnectIsOk == 1
   → requestTokenUpdate() 拿 OAuth2 Token
   → 初始上报（uploadPara, uploadLogFile...）
   → for(;;) 死循环：
       ├── Token 续期
       ├── 15 个标志位轮询 → 异步上传（HTTP）
       └── while recvMsgList 非空
             → msg = recvMsgList[0]
             → doParseMessage()
                 → JSON 解析 → 提取 cmdCode/content/uuid
                 → 特殊命令：CMD_LockStatusUpload / CMD_ConnectVPN
                 → 先回 ACK（RECEIVE）
                 → 4 层状态检查（清灰？采图中？特殊页面？）
                 → 20+ 个 if-else 分发到 onXxx()
                   ├── onCmdFeed → 写 struCnfp → myFlow.resetFeederRG() → 串口
                   ├── onRemotecontrol → 改 remoControlmode → emit 信号 → 锁屏页
                   ├── onShutDown → system("shutdown -h now")
                   ├── onUpdateZKSort → emit cmd_updatezksort_ask → httpDownloadManager
                   └── onParaUpload → n_uploadRealTimeParaFlag = 1（异步）
                 → onCmdReturn() → publish "MqttServerClient" → 云端收到结果
             → 删队头 msleep(200)

```