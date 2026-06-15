# arithprofilewidget/ 算法参数详解

---

## 一、目录定位

`arithprofilewidget/` 是**每种分选算法的详细参数设置页**。当用户在 `SenseSetWidget`（mainpage/ 中 78KB 那个页面）点某个算法的"详细"按钮时，就会跳转到这里对应的页面。

22 个页面，**全部继承 basewidget，全部遵循同一套模板**。

---

## 二、统一模板（每个页面都是这个结构）

```
class XxxSenseSetWidget : public basewidget {
    void showPage(bool)       → 从 struCnfp.struGroupIdentify[...] 读参数 → 填到 UI
    void receiveMsg(msg1,...) → 参数被修改 → 写回结构体 → myFlow.xxx() 下发硬件
    void retranslateUiWidget() → 语言切换时刷新所有文字
};
```

这些页面**不直接 new 出来**——它们由 `MiddleWidgetManager::CreatePageByID()` 在首次访问时创建（延迟加载）。

---

## 三、页面按算法的分类

### A. 灰度/颜色类（最核心，4 个）

| 页面 | 规模 | 对应算法 |
|------|------|---------|
| `graysensesetwidget` | 30KB | 灰度/色差（GREY_A/B/C/D, CROSS） |
| `discolorsensesetwidget` | 32KB | 异色粒（DISCOLOR_A/B） |
| `svmsensesetwidget` | 26KB | SVM 支持向量机（SVM_A/B/C/D） |
| `shapesvmsensesetwidget` | 13KB | 形状 SVM（SHAPE_SVM_A/B/C） |

这是四种最核心的分选算法，对应的页面也最复杂（30KB+）。每个页面都管理多个子类型的参数。

### B. HSV/颜色属性类（5 个）

| 页面 | 规模 | 对应算法 |
|------|------|---------|
| `hsvsensesetwidget` | 23KB | HSV 颜色空间 |
| `colorsatsensesetwidget` | 19KB | 颜色饱和度 |
| `colorhuesensesetwidget` | 19KB | 色相 |
| `colorscalesensesetwidget` | 10KB | 颜色比例 |
| `colorvaluesensesetwidget` | 10KB | 颜色明度 |

### C. 形状/尺寸类（4 个）

| 页面 | 规模 | 对应算法 |
|------|------|---------|
| `bigsmallsensesetwidget` | 12KB | 大小（BIG_SMALL_A/B） |
| `longshortsensesetwidget` | 12KB | 长短（LONG_SHORT_A/B） |
| `circlelongsensesetwidget` | 11KB | 圆长（CIRCLE_LONG_A/B） |
| `roundsensesetwidget` | 11KB | 圆度（ROUND_A/B） |

### D. 缺陷检测类（7 个）

| 页面 | 规模 | 对应算法 |
|------|------|---------|
| `mildewsensessetwidget` | 13KB | 霉斑 |
| `splmildewsensesetwidget` | 9KB | 特殊霉斑 |
| `outlinesensesetwidget` | 10KB | 轮廓 |
| `linesensesetwidget` | 12KB | 线状病斑 |
| `budsensesetwidget` | 12KB | 芽头 |
| `inhibitsensesetwidget` | 0.5KB | 抑制算法（几乎空壳） |
| `straightsensesetwidget` | 0.5KB | 直选算法（几乎空壳） |

### E. 特殊物料类（4 个）

| 页面 | 规模 | 对应算法 |
|------|------|---------|
| `whitepeanutsensesetwidget` | 8.8KB | 白花生 |
| `pepperhandlesensewidget` | 9.3KB | 辣椒柄 |
| `suppresslargesensesetwidget` | 13KB | 抑制大物料（SUPPRESS_LARGE_A/B/C） |
| `wheatsproutsensesetwidget` | 11KB | 小麦芽头 |

### F. 茶叶专用

| 页面 | 规模 | 对应算法 |
|------|------|---------|
| `teacolorsensesetwidget` | 6.3KB | 茶色（TEA_COLOR_0~4） |

---

## 四、典型页面的内部结构

以 GraySenseSetWidget（30KB，最复杂的灰度算法）为例：

```
GraySenseSetWidget 内部包含:

  模式切换: QButtonGroup 选低灵敏度/高灵敏度
  通道灵敏度: 每个通道一个 myLineEdit（0-100）
  切边设置: 边缘切除像素数
  传染设置: 传染算法开关
  纯度百分比: 纯度阈值
  差值颜色: RGB 差值上下限
  线状病斑: 线状检测开关
  灵敏度校正: 自动校正开关
  扩展参数: 延迟/灰度类型下拉框

数据流:
  showPage() → 遍历所有通道 → 从 struCnfp.struGroupIdentify[...].struGreyColor[...]
    读取: nSensLow, nSensUp, nRow, nPercent, nDiffColor, nArithEdge...
    → 填入 UI 控件
  receiveMsg(MSG_DATA_CHANGED) → 用户改了某个值
    → 写回 struCnfp.struGroupIdentify[...].struGreyColor[...]
    → myFlow.materialIdentifyParamsSend(...) → 下发给 FPGA
```

