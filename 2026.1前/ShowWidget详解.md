# `ShowWidget()` 逐层拆解

## 函数全貌

```cpp
void MiddleWidgetManager::ShowWidget(uint pageId, bool bBack)
{
    int index;
    bool bFirstCreate = false;

    // ===== 第一阶段：确保页面存在 =====
    if (!m_WidgetMap.contains(pageId)) {
        index = CreatePageByID(pageId);
        bFirstCreate = true;
        if (index == -1) {
            logger()->warn("create page fail,page name is not exist");
            return;                                    // ① 出口1：创建失败
        }
    }
    index = m_WidgetMap[pageId];

    // ===== 第二阶段：检查是否需要切换 =====
    if (index == m_CurIndex)
        return;                                        // ② 出口2：已是当前页

    basewidget* widget = (basewidget *)m_pStackedWidget->widget(index);
    if (NULL == widget)
        return;                                        // ③ 出口3：取不到widget

    // ===== 第三阶段：初始化新页面 =====
    if (bFirstCreate) {
        widget->retranslateUiWidget();                 // 首次创建→刷新语言
    }
    widget->m_CurrentPageId = pageId;

    // ===== 第四阶段：隐藏旧页面 + 记录跳转链 =====
    if (m_pCurWidget != NULL) {
        if (false == bBack) {
            widget->m_PreviousPageId = m_pCurWidget->m_CurrentPageId;
        }
        m_pCurWidget->showPage(false);                 // 隐藏旧页
    }

    // ===== 第五阶段：切换到新页面 =====
    m_CurIndex = index;
    m_pCurWidget = widget;
    m_pStackedWidget->setCurrentIndex(m_CurIndex);     // QStackedWidget 切换
    m_pCurWidget->showPage(true);                      // 显示新页

    // ===== 第六阶段：通知全局 =====
    struGsh.nPageSm = m_pCurWidget->m_CurrentPageId;
    m_pMainWidget->PageChanged(pageId);                // 更新顶部/底部栏
}
```

---

## 何时进入

整个程序任何地方需要切换页面时，都调用这个函数：

```cpp
g_MainManager().ShowWidget(SM_MAIN_PAGE_WIDGET_NEW);  // 进主页
g_MainManager().ShowWidget(SM_FACTORY_SET_WIDGET);    // 进工厂设置
g_MainManager().ShowWidget(m_PreviousPageId, true);   // 返回上一页
```

调用来源遍布 90+ 个页面 widget 的按钮 click 槽函数中。

---

## 第一阶段：确保页面存在

```cpp
if (!m_WidgetMap.contains(pageId))
```

`m_WidgetMap` 是 `QMap<int, int>` —— key 是页面 ID，value 是在 QStackedWidget 的索引。`QMap::contains()` 内部走红黑树查找。

```
找到 → 页面已创建 → 跳过
没找到 → 页面首次被访问 → 需要创建
```

```cpp
index = CreatePageByID(pageId);
```

内部 90+ case 的 switch：

```cpp
case SM_CAMERA_SET_WIDGET:
    index = m_pStackedWidget->addWidget(new CameraSetWidget);
    break;
```

`QStackedWidget::addWidget()` 内部：

```
addWidget(newWidget)
    ↓
QStackedLayout::addWidget(newWidget)
    ↓
QList<QWidget*> list; list.append(newWidget);
    ↓
索引0 = 第1个加入的页面
索引1 = 第2个加入的页面
索引2 = 第3个加入的页面
...
```

新 widget 以 QStackedWidget 为 parent，ZKSort 销毁时自动回收。

---

## 第二阶段：检查是否需要切换

```cpp
index = m_WidgetMap[pageId];   // 从 QMap 取索引

if (index == m_CurIndex)
    return;                     // 已经是当前页，不操作
```

---

## 第三阶段：初始化新页面

```cpp
basewidget* widget = (basewidget *)m_pStackedWidget->widget(index);
```

`QStackedWidget::widget(index)` 按索引取 widget 指针：

```
widget(0) → NewMainPage
widget(1) → FactorySetWidget
widget(2) → CameraSetWidget
...
```

```cpp
if (bFirstCreate) {
    widget->retranslateUiWidget();    // 首次创建时刷新语言
}
```

为什么首次创建要调？页面构造函数里 `ui->setupUi()` 设的文字是从 `.ui` 文件读的（Qt Designer 中的默认值，通常是中文）。`retranslateUiWidget()` 用当前选择的语言（可能是英文、俄文...）重新设所有控件文字。

```cpp
widget->m_CurrentPageId = pageId;   // 记录"我是谁"
```

---

## 第四阶段：隐藏旧页面 + 记录跳转链

```cpp
if (m_pCurWidget != NULL)   // 首次 ShowWidget 时 m_pCurWidget 是 NULL，跳过
```

