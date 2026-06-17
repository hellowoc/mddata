# MySqlite 与 SQL 数据库详解

---

## 第一部分：数据库是什么

### 一、没有数据库时怎么存数据

```
程序运行时:
  → 数据在内存中（struct、变量）
  → 关机 → 数据消失 ✗

想保留数据:
  → 自己写文件（JSON、ini、CSV...）
  → 每次读都要解析整个文件
  → 想查"上周产量大于 100 的记录" → 需要自己写遍历代码
```

### 二、数据库解决什么问题

数据库就是**专门管理数据的软件**，帮你做三件事：

| 能力 | 自己写文件 | 数据库 |
|------|----------|--------|
| **存数据** | open + write | INSERT |
| **查数据** | open + 遍历 + if | SELECT ... WHERE ... |
| **改数据** | open + 找位置 + 覆盖写 | UPDATE ... WHERE ... |
| **删数据** | 标记 + 重建文件 | DELETE ... WHERE ... |
| **不丢数据** | 自己处理断电 | 数据库自带事务保证 |
| **多线程安全** | 自己加锁 | 数据库自带 |

---

### 三、SQLite 是什么

```
数据库类型:
  MySQL     → 独立服务器进程，适合网站（百万并发）
  PostgreSQL → 独立服务器进程，适合复杂查询
  SQLite    → 无服务器，直接嵌入你的程序，数据存在一个 .db 文件里
```

**SQLite 只有一个文件。** 没有服务进程，不需要安装，不需要配置。嵌入式设备的标准选择——代码就是数据库，数据库就是文件。

本项目用的就是 SQLite，文件路径：

```cpp
#define STATISTIC_DATABASE "/sdcard/ts/statisticDataLog.db"
```

---

### 四、SQL 语言是什么

SQL = Structured Query Language，操作数据库的**命令语言**。四种核心操作：

```sql
-- 插入一行数据
INSERT INTO 表名 (列1, 列2) VALUES (值1, 值2);

-- 查询数据
SELECT 列1, 列2 FROM 表名 WHERE 时间 = '2026-05-22';

-- 更新数据
UPDATE 表名 SET 列1 = 新值 WHERE 时间 = '2026-05-22';

-- 删除数据
DELETE FROM 表名 WHERE 时间 < '2026-01-01';
```

SQL 本身不是编程语言——它就是几条固定的命令语法，所有关系数据库都认。

---

### 五、数据库的内部结构

```
数据库文件 (statisticDataLog.db)
  └── 表 (Table) — 类似 Excel 的一个 Sheet
        ├── 列 (Column) — 每一列有一个名字和类型
        │    如: CreatedTime TEXT, ThroughPut INTEGER, GreyA INTEGER
        │
        └── 行 (Row) — 每一条数据记录
              ┌─────────────┬───────────┬───────┬───────┐
              │ CreatedTime │ ThroughPut│ GreyA │ GreyB │
              ├─────────────┼───────────┼───────┼───────┤
              │ 2026-05-20  │   1523    │  45   │  12   │  ← 一行
              │ 2026-05-21  │   1680    │  51   │   8   │  ← 另一行
              │ 2026-05-22  │   1440    │  38   │  15   │
              └─────────────┴───────────┴───────┴───────┘
```

---

## 第二部分：Qt 如何操作 SQLite

Qt 提供了 `QSqlDatabase` + `QSqlQuery` 两个类来操作数据库：

### 打开数据库

```cpp
QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE");  // 指定用 SQLite 驱动
db.setDatabaseName("/sdcard/ts/statisticDataLog.db");     // 指定 .db 文件路径
db.open();    // 打开（文件不存在则自动创建）
```

### 执行 SQL 语句

```cpp
QSqlQuery query(db);                       // 创建查询对象，关联到 db
query.exec("SELECT * FROM 表名 WHERE ..."); // 执行 SQL 语句
```

### 读取查询结果