---

## 五、两个极简页面

### InhibitSenseSetWidget（510 字节 cpp）

```
几乎空的页面——只有 .ui 文件定义的几个控件
代码只实现了 showPage/receiveMsg/retranslateUiWidget 三个空壳
实际功能可能由 .ui 文件的信号槽直接处理
```

### StraightSenseSetWidget（520 字节 cpp）

```
同上——极简空壳，功能在 .ui 文件中
```

这两个算法可能尚未完全实现，或者参数极少不需要额外的 C++ 逻辑。

---

## 六、完整调用链路

```
用户在 SenseSetWidget（mainpage）中点"灰度A"的[详细...]按钮
    ↓
g_MainManager().ShowWidget(SM_GRAYSENSE_SET_WIDGET)
    ↓
MiddleWidgetManager::CreatePageByID() → new GraySenseSetWidget
    ↓
GraySenseSetWidget::showPage()
    → 从 struCnfp.struGroupIdentify[nLayer][nView][nGroup].struGreyColor[0]
      读取所有参数 → 填入 UI 控件
    ↓
用户修改参数（如灵敏度从 50 调到 60）
    ↓
myLineEdit 发出信号 → receiveMsg(MSG_DATA_CHANGED, index)
    → struCnfp.struGroupIdentify[...].struGreyColor[0].nSensLow = 60
    → myFlow.materialIdentifyParamsSend(ARITH_GREY_A, ...)
      → GlobalFlow 组包 → myQueue.push → 串口发送 → FPGA 更新寄存器
    ↓
用户点"返回"
    → returnBack() → 回到 SenseSetWidget
```

---

## 七、文件总览

| 文件 | cpp 大小 | 对应算法常量 |
|------|---------|------------|
| discolorsensesetwidget | 32KB | ARITH_DISCOLOR_A/B |
| graysensesetwidget | 30KB | ARITH_GREY_A/B/C/D, ARITH_CROSS |
| svmsensesetwidget | 26KB | ARITH_SVM_A/B/C/D |
| hsvsensesetwidget | 23KB | ARITH_HSV |
| colorhuesensesetwidget | 19KB | ARITH_HUE |
| colorsatsensesetwidget | 19KB | ARITH_SAT |
| shapesvmsensesetwidget | 13KB | ARITH_SHAPE_SVM_A/B/C |
| suppresslargesensesetwidget | 13KB | ARITH_SUPPRESS_LARGE |
| mildewsensessetwidget | 13KB | ARITH_MILDEW |
| bigsmallsensesetwidget | 12KB | ARITH_BIG_SMALL |
| longshortsensesetwidget | 12KB | ARITH_LONG_SHORT |
| linesensesetwidget | 12KB | ARITH_LINE |
| budsensesetwidget | 12KB | ARITH_BUD |
| wheatsproutsensesetwidget | 11KB | ARITH_WHEAT_SPROUT |
| circlelongsensesetwidget | 11KB | ARITH_CIRCLE_LONG |
| roundsensesetwidget | 11KB | ARITH_ROUND |
| colorvaluesensesetwidget | 10KB | ARITH_VALUE |
| colorscalesensesetwidget | 10KB | ARITH_SCALE |
| outlinesensesetwidget | 10KB | ARITH_OUTLINE |
| pepperhandlesensewidget | 9.3KB | ARITH_PEPPER_HANDLE |
| splmildewsensesetwidget | 9KB | ARITH_SPL_MILDEW |
| whitepeanutsensesetwidget | 8.8KB | ARITH_W_PEANUT |
| teacolorsensesetwidget | 6.3KB | TEA_ARITH_COLOR |
| inhibitsensesetwidget | 0.5KB | (抑制) |
| straightsensesetwidget | 0.5KB | (直选) |

---

## 八、一句话总结

`arithprofilewidget/` 就是 23 个长得几乎一样的页面——每个对应一种分选算法的详细参数。全部继承 basewidget，全部通过 showPage 读 `struCnfp` 结构体 → 用户修改 → receiveMsg 写回 → myFlow 下发到 FPGA。最复杂的四个（灰度/SVM/异色/HSV）30KB 左右，最简单的两个只有 500 字节。
