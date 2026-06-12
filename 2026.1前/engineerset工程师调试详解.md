# engineerset/ 工程师调试详解

---

## 一、目录定位

`engineerset/` 是**工程师调试和开发工具**页面集合。与 `factoryset/`（配置机器参数）和 `mainpage/`（日常操作）不同，这里面向的是**调试、训练、分析**场景。核心功能是图像采集、SVM 训练和 AI 模型管理。

---

## 二、核心三大件

### 2.1 ImageWidget（87KB）— 图像显示和处理引擎

```
basewidget → ImageWidget
```

**这是整个项目最核心的图像处理 widget。** 它不是独立页面，而是被嵌入到多个页面中（ImageCaptureWidget、ImageAnalysisWidget 等）作为"图像显示区域"。

**三种工作模式**（`m_widgetType`）：

| 模式 | 用途 |
|------|------|
| `captureType` | 采集模式——显示刚采集到的原始图像 |
| `analyzeType` | 分析模式——可缩放/平移/选点/标记好料坏料 |
| `simulateType` | 模拟模式——运行 SVM 仿真，用蓝色标记坏料像素 |

**9 种颜色通道显示**（`m_colorType`）：

```
origin | R | G | B | N(红外) | N+R+G | N+R+B | N+G+B
可以单独看某个颜色通道或组合显示
```

**核心处理流程**：

```
加载图像 → setImage()
  ↓
像素分类: myFlow.imagePixelIsBackground() → bgPoint[]
  (每个像素标记为: 背景 / 物料 / 坏料)
  ↓
仿真: displaySimulate()
  → 连接组件分析(ConnectedComponentLabeling)
  → 遍历每种算法 → colorSimulate() (SVM 二次公式)
  → 标记坏料像素 → chgBuf[] 涂蓝色
  ↓
显示: showGrayColor() → updateImageSize()
  → 应用缩放/平移 → 渲染到 QLabel
```

**鼠标交互**：

```
平移模式: 拖拽移动图像
分析模式: 框选区域 → 右键标记好料(绿) / 坏料(红)
         → on_saveSampleBtn_clicked() → 保存到 struGsh.nGoodSamp/nBadSamp
```

**连接组件分析**（ConnectedComponentLabeling）：把相邻的前景像素聚合成一个"物料"，在仿真时以物料为单位判断——如果物料中坏像素超过阈值，整个物料被标记为坏料。

---

### 2.2 SvmImageWidget（51KB）— SVM 训练工具

```
basewidget → SvmImageWidget
```

**完整的 SVM 模型训练流程页面**：

```
1. 加载图像 / 选样本页面

2. 标记样本:
   [好料按钮] [坏料按钮] → 点击图像上的像素
   传染标记: 通过 RGB 距离自动扩展标记区域
   样本保存在 g_Runtime().m_sampleMap

3. AI 参数设置 (AiSetDialog):
   RGB 阈值、比例/距离阈值、SVM 类型选择
   边缘切割、冗余移除等

4. 训练 (on_svmLearnBtn_clicked):
   启动 SvmLearnThread → 调用外部 train.exe
   → 生成 SVM 模型系数

5. 模拟验证:
   colorSimulate() → 用训练好的系数对图像仿真
   → 蓝色标记坏料 → 肉眼验证效果
```

**三个工作线程**：

| 线程 | 作用 |
|------|------|
| `SvmLearnThread` | 后台 SVM 训练（调用外部可执行文件 `train.exe`） |
| `SvmSimulateThread` | 后台仿真运行（不阻塞 UI） |
| `SvmMarkThread` | 后台传染标记（不阻塞 UI） |

---

### 2.3 ImageManageWidget（32KB）— 图像文件管理 + SVM 学习

```
basewidget → ImageManageWidget
  + SvmThread (QThread 子类)
  + ExportThread (QThread 子类)
```

**图像浏览和管理**：

```
每页 12 张缩略图
[缩略图] [属性按钮: 空→坏→好] [复选框]
 ↑ 点击跳转到分析页面  ↑ 标记训练标签  ↑ 用于批量操作

操作: [删除] [导出到U盘] [SVM学习]
```

**自动 SVM 学习**（`SvmThread::run()`）：

```
1. 遍历所有标记为"好"的图像 → 采样像素
2. 遍历所有标记为"坏"的图像 → 采样像素
3. 去冗余 (SampleObj 比较)
4. 调用 train() → 生成模型
```

---

## 三、图像采集（3 个页面）

| 文件 | 大小 | 用途 |
|------|------|------|
| `imagecapturewidget` | 15KB | 图像采集控制——单张/连续/连续同步三种模式 |
| `imagefastcapturewidget` | 11KB | 快速采集——单视图和前后视图并行采集 |
| `imageanalysiswidget` | 1.2KB | 图像分析容器——嵌入 ImageWidget（analyzeType） |

**采集数据流**：

```
相机 → UDP/TCP → myNetWork->udpthread → 信号 → updateImage()
  → ImageWidget::setImage() → 处理 → 显示
```

