# GlobalFlow 全局流程控制器

## 一、它不是传统意义的状态机

传统状态机有显式的 `enum State` + `switch(state)` 转换表。GlobalFlow 不同——**没有显式的状态转换结构**。它的本质是一个超大的硬件操作工具箱（100+ 个函数），按顺序调用来完成设备初始化、通信、配置下发。

---

## 二、本质：过程式流程控制器

```
GlobalFlow = 硬件指令翻译官 + 配置读写 + 流程编排

职责:
  ① 把 C++ 结构体的参数值 → 翻译成串口数据包 → 发给 FPGA/传感器/接口板/相机
  ② 从串口接收数据 → 解析 → 更新全局状态结构体
  ③ 管理开机流程的执行顺序
  ④ 提供工具函数（延时、排序、颜色转换等）
```

---

## 三、initAll() 完整调用链

```cpp
void GlobalFlow::initAll()
{
    // 第1步: 读配置文件
    getSetting();           // QSettings 读取 APP 配置
    getParamsFileStaus();   // 检查参数文件状态
    getGlobal();            // JSON → struCnfg
    getProfile();           // JSON → struCnfp

    // 第2步: 初始化全局共享状态
    initGsh();
    initIpcAttr();

    // 第3步: 打开串口
    mySerial.comOpen(struGsh.serialNum);       // 串口1 (FPGA)
    mySerial.comOpen(struGsh.serialNum_2);     // 串口2 (其他板)

    // 第4步: 创建并启动通信线程
    mySerialSend = new mySerialSendThread;   mySerialSend->start();
    mySerialRecv = new mySerialRecvThread;   mySerialRecv->start();
    mySerial2Send = new mySerial2SendThread; mySerial2Send->start();
    mySerial2Recv = new mySerial2RecvThread; mySerial2Recv->start();

    // 第5步: 创建状态轮询线程
    myUpdateStatusThread = new updateStatusThread;
    myUpdateStatusThread->start();
    // ...
}
```

完全是过程式的初始化序列。

---

## 四、与硬件交互的核心模式

所有跟 FPGA/硬件通信的函数都遵循同一模式：

```cpp
int GlobalFlow::resetCommunication()
{
    // 1. 初始化局部结果数组
    int cameraResult[MAX_LAYER][MAX_VIEW][MAX_VIEW_UNIT];

    // 2. 遍历每层每视每单元
    for (int i = 0; i < MAX_LAYER; i++) {
        for (int j = 0; j < MAX_VIEW; j++) {
            // 组包: 填充串口指令数据
            memset(data, 0, SEND_PACKET_DATA_SIZE);
            // ...

            // 发送: 通过串口线程发出
            mySerialSend->sendData(data, boardType, boardNum);

            // 等待: 延时等待硬件响应
            sleep(1);

            // 接收: 从接收线程队列取回包
            // 检查: 解析回包，判断通信是否成功
        }
    }

    // 3. 汇总结果 → 返回成功/失败
    return result;
}
```

---

## 五、函数分类

| 类别 | 函数数量 | 代表函数 | 作用 |
|------|---------|---------|------|
| **初始化/启动** | ~10 | `initAll()`, `initGsh()`, `initSendAllParams()` | 开机流程 |
| **串口通信** | ~50 | `resetCommunication()`, `resetCameraRG()`, `resetEjectTimeRG()` | 向硬件发送参数 |
| **配置读写** | ~20 | `getGlobal()`, `saveGlobal()`, `getProfile()`, `saveProfile()` | JSON ↔ 结构体 |
| **算法参数** | ~15 | `resetAllArithParams()`, `setSvmParas()`, `materialParamsCopy()` | 分选算法配置下发 |
| **IPC 相机** | ~25 | `configIpcClassParams()`, `loadIpcModel()`, `getIpcInfo()` | 智能相机管理 |
| **工具** | ~10 | `sleep()`, `msleep()`, `getHsv()`, `dataSort()` | 通用工具函数 |

---

## 六、延时函数的特殊性

```cpp
void GlobalFlow::sleep(int secs)
{
    QTime dieTime = QTime::currentTime().addSecs(secs);
    while (QTime::currentTime() < dieTime) {
        QCoreApplication::processEvents(QEventLoop::AllEvents, 100);
        //  ↑ 关键：等待期间继续处理事件循环！
        //  不调这个 → 窗口卡死
    }
}
```

这不是 `QThread::sleep()`（完全挂起线程），而是在等待期间继续处理事件循环——定时器、串口数据、界面刷新都能正常进行。

---

## 七、和 ZKSort 的关系

```
ZKSort (决策者 — 什么时候做什么)
  ├── communication():
  │     myFlow.resetCommunication()     ← 握手
  │     myFlow.resetFpgaRunModeRG()     ← FPGA 模式
  │
  ├── startEnterMainPage():
  │     myFlow.initSendAllParams()      ← 下发所有参数
  │
  └── 各 slot:
        myFlow.resetSortOnOff()         ← 开关分选
        myFlow.resetFeederRG()          ← 供料
        myFlow.vavleOnOff()             ← 喷阀
        myFlow.saveGlobal()             ← 保存配置

GlobalFlow (执行者 — 怎么做：组包、发串口、等回包、解析)
```

---

## 八、为什么是 extern 而不是单例

```cpp
// globalflow.h
extern GlobalFlow myFlow;

// globalflow.cpp
GlobalFlow myFlow;    // 直接全局变量，不是 Meyers 单例
```

原因：
- 构造不需要参数
- 不需要懒加载（程序启动就需要）
- 所有状态通过 `struGsh`、`struCnfg` 等全局结构体共享
- 不需要 `g_xxx()` 的函数调用开销

---

## 理解

实现各种工作（例如分选，识别）的是对应的函数中的算法，这些算法在操作的时候会去修改下位机寄存器中的各种参数，从而实现算法的目标功能。同时后台线程会一直轮询，把硬件的新状态写入结构体中。
globalflow就是负责更新上下位机的状态，完成上位机的结构体和下位机的二进制寄存器之间的数据通信和更新。

而这些通信是依赖[[（通信层）bus]]来完成的