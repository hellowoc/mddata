# mainpage/ 主页面群详解

---

## 一、目录定位

`mainpage/` 是用户和工程师直接交互的**业务页面集合**，包含约 30 个 widget。所有页面都继承自 `basewidget`，通过 `MiddleWidgetManager::ShowWidget()` 互相切换。

---

## 二、所有页面的统一模式

```
每个 mainpage 页面都遵循同一套模板：

1. 继承 basewidget（获得视/层/通道/组切换能力）
2. showPage() → 从全局结构体读数据 → 填到控件上
3. 用户操作 → 改全局结构体 → 调 myFlow.xxx() 下发到硬件
4. receiveMsg() → 接收子页面发来的参数变更通知
5. retranslateUiWidget() → 语言切换时刷新所有文字
```

---

## 三、页面分类一览

### 3.1 主页（2 个）

| 页面 | 对应 SM_ 常量 | 说明 |
|------|-------------|------|
| `NewMainPage` | `SM_MAIN_PAGE_WIDGET_NEW` | **当前使用的主页**，Run/Stop 切换 + 导航按钮 |
| `MainPageWidget` | `SM_MAIN_PAGE_WIDGET` | 旧版主页，位图按钮风格 |

**NewMainPage 的核心逻辑**：

```
用户点 Run → 
  检查 IPC 模型是否加载完 → 检查报警状态 → 弹确认框 →
  myFlow.onOff() → 供料开始 → 分选开始

用户点 Stop →
  myFlow.onOff() → 停止供料 → 停止分选 → 关阀

关机 →
  弹确认框 → 停止分选 → myFlow.shutdownIpc() →
  g_Runtime().save() → system("shutdown -h now")
```

---

### 3.2 用户/工程师入口（2 个）

| 页面 | 说明 |
|------|------|
| `UserWidget` | 操作员灵敏度设置——按通道分组显示灵敏度柱状图 |
| `EngineerWidget` | 工程师设置中心——系统时间、权限切换、导航到各高级页面 |

**EngineerWidget 的权限分级**：

```
用户模式:  只能看主页和灵敏度
工程师模式: 输入密码 "123.321" → 可进所有设置页
工厂模式:  输入日期相关密码 → 可进 FPGA 升级等危险操作
```

---

### 3.3 方案/模式管理（5 个）

| 页面 | 说明 |
|------|------|
| `ModeManageWidget` | **方案管理中心**：新建/删除/重命名/复制/锁定/导出/导入/备份 |
| `CreateNewModeWidget` | 向导式创建新方案（选模板） |
| `ModelChangeWidget` | **IPC AI 模型切换**：查看/选择/删除/应用 IPC 模型 |
| `ModelSelectWidget` | 云端模型浏览和下载（通过 MQTT） |
| `ModeSetWidget` | Tab 容器（识别/剔除/灯光/背景子页面） |

**方案 = 一组完整的机器参数**（算法灵敏度、喷阀时间、灯光亮度...），切换方案即切换整个机器的工作模式。方案以文件形式存储在 `/sdcard/cnf/`。

---

### 3.4 供料设置（6 个）

| 页面 | 说明 |
|------|------|
| `FeedSetWidget` | **供料总容器**：按通道/按组/清料三种模式切换 |
| `FeedSetByChutePage` | 逐个通道设置供料量 |
| `FeedSetByGroupPage` | 按组设置供料量 |
| `FeedSetByClearUpPage` | 清料平衡值设置 |
| `FeederAotuAjust` | **自动校准**：称重 → 计数像素 → 计算系数 |
| `FeedBanlanceAimSingle` | 单通道供料偏置微调 |

**FeederAotuAjust 自动校准流程**：

```
1. 关闭擦拭 → 清零统计 → 等 3 秒
2. 开启统计 + 振动器 → 运行 60 秒
3. 关闭振动器 → 等 5 秒 → 读取各通道总像素
4. 用户输入实际称重质量
5. 自动计算系数: n_coff = 总像素 / 质量
```

---

### 3.5 算法灵敏度设置（2 个）

| 页面 | 大小 | 说明 |
|------|------|------|
| `SenseSetWidget` | **78KB cpp + 72KB ui** | 核心——21 种算法的灵敏度滑动条 + 启用/禁用 + 详细设置入口 |
| `AiWipeSetWidget` | 1.7KB | AI 智能清灰参数设置 |

