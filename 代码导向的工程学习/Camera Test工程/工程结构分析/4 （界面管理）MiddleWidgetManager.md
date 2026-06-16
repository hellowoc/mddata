# MiddleWidgetManager 页面管理器详解

## 一、类继承和数据成员

```
MiddleWidgetManager
  继承: QObject            ← 不是 widget，是纯逻辑控制器
  职责: 页面切换调度中心

数据成员:
  m_pStackedWidget   → 指向 ZKSort 的 QStackedWidget（物理的页面容器）
  m_pMainWidget      → 指向 ZKSort（页面切换时通知它更新顶部/底部栏）
  m_WidgetMap        → QMap<pageId, index>  记录"页面ID → 在stacked中的索引"
  m_pCurWidget       → 当前显示的页面指针
  m_CurIndex         → 当前在 stacked 中的索引
```

---

## 二、控制权交接 — setStackedWidget()

```cpp
// ZKSort 构造函数中
g_MainManager().setStackedWidget(ui->ContainerStackedWidget, this);
//                                   ①                           ②
//                    QStackedWidget*                   ZKSort*
```

```cpp
void MiddleWidgetManager::setStackedWidget(QStackedWidget *stack, ZKSort *mainWidget)
{
    m_pStackedWidget = stack;      // 记住页面容器
    m_pMainWidget = mainWidget;    // 记住主窗口（用来调 PageChanged）
}
```

这一行之后，MiddleWidgetManager 拿到了 QStackedWidget 的控制权。此后所有页面创建和切换都由它管理。

---

## 三、核心机制 — 页面创建（CreatePageByID）

一个巨大的 switch-case 工厂函数，90+ 种页面：

```cpp
int MiddleWidgetManager::CreatePageByID(uint pageId)
{
    int widgetIndex = 0;
    switch (pageId) {
    case SM_USER_WIDGET:              // pageId = 1
        widgetIndex = m_pStackedWidget->addWidget(new UserWidget);
        break;                         // ↑ 创建 + 加入 QStackedWidget，返回索引
    case SM_ENGINEER_WIDGET:          // pageId = 2
        widgetIndex = m_pStackedWidget->addWidget(new EngineerWidget);
        break;
    case SM_FACTORY_SET_WIDGET:       // pageId = 4
        widgetIndex = m_pStackedWidget->addWidget(new FactorySetWidget);
        break;
    // ... 90+ 个 case ...
    }
    m_WidgetMap[pageId] = widgetIndex; // 记录 "pageId 4 → stacked索引 3"
    return widgetIndex;
}
```

**延迟创建**：不是启动时全部 new，而是第一次 ShowWidget 时才创建，不用的页面不占内存。

---

## 四、核心机制 — 页面切换[[ShowWidget详解]]

```cpp
void MiddleWidgetManager::ShowWidget(uint pageId, bool bBack)
{
    // ① 如果页面还没创建过 → 先创建
    if (!m_WidgetMap.contains(pageId)) {
        index = CreatePageByID(pageId);
    }
    index = m_WidgetMap[pageId];

    // ② 如果已经是当前页 → 不切
    if (index == m_CurIndex) return;

    // ③ 如果是首次创建 → 刷新语言
    basewidget* widget = m_pStackedWidget->widget(index);
    if (bFirstCreate) {
        widget->retranslateUiWidget();
    }

    // ④ 记录页面跳转链（用于返回按钮）
    widget->m_CurrentPageId = pageId;
    if (m_pCurWidget && !bBack) {
        widget->m_PreviousPageId = m_pCurWidget->m_CurrentPageId;
    }

    // ⑤ 隐藏旧页面
    m_pCurWidget->showPage(false);

    // ⑥ 切换到新页面
    m_CurIndex = index;
    m_pCurWidget = widget;
    m_pStackedWidget->setCurrentIndex(m_CurIndex);  // QStackedWidget 切换
    m_pCurWidget->showPage(true);

    // ⑦ 通知主窗口（更新顶部栏标题、底部栏显隐）
    m_pMainWidget->PageChanged(pageId);
}
```

---

## 五、页面跳转链 — 返回按钮怎么知道回到哪

```
用户操作: 主页 → 设置 → 工厂设置 → 相机设置

每次 ShowWidget 时:
  相机设置.m_PreviousPageId = 工厂设置的 pageId
  工厂设置.m_PreviousPageId = 设置页的 pageId
  设置页.m_PreviousPageId  = 主页的 pageId

用户按返回:
  returnBackPage() → m_pCurWidget->returnBack()
    → widget 内部: g_MainManager().ShowWidget(m_PreviousPageId, true)
    → ShowWidget 检测到 bBack=true → 不记录跳转链 → 原路返回
```

---

## 六、消息传递

```cpp
// 方式1：给当前页面发消息
void SendCurWidgetMsg(int msg1, int msg2, int msg3)
{
    m_pCurWidget->receiveMsg(msg1, msg2, msg3);
}

// 方式2：给指定页面发消息（即使它不是当前页）
void SendWidgetMsg(uint pageId, int msg1, int msg2, int msg3)
{
    basewidget *widget = m_pStackedWidget->widget(m_WidgetMap[pageId]);
    widget->receiveMsg(msg1, msg2, msg3);
}
```

---

## 七、其他方法

| 方法 | 作用 |
|------|------|
| `RefreshCurWidget()` | 刷新当前页面 |
| `GetCurWidget()` | 获取当前页面指针 |
| `GetWidgetByID(pageId)` | 按 ID 查找页面 |
| `SetLanguage()` | 通知当前页刷新语言 |
| `SetBottomEnable(bool)` | 委托 ZKSort 控制底部栏按钮 |
| `RefreshBottomStatus()` | 委托 ZKSort 刷新底部状态 |

---

## 八、完整生命周期

```
ZKSort 构造函数:
  g_MainManager().setStackedWidget(...)   ← 获得 QStackedWidget 控制权

communication() 成功后:
  startEnterMainPage()
    → g_MainManager().CreateWidgets()     ← 预创建少数页面
    → g_MainManager().ShowWidget(主页)    ← 第一次调用 ShowWidget
         → CreatePageByID(主页)            ← 延迟创建主页
         → m_WidgetMap[39] = 0            ← 记录索引
         → m_pCurWidget = 主页
         → PageChanged(39)                ← 通知 ZKSort 更新导航栏

用户点击"设置":
  → 设置按钮 click 信号
    → g_MainManager().ShowWidget(SM_FACTORY_SET_WIDGET)
         → CreatePageByID(工厂设置)        ← 第一次用才创建
         → 隐藏主页 → 显示工厂设置页
         → 工厂设置.m_PreviousPageId = 主页

用户按"返回":
  → factorySetWidget::returnBack()
    → g_MainManager().ShowWidget(m_PreviousPageId, true)
         → 隐藏工厂设置 → 显示主页
```

---

## 九、一句话

MiddleWidgetManager 是页面系统的**总调度员**：拿着 QStackedWidget，谁要看哪个页面就切过去，页面没创建就先 new 再切，切换时自动记录"从哪来的"以供返回，切完后通知 ZKSort 更新顶部栏和底部栏。90+ 个页面全部延迟创建，只在第一次访问时初始化。