---

## 四、图像分析图表（4 个页面）

都是薄容器——内部嵌入 6 个子图表 widget：

| 页面 | 显示内容 |
|------|---------|
| `imagedatachartwidget` | HSV 直方图（H×2, S×2, V×2） |
| `imagedatachartrgbwidget` | RGB 直方图（R×2, G×2, B×2） |
| `imagedatachartdiscolorwidget` | 变色比例直方图（RG×2, RB×2, GB×2） |
| `imageinforgbwidget` | 物料 RGB 信息图（R, G, B） |

---

## 五、AI 模型升级

### upgradeModelWidget（40KB）

```
basewidget → upgradeModelWidget
```

**三种升级模式**：

| 模式 | 用途 | 通信方式 |
|------|------|---------|
| RK 模型升级 | 升级 IPC 相机上的 AI 模型数据文件 | UDP 分包（1023 字节/包） |
| X 模型升级 | 升级 X 系统相机板的 AI 模型 | UDP 分包 |
| X 系统升级 | 升级完整 X 系统固件 | UDP + 切换工厂模式 |

**升级流程**：
```
选文件 → 命令头包(CMD_UDP_IPC_MODEL_ADD_HEAD) → 数据包(CMD_UDP_IPC_MODEL_ADD_DATA)
→ 逐包确认 → 完成
```

---

## 六、背景/灯光设置（3 个页面）

| 页面 | 大小 | 用途 |
|------|------|------|
| `backgroundsetwidget` | 12KB | 相机背景颜色配置——颜色类型、上下限、比例、HSV 值、时分模式 |
| `lightsetwidget` | 8.3KB | 物料灯/背景灯控制——最多 6 路物料灯 + 4 路背景灯 |
| `doublelightsetwidget` | 7.5KB | 双灯板配置——背靠背或双板模式，最多 6 块灯板 |

**背景颜色类型**：红、绿、蓝、黑白、灰、黄、近红外

---

## 七、剔除/识别设置（3 个页面）

### 7.1 IdentifyWidget（25KB）— 算法使能网格

```
动态创建 MyArithUi 控件的网格（3 列布局）
每种可见算法一行: [✓ 灰度A] [✓ SVM-B] [✓ 大小A] ...
```

### 7.2 EjectWidget（17KB）— 喷阀参数配置

```
边缘切割 | 传染尺寸 | 闭运算 | 正反选模式 | 坏点计数 | 中心化尺寸 | IPC 延迟行数
```

### 7.3 IpcEjectWidget（5.9KB）— IPC 剔除统计

显示 IPC 相机的直方图数据，左右像素边界的坏料百分比。

---

## 八、OneKeyDialog（8.4KB）— 一键报修

```
QDialog → OneKeyDialog
```

收集客户信息（姓名/手机/公司/地址）+ 机器信息 → 通过 MQTT 上传维修请求。

---

## 九、页面间导航关系

```
ImageCaptureWidget (采集)
    ├── [分析] → ImageManageWidget (图像管理)
    │               ├── 点击缩略图 → ImageAnalysisWidget (分析)
    │               │                   └── 嵌入 ImageWidget (analyzeType)
    │               ├── [数据图表] → ImageDataChartWidget
    │               ├── [RGB图表] → ImageDataChartRgbWidget
    │               ├── [变色图表] → ImageDataChartDiscolorWidget
    │               └── [RGB信息] → ImageInfoRgbWidget
    │
    └── [快速采集] → ImageFastCaptureWidget

SvmImageWidget (SVM训练)
    ├── [AI设置] → AiSetDialog
    └── [导入样本] → MySvmSampleClass (在 mysortwidget/)
```

---

## 十、文件总览

| 文件 | 规模 | 用途 |
|------|------|------|
| **imagewidget** | **87KB** | 核心图像显示/处理引擎（嵌入所有图像页面） |
| **svmimagewidget** | **51KB** | SVM 训练工具（标记、训练、模拟） |
| **upgrademodelwidget** | **40KB** | AI 模型/固件升级（UDP 分包传输） |
| **imagemanagewidget** | **32KB** | 图像浏览/标记/SVM 自动学习 |
| **identifywidget** | **25KB** | 算法使能网格配置 |
| **ejectwidget** | **17KB** | 喷阀参数（切割/传染/正反选/延迟） |
| imagecapturewidget | 15KB | 图像采集控制 |
| backgroundsetwidget | 12KB | 背景颜色设置 |
| imagefastcapturewidget | 11KB | 快速采集 |
| onekeyfixdialog | 8.4KB | 一键报修 |
| lightsetwidget | 8.3KB | 灯光控制 |
| doublelightsetwidget | 7.5KB | 双灯板配置 |
| aisetdialog | 6.6KB | AI/SVM 高级参数 |
| ipcejectwidget | 5.9KB | IPC 剔除统计 |
| imageanalysiswidget | 1.2KB | 图像分析容器 |
| imagedatachart* | 各 1KB | HSV/RGB/变色直方图容器 |
