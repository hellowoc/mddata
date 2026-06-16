# bus/ 通信层

## 一、模块总览

```
bus/ 目录 — 所有硬件/网络通信的代码

六大通道:
  ① 串口 (myqextserialport)  ───→ FPGA/控制板/接口板  (主干道)
  ② TCP  (mynetwork)         ───→ 相机板(图像/波形)
  ③ UDP  (myudpthread)       ───→ IPC 智能相机
  ④ TCP  (myfastnetwork)     ───→ 高速图像传输
  ⑤ MQTT (mqttsrv)           ───→ 云平台/远程管理
  ⑥ HTTP (myhttpfileclient)  ───→ 文件下载/升级

公共组件:
  myqueue ─── 线程安全的环形队列(串口数据中转)
  myeeprom ─── EEPROM 读写(加密/语言存储)
```

---

## 二、串口通信 — 主干道

串口是上位机与所有硬件板卡之间**最主要的通信通道**：

```
上位机(Qt 程序)
    ├── /dev/ttyS1 (串口1) ──→ FPGA 主板 + 相机板 + 接口板
    └── /dev/ttyS2 (串口2) ──→ 控制板 + 恒流源板
```

### 四线程模型

```
GlobalFlow → myQueue.push(packet)
    ↓
mySerialSendThread: pop() → write(串口)
    → 串口硬件 →

mySerialRecvThread: read(串口) → 解析帧头 → 匹配命令码 → 存回包
```

收发分离的原因：串口是全双工的，发出去后硬件需要时间处理，发线程和收线程各跑各的。

---

## 三、通信协议

文件头部定义了 70+ 条通信指令：

```
命令编码体系:
  0x0001~0x000F  系统命令  (加密、机型、模式、版本、开关机)
  0x0010~0x001F  相机参数  (增益、偏置、像元、校正、偏移)
  0x0020~0x0026  设备操作  (色选开关、清灰、统计、供料)
  0x0027~0x00??  控制/灯光  (供料量、亮度、报警、阀门...)
```

数据包格式（推测）：

```
┌────────┬────────┬────────┬────────┬────────┬──────┬────────┬──────────┐
│ 0xAA   │ 0x55   │ board  │ board  │ cmd    │ len  │ data   │ checksum │
│ 帧头   │ 帧头   │ 类型   │ 编号   │ 指令码 │ 长度 │ 数据   │ 校验     │
└────────┴────────┴────────┴────────┴────────┴──────┴────────┴──────────┘
```

---

## 四、myQueue — 线程安全的环形缓冲区

```cpp
#define QUEUE_SIZE 5000

typedef struct _SqQueue {
    char data[QUEUE_SIZE][SEND_PACKET_SIZE];  // 5000 个槽
    unsigned int front, rear;                  // 环形指针
} SqQueue;

class MySerialQueue {
    SqQueue *m_queue;
    QMutex m_queueLock;    // 每个 push/pop 都加锁
};
```

数据流向：GlobalFlow(push) → 队列 → 发送线程(pop) → 串口。
[[锁机制]]


## 五、TCP 网络 — 图像/波形数据

串口传参数和指令（几到几十字节），TCP 传图像和波形（几百KB 到几 MB）：

```
相机板
    ├── TCP → MyNetWorkThread   → 波形数据 → MyWaveWidget
    │                           → 图像数据 → ImageWidget
    │                           → 存 BMP 文件
    └── TCP → MyFastNetWorkThread (高速通道)
```

多种采集模式：单行波形、400行图像、连续采图、多张采集、背景波形等。

---

## 六、UDP 通信 — IPC 智能相机

IPC 是独立的智能相机，运行自己的 Linux 系统和 AI 模型：

```
UDP: 上位机 ←→ IPC 智能相机

指令: 心跳查询、加载模型、开始检测、配置 AI、关机重启...
```

---

## 七、MQTT / HTTP

- **MQTT**: 连接云平台，上报状态/报警，接收远程指令，OTA 升级
- **HTTP**: 从升级服务器下载固件/模型文件

---

## 八、完整数据流

```
ZKSort / 业务 UI
        ↕
  GlobalFlow (翻译官: 读结构体 → 组包 → 发队列)
        │                    │                    │
   myQueue.push()     QTcpSocket          QUdpSocket
        │                    │                    │
        ▼                    ▼                    ▼
  串口发送线程          TCP 图像线程         UDP IPC 线程
   (指令/参数)          (波形/图像)          (AI/模型)
        │                    │                    │
        ▼                    ▼                    ▼
  FPGA/控制板/接口板      相机板              IPC 智能相机
```