```cpp
while (query.next()) {                     // 逐行遍历结果
    int val = query.value(0).toInt();      // 取第0列的值
    int val2 = query.value(1).toInt();     // 取第1列的值
}
```

---

## 第三部分：MySqlite 类详解

### 类结构

```cpp
class MySqlite : public QObject
{
    QSqlDatabase db;           // 数据库连接对象
    QStringList viewName;      // 视的名称列表: "MainFront","MainRear","ViceFront"...

    bool openDatabase();       // 打开 .db 文件
    void closeDatabase();      // 关闭 .db 文件

    bool creatTable(name);     // 创建表结构
    bool updateTable(name);    // 更新当天数据（UPDATE 或 INSERT）
    bool clearTable(name);     // 清理过期数据
    bool deleteTable(name);    // 删除表

    bool isTableExist(table);  // 表是否存在
    bool isItemExist(table, item); // 当天记录是否已存在

    double calculateSingleColumnAverage(begin, end, col); // 计算平均值

    void updateStagedThroughout();    // 从硬件统计累计到阶段性缓冲区
    void clearStagedThroughout();     // 清空阶段性缓冲区
};
```

---

### 1. openDatabase — 打开数据库

```cpp
bool MySqlite::openDatabase()
{
    if (db.isOpen())          // 已经开了 → 不重复开
        return TRUE;

    if (db.contains("qt_sql_default_connection"))
        db = QSqlDatabase::database("qt_sql_default_connection");  // 复用已有连接
    else
        db = QSqlDatabase::addDatabase("QSQLITE");  // 新建 SQLite 连接

    db.setDatabaseName(STATISTIC_DATABASE);  // 指定文件: /sdcard/ts/statisticDataLog.db
    if (!db.open()) {        // 打开文件（不存在则创建）
        db.close();
        return FALSE;
    }
    return TRUE;
}
```

**`QSqlDatabase::addDatabase("QSQLITE")`** — Qt 的数据库工厂方法。"QSQLITE" 是驱动名称，Qt 内置支持。调用后返回一个 `QSqlDatabase` 对象，后续用它创建 `QSqlQuery` 来执行 SQL。

---

### 2. creatTable — 创建表结构

这是数据写入前的准备工作——先定义好表中每一列的名字和类型。

```cpp
bool MySqlite::creatTable(QString tableName)
{
    openDatabase();

    if (isTableExist(tableName)) {  // 表已存在 → 直接退出
        db.close();
        return TRUE;
    }

    QSqlQuery query(db);
    QString sql = "CREATE TABLE " + tableName + "("
                  "CreatedTime TimeStamp NOT NULL DEFAULT (date('now','localtime')),";

    // 从全局结构中读取 IPC 模型分类名称作为列名
    for (int i = 0; i < modelClassCount; i++) {
        classNameStr = modelClassIf.className[i];  // 如 "GreyA", "SvmB"...
        sql += QString("%1 INTEGER, ").arg(classNameStr);  // 每列: 列名 类型,
    }
    sql += ")";   // 去掉最后的逗号，加右括号

    query.exec("DROP TABLE " + tableName);  // 先删旧表
    query.exec(sql);                        // 执行 CREATE TABLE
    db.close();
}
```

生成的 SQL 示例：

```sql
CREATE TABLE LAYER1_STATISTIC_DATA (
    CreatedTime TimeStamp NOT NULL DEFAULT (date('now','localtime')),
    GreyA INTEGER,
    GreyB INTEGER,
    GreyC INTEGER,
    SvmA INTEGER,
    SvmB INTEGER,
    ThroughOut INTEGER,
    TotalEjectTimes INTEGER,
    ...
)
```

**重要设计**：列名不是固定的，而是从 `struIpcShare.struIpcModelClass[0].className[]` 动态读取。这意味着不同的机器配置会产生不同的表结构——表结构完全由当前机型的 IPC 模型分类决定。

---

### 3. updateTable — 更新当天数据（核心写入逻辑）