**SenseSetWidget 是 mainpage/ 中最核心的页面**。它用 21 个水平滑动条分别控制灰度、SVM、HSV、形状、大小、霉斑、芽头等算法的灵敏度。每个算法行有三个控件：

```
[算法名称] [========●========] [✓启用] [详细...]
                           ↑
                    灵敏度滑动条
```

点"详细"按钮 → `g_MainManager().ShowWidget()` 跳转到 `arithprofilewidget/` 中的对应算法详细设置页。

数据流：

```
用户拖滑动条 → onSenseValueChanged(index)
  → 找到对应算法的 struct 字段
  → struCnfp.struGroupIdentify[...].struGreyColor[...].nSensLow = 新值
  → 如果是复制模式 → 同步到关联视
  → myFlow.materialParamsCopySendAssemble() → 下发到硬件
```

---

### 3.6 IPC 相机类参数（2 个）

| 页面 | 大小 | 说明 |
|------|------|------|
| `ipcclassparams` | **97KB cpp + 412KB ui** | IPC AI 输出分类参数——52 个分类的完整参数配置 |
| `ipcclassadvparams` | 只有 .h 和 .ui | IPC 高级参数精简版（无 .cpp） |

**ipcclassparams 是 mainpage/ 中体量最大的页面**。IPC 相机的 AI 模型输出最多 52 个类别（如"好米"、"碎米"、"异色粒"、"霉变粒"...），每个类别可配置：

```
每个类别 (×52) 的参数:
  ✓ 启用/禁用
  ✓ 灵敏度 (0-100)
  ✓ 阈值 (0-100)
  ✓ 名称
  □ 踢出框大小
  □ 喷阀模式（重吹/轻吹）
  □ 延迟模式（关键点/中心点）
  □ 算法模式 (0-4)
  □ 高宽比阈值
  □ 位置调整 (-128~127)
  □ 反选模式
  □ AND 组合
```

52 个类别 × 10+ 个参数 = 500+ 个控件。通过分页（每页 24 个）和 QSignalMapper 统一管理。

---

### 3.7 维护/杂项（6 个）

| 页面 | 说明 |
|------|------|
| `wipeSetWidget` | 清灰参数：时长、间隔、模式、AI 清灰 |
| `SelfCheckWidget` | **开机自检页**：显示各硬件板通信状态（正常/异常） |
| `SimulateWidget` | 图像回放：加载保存的测试图片，验证分选参数 |
| `DccryptWidget` | 解密/激活页面（跳转到加密流程） |
| `ejectDelay` | **喷阀时序配置**：延迟、吹气时长、尾气、二次吹 |
| `RemoteControlWidget` | 远程控制锁屏（MQTT 远程操控时显示） |

**SelfCheckWidget 自检流程**：

```
communication() 失败后 → ShowWidget(SM_SELF_CHECK_WIDGET)
  → showCheckInfo() 读取 struGsh.struVer（版本/通信状态结构体）
  → 列表显示: 接口板 ✓ / 相机1 ✓ / 相机2 ✓ / 阀板1 ✗ / ...
  → 全部正常 → 自动进主页
  → 有异常 → 显示"进入主页"按钮 → 用户手动进入
```

---

## 四、跨页面统一模式

### 4.1 QSignalMapper 模式

多个页面有大量同类控件（SenseSetWidget 的 21 个算法、ipcclassparams 的 52 个分类），都用 `QSignalMapper` 统一管理：

```cpp
// 创建映射器
m_pSignalMapper = new QSignalMapper(this);

// 批量创建控件 + 映射
for (int i = 0; i < 21; i++) {
    QPushButton *btn = new QPushButton;
    m_pSignalMapper->setMapping(btn, i);     // btn → 索引 i
    connect(btn, SIGNAL(clicked()), m_pSignalMapper, SLOT(map()));
}

// 一个槽函数处理所有
connect(m_pSignalMapper, SIGNAL(mapped(int)), this, SLOT(onBtnClicked(int)));
// onBtnClicked(int index) 里 switch(index) 分发到对应逻辑
```

### 4.2 全局结构体读写模式

所有页面都直接读写全局变量，不通过参数传递：