```cpp
if (false == bBack) {
    widget->m_PreviousPageId = m_pCurWidget->m_CurrentPageId;
}
```

**bBack 参数的关键作用：**

```
正常前进:
  主页 → 设置 → 相机设置
  bBack=false，每次都记录:
    相机设置.PreviousPageId = 设置
    设置.PreviousPageId     = 主页

返回:
  相机设置 → ShowWidget(设置, bBack=true)
  bBack=true → 不覆盖 PreviousPageId
  设置.PreviousPageId 仍是 主页 ✓

如果返回时也记录了:
  设置.PreviousPageId 被覆盖为 相机设置
  再按返回 → 相机设置 → 设置 → 相机设置...死循环
```

```cpp
m_pCurWidget->showPage(false);   // 隐藏旧页面
```

`showPage(false)` 内部 `this->hide()`——widget 从屏幕消失但不销毁。

---

## 第五阶段：切换到新页面

```cpp
m_CurIndex = index;
m_pCurWidget = widget;
```

先更新管理器的内部状态。

```cpp
m_pStackedWidget->setCurrentIndex(m_CurIndex);
```

`QStackedWidget::setCurrentIndex()` 内部：

```
setCurrentIndex(newIndex)
    ↓
QStackedLayout::setCurrentIndex(newIndex)
    ├── oldWidget->hide()
    ├── newWidget->show()
    └── newWidget 获得焦点
```

这是在代码中同步完成的，不经过事件循环。

```cpp
m_pCurWidget->showPage(true);   // 通知新页面"你现在被显示了"
```

---

## 第六阶段：通知全局

```cpp
struGsh.nPageSm = m_pCurWidget->m_CurrentPageId;
// 全局共享状态中记录当前页面ID
```

```cpp
m_pMainWidget->PageChanged(pageId);  // ZKSort::PageChanged()
```

`ZKSort::PageChanged()` 内部：

```cpp
void ZKSort::PageChanged(const uint pageId)
{
    ui->topWidget->PageChanged(pageId);      // 顶部栏更新标题
    ui->bottomWidget->PageChanged(pageId);   // 底部栏更新按钮

    if (m_hideBottomPageId.contains(pageId))
        ui->bottomWidget->hide();            // 某些页面隐藏底部栏
    else
        ui->bottomWidget->show();

    if (m_hideTopPageId.contains(pageId))
        ui->topWidget->hide();               // 某些页面隐藏顶部栏
    else
        ui->topWidget->show();
}
```

---

## 三个出口

| 出口 | 条件 | 行为 |
|------|------|------|
| ① | 创建失败（pageId 无对应 case）| 打 warn 日志，不切换 |
| ② | 要显示的页面就是当前页 | 不操作，直接返回 |
| ③ | QStackedWidget 取到的指针为 NULL | 异常情况，返回 |

---

## 完整调用时序

```
g_MainManager().ShowWidget(SM_CAMERA_SET_WIDGET);
                        ↑ pageId=13   ↑ bBack=false

ShowWidget 内部:
  m_WidgetMap.contains(13)?
    首次: 否 → CreatePageByID(13)
              → switch(13): case SM_CAMERA_SET_WIDGET:
                  m_pStackedWidget->addWidget(new CameraSetWidget) → 返回索引 3
              → m_WidgetMap[13] = 3, bFirstCreate = true
    非首次: 直接从 m_WidgetMap[13] 取 3

  index=3, m_CurIndex=0(主页) → 不同 → 继续

  widget = m_pStackedWidget->widget(3) → CameraSetWidget*

  bFirstCreate=true → widget->retranslateUiWidget()
    → CameraSetWidget 用 g_myLan() 刷新所有文字

  widget->m_CurrentPageId = 13

  bBack=false + m_pCurWidget != NULL
    → widget->m_PreviousPageId = 主页的ID

  m_pCurWidget->showPage(false) → 主页隐藏

  m_CurIndex=3, m_pCurWidget=CameraSetWidget
  m_pStackedWidget->setCurrentIndex(3) → 主页.hide(), 相机设置.show()
  m_pCurWidget->showPage(true) → 相机设置显示

  struGsh.nPageSm = 13
  m_pMainWidget->PageChanged(13)
    → ui->topWidget->PageChanged(13)
    → ui->bottomWidget->PageChanged(13)
    → 相机设置不在隐藏列表 → bottomWidget 显示
```

---

## 关键设计要点

| 要点 | 说明 |
|------|------|
| **延迟创建** | 页面只在首次访问时 new |
| **bBack 参数** | 返回时跳过跳转链记录，防止循环 |
| **首次 retranslate** | 新页面创建后立即刷新语言 |
| **分离关注点** | ShowWidget 只管切页面，顶部/底部栏更新交给 ZKSort::PageChanged |
| **showPage(bool)** | 通知页面自己被显示/隐藏，页面内部做相应初始化/清理 |