每天会多次调用这个函数，根据"今天是否已有记录"决定是 UPDATE 还是 INSERT。

```cpp
bool MySqlite::updateTable(QString tableName)
{
    openDatabase();

    // 检查今天是否已有记录
    QString today = QDateTime::currentDateTime().toString("yyyy-MM-dd");

    if (isItemExist(tableName, today))
    {
        // ===== 情况A: 今天已有记录 → UPDATE 更新 =====
        QSqlQuery query(db);
        QString sql = "UPDATE " + tableName + " SET ";

        // 拼接: GreyA = 45, GreyB = 12, ThroughOut = 1523, ...
        for (int i = 0; i < modelCount; i++) {
            sql += QString("%1 = %2,")
                   .arg(className)
                   .arg(curBadPoints[i] + lastBadPoints[i]);
        }
        sql += QString(" WHERE CreatedTime = '%1'").arg(today);

        query.prepare(sql);
        query.exec();    // 执行 UPDATE
    }
    else
    {
        // ===== 情况B: 今天还没有记录 → INSERT 插入新行 =====
        QSqlQuery query(db);
        QString sql = "INSERT INTO " + tableName + "(";
        QString valuePart = "VALUES (";

        for (int i = 0; i < modelCount; i++) {
            sql += QString("%1, ").arg(className);         // 列名列表
            valuePart += QString(":%1, ").arg(className);  // 值占位符
        }
        sql += ") " + valuePart + ")";

        query.prepare(sql);
        // 绑定每个占位符的实际值
        for (int i = 0; i < modelCount; i++) {
            query.bindValue(QString(":%1").arg(className),
                           curBadPoints[i] + lastBadPoints[i]);
        }
        query.exec();    // 执行 INSERT
    }

    db.close();
}
```

**两种情况的含义：**

```
今天第一次调用 → 表中没有今天的行 → INSERT（插入新行）
今天第二次及以后 → 表中已有今天的行 → UPDATE（更新那行）
```

这保证了**每天只有一行数据**，每次更新是覆盖而不是追加。

**`query.prepare(sql)` 和 `query.bindValue()`** — 预处理语句，防止 SQL 注入。先用 `:占位符` 写 SQL 模板，再用 `bindValue()` 填入实际值。

---

### 4. clearTable — 清理过期数据

```cpp
bool MySqlite::clearTable(QString tableName)
{
    openDatabase();
    QSqlQuery query(db);

    // 删除今天的记录
    query.exec("DELETE FROM " + tableName +
               " WHERE CreatedTime < datetime('now','start of day','+1 day')");

    query.exec("VACUUM");  // 回收空间（SQLite 的 DELETE 不自动释放磁盘空间）
    db.close();
}
```

**VACUUM** — SQLite 的 DELETE 只是标记删除，不释放磁盘空间。VACUUM 重建数据库文件回收空间，相当于碎片整理。

---

### 5. isTableExist / isItemExist — 存在性检查

```cpp
bool MySqlite::isTableExist(QString tableName)
{
    QSqlQuery query(db);
    // 查询 SQLite 的系统表 sqlite_master
    query.exec(QString("SELECT * FROM sqlite_master WHERE name='%1'").arg(tableName));
    return query.next();  // 有结果 → 表存在
}

bool MySqlite::isItemExist(QString table, QString item)
{
    QSqlQuery query(db);
    // 查表中是否有指定日期的行
    query.exec(QString("SELECT * FROM %1 WHERE CreatedTime = '%2'").arg(table).arg(item));
    return query.next();  // 有结果 → 该日已有记录
}
```

---

### 6. calculateSingleColumnAverage — 平均值查询