```
读:  int value = struCnfp.struGroupIdentify[layer][view][group].struGreyColor[0].nSensLow;
写:  struCnfp.struGroupIdentify[layer][view][group].struGreyColor[0].nSensLow = newValue;
下发: myFlow.materialParamsCopySendAssemble();
```

### 4.3 页面导航模式

```
前进: g_MainManager().ShowWidget(SM_XXX_WIDGET)
返回: returnBack() → g_MainManager().ShowWidget(m_PreviousPageId, true)
```

---

## 五、数据流完整链路

以"用户调整灰度灵敏度"为例：

```
1. 用户进入主页 → 点"灵敏度" → ShowWidget(SM_SENSE_SET_WIDGET)
2. SenseSetWidget::showPage() → loadSenseData() → 从 struCnfp 读 21 种算法数据
3. 用户拖动灰度A的滑动条
4. onSenseValueChanged(ARITH_GREY_A)
   → struCnfp.struGroupIdentify[...].struGreyColor[0].nSensLow = 新值
   → myFlow.materialParamsCopySendAssemble()
     → GlobalFlow 把结构体打包成串口数据
     → myQueue.push(packet) → mySerialSendThread → write(串口)
     → FPGA 收到 → 更新寄存器 → 分选参数生效
5. 用户点"保存" → myFlow.saveProfile() → JsonDataConvert → JSON 文件写入 SD 卡
```

---

## 六、页面间关系图

```
                     NewMainPage (主页)
                      ├── [灵敏度] → UserWidget
                      ├── [设置]   → EngineerWidget
                      │               ├── [工厂设置] → FactorySetWidget
                      │               ├── [图像采集] → ImageCaptureWidget
                      │               ├── [相机设置] → CameraSetWidget
                      │               ├── [背景设置] → BackgroundSetWidget
                      │               └── ...
                      ├── [模式]   → ModeManageWidget
                      │               ├── [新建] → CreateNewModeWidget
                      │               ├── [切换模型] → ModelChangeWidget
                      │               └── [云端下载] → ModelSelectWidget
                      ├── [阀门]   → ejectDelay
                      ├── [清灰]   → wipeSetWidget
                      └── [关机]   → 关机流程

自检失败时:
  communication() → SelfCheckWidget → (全部正常) → 自动进主页
```

---

## 七、文件总览

| 文件 | 规模 | 继承 | 用途 |
|------|------|------|------|
| newmainpage | 8.5KB | basewidget | **新主页**（Run/Stop/导航） |
| mainpagewidget | 7.3KB | basewidget | 旧主页 |
| userwidget | 19KB | basewidget | 灵敏度调节页 |
| engineerwidget | 10KB | basewidget | 工程师设置中心 |
| modemanagewidget | 23KB | basewidget | 方案管理 |
| createnewmodewidget | 4.6KB | basewidget | 新建方案向导 |
| modelchangewidget | 17KB | basewidget | IPC 模型切换 |
| modelselectwidget | 9.5KB | basewidget | 云端模型下载 |
| modesetwidget | 1.5KB | basewidget | Tab 容器 |
| feedsetwidget | 5.8KB | basewidget | 供料总容器 |
| feedsetbychutepage | 4.8KB | basewidget | 按通道供料 |
| feedsetbygrouppage | 4.3KB | basewidget | 按组供料 |
| feedsetbyclearuppage | 1KB | basewidget | 清料平衡 |
| feederaotuajust | 13KB | basewidget | 自动校准 |
| feedbanlanceaimsingle | 2.1KB | basewidget | 单通道偏置 |
| **sensesetwidget** | **78KB** | basewidget | **21 种算法灵敏度** |
| aiwipesetwidget | 1.7KB | basewidget | AI 清灰参数 |
| **ipcclassparams** | **97KB** | basewidget | **52 分类 IPC 参数** |
| ipcclassadvparams | — | basewidget | IPC 高级参数(无实现) |
| wipesetwidget | 4.2KB | basewidget | 清灰参数 |
| selfcheckwidget | 7.8KB | basewidget | 开机自检 |
| simulatewidget | 3.1KB | basewidget | 图像回放 |
| dccryptwidget | 0.6KB | basewidget | 解密页面 |
| ejectdelay | 16KB | basewidget | 喷阀时序 |
| remoteControl | 5.5KB | basewidget | 远程控制锁屏 |
