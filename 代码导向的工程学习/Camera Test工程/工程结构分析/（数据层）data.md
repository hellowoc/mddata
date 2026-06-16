# data/ 数据层详解

## 一、三层架构

```
业务层 (GlobalFlow, UI, ...)
    │
    ├── "保存配置" → JsonDataConvert::CnfGlobalToJson()  → JSON 文件
    ├── "读取配置" → JsonDataConvert::JsonToCnfGlobal()  ← JSON 文件
    └── "统计查询" → MySqlite::select(...)  → SQLite 数据库
    │
    ▼
┌─────────────────────────────────────────────────┐
│                  data/ 层                        │
│                                                  │
│  CMyJson (myjson.cpp 11KB)                       │
│    底层工具: Load/Save JSON 文件, 按 key 读写      │
│                                                  │
│  JsonDataConvert (jsondataconvert.cpp 193KB)     │
│    结构体映射: 40+ 种 struct ↔ JSON 互转方法       │
│                                                  │
│  MySqlite (mysqlite.cpp 45KB)                    │
│    SQLite 统计: 产量/吹气次数 增删查改            │
└─────────────────────────────────────────────────┘
```

---

## 二、CMyJson — JSON 底层工具[[json库使用]]

不继承 QObject，纯 C++ 类，所有方法都是 `static`。封装了 `jsoncpp` 库的操作：

```cpp
class CMyJson {
    // 文件级操作
    static bool LoadJsonFile(filename, Json::Value& root);
    static bool SaveJsonFile(filename, Json::Value& root);

    // 基本类型读写
    static JSON_RETURN_VAL ReadIntKey(root, "key", outInt);
    static JSON_RETURN_VAL ReadFloatKey(root, "key", outFloat);
    static JSON_RETURN_VAL ReadStringKey(root, "key", outStr);
    static JSON_RETURN_VAL WriteIntKey("key", value, root);
    static JSON_RETURN_VAL WriteFloatKey("key", value, root);
    static JSON_RETURN_VAL WriteStringKey("key", value, root);

    // 多维数组读写
    static JSON_RETURN_VAL ReadIntArrayKey(root, "key", dims, outBuf, outArray);
    static JSON_RETURN_VAL ReadFloatArrayKey(root, "key", dims, outBuf, outArray);
};
```

### SaveJsonFile 实现

```cpp
bool CMyJson::SaveJsonFile(const char* filename, const Json::Value& root)
{
    Json::FastWriter writer;           // jsoncpp 写入器
    std::string strOut = writer.write(root);  // Json::Value → JSON 文本
    ofstream ofs(filename, std::ios::out | std::ios::binary);
    ofs << strOut;                     // 写入磁盘
    ofs.close();
    return true;
}
```

### LoadJsonFile 实现

```cpp
bool CMyJson::LoadJsonFile(const char* filename, Json::Value& root)
{
    Json::Reader reader;               // jsoncpp 解析器
    ifstream ifs(filename, std::ios::in | std::ios::binary);
    root.clear();
    return reader.parse(ifs, root);    // 文件 → 解析 JSON → 填充 root
}
```

---

## 三、JsonDataConvert — 结构体映射器[[JsonDataConvert详解]]

**全是静态方法，不保存任何状态。** 作用是 C++ struct ↔ JSON 文件：

```
struCnfGlobal          ←→  /sdcard/cnf/cnf.global.json
struCnfProfile          ←→  /sdcard/cnf/cnf.profile.json
stu_color_arith[]       ←→  JSON 中的灰度算法段落
stu_svm[]               ←→  JSON 中的 SVM 算法段落
stu_hsv[]               ←→  JSON 中的 HSV 算法段落
... (40+ 种结构体)
```

### 映射方法模式

```cpp
// JSON → 结构体
bool JsonDataConvert::JsonToCnfGlobal(const Json::Value &jsonVal, struCnfGlobal &CnfGlobal)
{
    CMyJson::ReadIntKey(jsonVal, "nLang",       CnfGlobal.nLang);
    CMyJson::ReadIntKey(jsonVal, "nProfile",    CnfGlobal.nProfile);
    CMyJson::ReadIntKey(jsonVal, "nLayerTotal", CnfGlobal.nLayerTotal);
    // ... 几百个字段的逐行读取 ...
}

// 结构体 → JSON
void JsonDataConvert::CnfGlobalToJson(const struCnfGlobal &CnfGlobal, Json::Value &jsonVal)
{
    CMyJson::WriteIntKey("nLang",       CnfGlobal.nLang,       jsonVal);
    CMyJson::WriteIntKey("nProfile",    CnfGlobal.nProfile,    jsonVal);
    // ... 几百个字段的逐行写入 ...
}
```

193KB 的 `.cpp` 就是 40+ 对这样的映射函数，之所以手写是因为 C++ 没有原生反射机制。

---

## 四、MySqlite — 产量统计[[MySqlite与SQL数据库详解]]

存储位置：`/sdcard/ts/statisticDataLog.db`

```cpp
class MySqlite : public QObject {
    QSqlDatabase db;

    bool openDatabase();        // 打开 .db 文件
    bool creatTable(name);      // 创建统计表
    double calculateSingleColumnAverage(begin, end, columnName);  // 按列求平均
    double calculateSinglePathAverage(begin, end, index);         // 按通道求平均
    void updateStagedThroughout();    // 更新阶段性产量
};
```

查询线程：

```cpp
class queryStatisticThread : public QThread {
    StatisticType m_nStatisticType;   // 产量 / 吹气次数
    QueryType m_nQueryType;           // 实时 / 昨天 / 上周 / 最近3月
    ViewType m_nViewType;             // 前主 / 前辅 / 后主 / 后辅

    void run();                       // 在线程里执行 SQL 查询
};
```

---

## 五、完整数据流

```
保存配置:
  UI → 改 struCnfg.xxx → myFlow.saveGlobal()
    → JsonDataConvert::CnfGlobalToJson(struCnfg, jsonVal)
      → CMyJson::WriteIntKey(...) × N
    → CMyJson::SaveJsonFile("/sdcard/cnf/cnf.global.json", jsonVal)
      → ofstream → write 到 SD 卡

读取配置:
  myFlow.getGlobal()
    → CMyJson::LoadJsonFile("/sdcard/cnf/cnf.global.json", jsonVal)
      → ifstream → jsoncpp::Reader::parse()
    → JsonDataConvert::JsonToCnfGlobal(jsonVal, struCnfg)
      → CMyJson::ReadIntKey(...) × N
  → struCnfg 现在有了所有持久化配置

产量统计:
  updateStatusThread 定时更新
    → MySqlite::updateStagedThroughout()  → INSERT INTO ...
  UI 请求统计图
    → queryStatisticThread::run()
      → SELECT ... FROM ... WHERE time BETWEEN ... AND ...
      → emit statisticQueryFinsishedSig()
        → barchart / plotcurve 绘制图表
```

---

## 六、三个类的定位

| 类 | 层级 | 依赖 QObject | 特点 |
|----|------|-------------|------|
| `CMyJson` | 底层 | 否 | 纯工具，全部 static，封装 jsoncpp |
| `JsonDataConvert` | 中间层 | 否 | 全部 static，结构体↔JSON 映射字典 |
| `MySqlite` | 上层 | 是 | 有信号槽，有独立查询线程 |