```cpp
double MySqlite::calculateSingleColumnAverage(
    QString t_begin,     // 开始日期 "2026-05-01"
    QString t_end,       // 结束日期 "2026-05-22"
    QString columnName)  // 列名 "ThroughPut"
{
    QSqlQuery query(db);

    // 生成: SELECT ThroughPut FROM LAYER1_STATISTIC_DATA
    //       WHERE CreatedTime >= '2026-05-01' AND CreatedTime <= '2026-05-22'
    QString sql = QString(
        "SELECT %1 FROM LAYER1_STATISTIC_DATA "
        "WHERE CreatedTime >= '%2' AND CreatedTime <= '%3'")
        .arg(columnName).arg(t_begin).arg(t_end);

    query.exec(sql);

    int cnt = 0;
    double evr = 0;
    while (query.next()) {
        // 递增平均算法：防止大数累加溢出
        evr += (double)((query.value(0).toInt() - evr) / ++cnt);
    }
    return evr;
}
```

**递增平均算法** `evr += (val - evr) / cnt` — 不需要先把所有值加起来再除（可能溢出），而是逐个累加时动态修正平均值。

---

### 7. updateStagedThroughout — 阶段性产量累计

```cpp
void MySqlite::updateStagedThroughout()
{
    // 把硬件上报的实时统计数据 累加到 阶段性缓冲区
    for (int m = 0; m < nGroupUnitTotal; m++) {
        // 产量 = 硬件计数值 / 系数 → 转换为千克
        tempvalue = (float)struGsh.struStatics[0][j][m].nThroughout
                  / (float)struCnfp.n_coff;

        // 累加到缓冲区
        struGsh.stagedThroughout[0][j][m] += tempvalue;

        // 累加吹气次数
        struGsh.stagedTotalArithEjectTimes[0][j][m]
            += struGsh.chanelArithEjectTimes[0][index][m];
    }
}
```

**数据流：**

```
硬件(FPGA) → 串口 → mySerialRecvThread → struGsh.struStatics (实时计数)
                                              ↓
                                   updateStagedThroughout()
                                   累加转换后 →
                                   struGsh.stagedThroughout (阶段性产量，单位：千克)
                                              ↓
                                   定时触发 → updateTable()
                                   写入 SQLite .db 文件
```

---

## 第四部分：queryStatisticThread 查询线程

### 线程结构

```cpp
class queryStatisticThread : public QThread
{
    StatisticType m_nStatisticType;  // 查产量 还是 查吹气次数
    QueryType m_nQueryType;          // 实时 / 昨天 / 上周 / 最近3月
    ViewType m_nViewType;            // 前主 / 前辅 / 后主 / 后辅

    QList<int> m_dataValueList;      // 查询结果数据
    QList<QString> m_dataStrList;    // 查询结果标签

    void run();                      // 线程主循环
    void setQueryInfo(type, queryType, viewType);
};
```

### 线程主循环

```cpp
void queryStatisticThread::run()
{
    MySqlite mysql;

    while (1) {
        if (!isQuerying && !isNeedInserting) {
            msleep(100);          // 没有任务 → 休眠 100ms
            continue;
        }

        if (isQuerying) {
            switch (m_nQueryType) {
            case LastDay:   // 查询最近 24 小时的产量或含杂
                // 按 2 小时间隔分段查询
                for (int i = -12; i < 0; i++) {
                    sql = "SELECT ThroughPut FROM LAYER1_STATISTIC_DATA "
                          "WHERE CreatedTime >= datetime('now','localtime','" + hour + " hour') "
                          "  AND CreatedTime <  datetime('now','localtime','" + hour2 + " hour')";
                    query.exec(sql);
                    while (query.next())
                        tempvalue += query.value(0).toInt();
                    m_dataValueList.append(tempvalue);  // 每 2 小时一个数据点
                }
                break;

            case LastWeek:  // 最近 7 天，每天一个数据点
                for (int i = -7; i < 0; i++) { ... }
                break;

            case Last3Month: // 最近 3 月，每月一个数据点
                for (int i = -3; i < 0; i++) { ... }
                break;

            case RealTime:  // 实时数据，直接读内存中的 stastics 结构体
                // 不查数据库，直接从 struGsh.struStatics 取
                // 因为实时数据还没写入数据库
                break;
            }
        }

        if (isNeedInserting) {
            mysql.updateTable();           // 写入数据库
            mysql.clearStagedThroughout(); // 清空缓冲区
            isNeedInserting = false;
        }

        isQuerying = false;
        isNeedRepainting = true;
        emit statisticQueryFinsishedSig(index);  // 通知 UI 更新图表
    }
}
```

