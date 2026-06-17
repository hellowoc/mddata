# QObject 及其在本项目中的衍生类族谱

---

## QObject — 一切根基

`QObject` 是 Qt 框架的**元对象系统根基**，提供三个核心能力：

| 能力 | 说明 |
|------|------|
| **信号与槽**（signal/slot） | 对象间通信机制，松耦合的事件驱动 |
| **对象树**（parent-child） | 父对象销毁时自动销毁所有子对象，内存管理 |
| **元对象反射**（`Q_OBJECT` 宏） | 运行时类型信息、动态属性、`qobject_cast` |

本项目所有自定义类，最终都追溯到 `QObject`。

---

## 本项目继承树全景

```
QObject
├── QCoreApplication
│   └── QApplication
│       └── zkApplication          ★ 应用入口 + 背光管理
│
├── QThread                        ★ 工作线程分支（15+ 子类）
│   ├── updateStatusThread         状态轮询（喂料/气压/加密...）
│   ├── mySerialSendThread / mySerialRecvThread     串口收发
│   ├── mySerial2SendThread / mySerial2RecvThread    第二路串口
│   ├── MyNetWorkThread / MyNetCptureThread          网络图像传输
│   ├── MyFastNetWorkThread                          高速网络
│   ├── mqttThread / mqttMsgParaseThread             MQTT收发/解析
│   ├── queryStatisticThread                         产量统计查询
│   ├── SendThread / IpcLoadThread                   IPC通信
│   ├── vavleTestThread                              阀门测试
│   ├── ipcTestNetThread / pingThread / SysInfoDebugThread   网络诊断
│   ├── getBadPointsThread                           IPC坏点查询
│   └── AcsUpdateThread                              自动校正更新
│
├── QObject                          ★ 纯逻辑对象分支（无界面）
│   ├── Runtime                      OS抽象层（文件/命令/USB）
│   ├── MiddleWidgetManager          页面管理器（核心调度）
│   ├── GlobalFlow                   全局流程状态机 (551KB)
│   ├── MyLanguage                   多语言管理 (96KB)
│   ├── MyFsCheck                    文件系统检查
│   ├── MySqlite                     SQLite数据库
│   ├── MyUdp / BackgroudObj / FastNetObj    UDP通信
│   ├── PlcAutoCtrlManager           PLC自动控制
│   ├── WifiInterface                WiFi接口
│   └── myDelayCode                  加密/授权管理
│
├── QWidget                          ★ 可视控件分支
│   ├── BaseUi                       ＜─自定义UI基类（语言切换+样式）
│   │   └── ZKSort                   主窗口
│   │
│   ├── QDialog                      ＜─对话框分支
│   │   ├── myMessageBox            通用消息弹窗
│   │   ├── MyInputDialog           输入对话框
│   │   ├── DownloadDialog          下载进度弹窗
│   │   ├── AiSetDialog             AI参数设置
│   │   ├── fpga                    FPGA管理
│   │   ├── myDccrypt / myEncrypt   加密/解密界面
│   │   ├── MySetDateTime           日期时间设置
│   │   ├── OneKeyDialog            一键修复
│   │   └── 其他各种弹窗...
│   │
│   ├── QPushButton                  ＜─按钮分支
│   │   └── myPushButton / myToolButton
│   │
│   ├── QLabel → myLabel            ＜─标签分支
│   ├── QLineEdit → myLineEdit      ＜─输入框分支
│   ├── QListWidget → mylistwidget  ＜─列表分支
│   │
│   └── 自定义视觉控件
│       ├── MyWaveWidget            波形显示
│       ├── MySwitchWidget          自定义开关
│       ├── SwitchBtn               切换按钮
│       ├── HsvColorCircle          HSV色环
│       ├── MyArithUi               算法参数面板
│       └── UserSenseSetPage        用户灵敏度设置页
```

---

## 三条分支的职责划分

```
QObject 派生出来后，分化出三条路：

1. QCoreApplication → QApplication
   └── "应用管理者"
       整个程序只有一个实例
       管事件循环、管进程级行为（背光）

2. QThread
   └── "后台工人"
       每个子类是一条独立线程
       管通信、管轮询、管耗时的后台任务
       不直接操作 UI

3. QWidget
   └── "前台交互"
       用户能看见、能摸到的界面
       按钮、列表、波形、对话框、主窗口
```

---

## QThread 分支 — 为什么这么多

这个项目是一个**实时工业控制系统**，多条通信链路必须并行工作：

```
主线程 (Mainthread)
  ├── updateStatusThread     ← 100ms周期轮询硬件状态（喂料/气压/温度...）
  ├── mySerialRecvThread     ← 串口1 接收（FPGA通信，数据量极大）
  ├── mySerialSendThread     ← 串口1 发送
  ├── mySerial2RecvThread    ← 串口2 接收（如果设备有第二路串口）
  ├── mySerial2SendThread    ← 串口2 发送
  ├── MyNetWorkThread        ← TCP 接收 IPC 相机图像数据
  ├── MyNetCptureThread      ← TCP 接收 IPC 抓拍图像
  ├── mqttThread             ← MQTT 长连接（远程管理/云平台）
  ├── mqttMsgParaseThread    ← MQTT 消息解析
  ├── queryStatisticThread   ← SQLite 产量统计查询
  ├── SendThread             ← UDP 指令发送
  ├── IpcLoadThread          ← IPC 模型加载
  └── ...其他诊断/测试线程
```

如果把这些都放在主线程，UI 会完全卡死。QThread 让每条通信链路独立运行，互不阻塞。

---

## QObject 直接子类 — 为什么不需要 QWidget

| 类 | 为什么是 QObject 而不是 QWidget |
|---|-------------------------------|
| `Runtime` | 纯工具类，操作文件系统和 shell，不显示任何界面 |
| `MiddleWidgetManager` | 管理页面切换逻辑，本身不可见，只操作 `QStackedWidget` |
| `GlobalFlow` | 551KB 的业务状态机，纯数据处理 |
| `MyLanguage` | 管理翻译表，不显示界面 |
| `MyUdp` / `FastNetObj` | 网络通信对象，收发包，数据交给 UI 线程显示 |
| `myDelayCode` | 加密校验逻辑，不涉及 UI |

继承 `QObject` 就能用**信号槽**（与其他对象通信）和**对象树**（自动内存管理），不需要 `QWidget` 的绘制能力。这就是 Qt 设计的精髓——**功能和显示分离**。

---

## 信号槽连接 — QObject 子类之间如何协作

以主窗口与后台线程的协作为例：

```cpp
// zksort.cpp — 主窗口中连接信号槽
connect(myUpdateStatusThread, SIGNAL(sWipeStartSig()),
        this,                 SLOT(myWipeStartSlt()));
//      ↑                       ↑
//   QThread 子类             QWidget 子类
//   (后台线程)               (主窗口)
//
// 后台线程检测到"该擦拭了" → 发信号 → 主窗口弹提示/执行擦拭
```

这就是继承 `QObject` 的核心价值——不管你是线程、纯逻辑对象还是可视控件，只要继承 `QObject`，就能用同一套信号槽机制彼此通信。
