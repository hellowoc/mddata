# `waitSetFpgaMode()` 详解

## 触发链路

```
communication()
    │
    ├── myFlow.resetCommunication()   ← 尝试和 FPGA 握手
    │
    ├── 检查 bFpgaModeAutoSwitchErr
    │     ├── == 0 (正常) → 直接进主页
    │     └── == 1 (异常) → myFlow.resetFpgaRunModeRG()   ← 复位 FPGA 运行模式
    │                       mytimerWaitFpgaMode->start(20000) ← 等 20 秒
    │
    └── 20 秒后...
              ↓
         waitSetFpgaMode()   ← ★ 这个函数
```

**触发条件：** FPGA 启动时自动切换运行模式失败。这是一个异常恢复路径——"模式没切过来，先复位一下等 20 秒，再试一次"。

---

## 逐行分析

```cpp
void ZKSort::waitSetFpgaMode()
{
    mytimerWaitFpgaMode->stop();   // ① 停止定时器（单次触发）
```

定时器是一次性的——到了就停，不重复触发。

---

```cpp
    g_infoWidget().setLabelText(g_myLan().ids_reset);  // ② 显示"重置中..."
    myFlow.resetMachineConfigRG();                      // ③ 重置机器配置
    myFlow.resetSensorConfigRG();                       // ④ 重置传感器配置
    int ret2 = myFlow.resetCommunication();             // ⑤ 再次尝试通信握手
    struCnfg.nBoardStatusAlarm = ret2;                  // ⑥ 保存握手结果
    g_infoWidget().hide();                              // ⑦ 隐藏提示
```

这是第二次尝试连接硬件的流程——先重置配置（恢复默认值），再重新通信。

---

```cpp
    if(ret2 == 0 || struCnfg.nPlcCtrl == 1 || struCnfg.nDefaultPlcCtrl == 1)
    {
        g_MainManager().ShowWidget(SM_SELF_CHECK_WIDGET);     // ⑧ 进自检页
    }
    else
    {
        g_MainManager().ShowWidget(SM_SELF_CHECK_WIDGET);     // ⑨ 还是进自检页
    }
}
```

**if 和 else 执行的是完全一样的代码。** 这不是 bug——这是说"不管第二次通信成不成功，都进自检页面"。原先可能两个分支做不同的事，现在统一为都进自检页。

---

## 和正常路径的对比

```
正常路径 (bFpgaModeAutoSwitchErr == 0):
  communication() → resetCommunication 成功 → startEnterMainPage()
                   → 直接进入主页

异常路径 (bFpgaModeAutoSwitchErr == 1):
  communication() → 检测到 FPGA 模式切换失败
                  → resetFpgaRunModeRG() 复位 FPGA
                  → mytimerWaitFpgaMode->start(20000) 启动 20 秒倒计时
                  → 20 秒后 waitSetFpgaMode()
                  → resetMachineConfigRG() + resetSensorConfigRG() + resetCommunication()
                  → ShowWidget(SM_SELF_CHECK_WIDGET) 进自检页
```

正常路径进入主页（一切就绪，可以直接分选），异常路径进入自检页（需要工程师检查 FPGA 状态）。

---

## 为什么等 20 秒

FPGA 的重新配置不是瞬间完成的。`resetFpgaRunModeRG()` 通过串口发指令让 FPGA 重新加载运行模式，FPGA 内部需要时间重新配置逻辑单元。20 秒是一个足够宽裕的等待时间。