---

## 第五部分：完整数据流

### 写入数据流

```
FPGA 硬件
    ↓ 串口上报
mySerialRecvThread (接收线程)
    ↓ 解析数据包
struGsh.struStatistics[层][视][通道].nThroughout  (原始计数值)
struGsh.chanelArithEjectTimes[][]                 (吹气次数)
    ↓ 定时调用 (如每 5 秒)
MySqlite::updateStagedThroughout()
    → 转换单位 (计数值 → 千克)
    → 累加到 struGsh.stagedThroughout[][]  (阶段性累计值)
    → 累加到 struGsh.stagedTotalArithEjectTimes[][]
    ↓ 定时调用 (如每 30 分钟)
queryStatisticThread.isNeedInserting = true
    → 线程唤醒
    → MySqlite::updateTable()
        → 今天第一次 → INSERT 新行
        → 今天第二次 → UPDATE 已有行
    → MySqlite::clearStagedThroughout()
        → 清零阶段性缓冲区
    ↓
/sdcard/ts/statisticDataLog.db
```

### 读取数据流

```
UI 用户点击"产量统计"按钮
    ↓
queryStatisticThread::setQueryInfo(产量, 最近7天, 前主)
queryStatisticThread::isQuerying = true
    ↓
线程主循环唤醒
    ↓
switch (m_nQueryType):
  case LastWeek:
    → mysql.openDatabase()
    → SELECT ThroughPut FROM LAYER1_STATISTIC_DATA
      WHERE CreatedTime >= ... AND CreatedTime < ...
    → 逐日读取 → m_dataValueList = [1200, 1350, 1100, ...]
    → mysql.closeDatabase()
    ↓
emit statisticQueryFinsishedSig(0)
    ↓
UI 槽函数:
    → 读取 m_dataValueList
    → barchart / plotcurve 控件绘制柱状图或折线图
```

---

## 第六部分：数据存储位置和生命周期

```
存储位置: /sdcard/ts/statisticDataLog.db

表结构:
  LAYER1_STATISTIC_DATA
    ├── CreatedTime (日期, 主键)
    ├── ThroughPut (总产量)
    ├── TotalEjectTimes (总吹气次数)
    ├── GreyA, GreyB, GreyC, GreyD (灰度算法含杂)
    ├── SvmA, SvmB, SvmC (SVM 算法含杂)
    ├── DiscolorA, DiscolorB (异色算法含杂)
    ├── ... (其余算法含杂列)
    └── ... (每个 IPC 分类一个 INTEGER 列)

每天一行，当天多次更新只改那行
清理策略: clearTable() 删除旧数据 + VACUUM 回收空间
```

---

## 第七部分：关键设计要点

| 要点 | 说明 |
|------|------|
| **每天一行** | 同一天的产量数据 UPDATE 而不是 INSERT 新行，避免一行一条小数据 |
| **动态列名** | 表结构由 IPC 模型分类动态决定，不同机型不同表结构 |
| **独立查询线程** | 数据库查询放在 `queryStatisticThread` 中，不阻塞 UI |
| **阶段性缓冲** | 原始数据先在 `stagedThroughout[]` 中累积，定时批量写入，减少数据库操作频率 |
| **实时数据不查库** | `RealTime` 查询直接读内存中的 `struGsh.struStatics`，不走 SQL |
| **VACUUM** | 定期回收被 DELETE 占用的磁盘空间 |
| **预处理语句** | `query.prepare()` + `bindValue()` 防止 SQL 注入和格式错误 |
