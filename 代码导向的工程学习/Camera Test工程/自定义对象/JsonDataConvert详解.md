# JsonDataConvert 详解

---

## 一、定位：C++ struct ↔ JSON 文件的翻译字典

```
JsonDataConvert = 全静态方法集合
  不存状态，不继承任何类
  唯一作用：把 C++ 结构体的每个字段对应到 JSON 的每个 key

┌──────────────────────┐         ┌──────────────────────┐
│  struCnfGlobal       │         │  cnf.global.json     │
│  ├─ nLang: 1         │  ←──→   │  "nLang": 1          │
│  ├─ nLayerTotal: 2   │         │  "nLayerTotal": 2    │
│  ├─ struLayerInfo[8] │         │  "struLayerInfo": [  │
│  └─ ... (100+字段）   │          │    {...}, {...}      │
└──────────────────────┘         │  ]                   │
                                └──────────────────────┘
              翻译官：JsonDataConvert
```

193KB 的 `.cpp` 文件为什么这么大？因为 **40+ 个结构体 × 每个几十到上百个字段 × 两个方向（读+写）= 几千行逐字段映射**。

---

## 二、整体架构

```cpp
class JsonDataConvert {
public:
    // ===== 顶层入口（4个）=====
    static bool JsonToCnfGlobal(json, cnfGlobal);         // JSON → 全局配置
    static void CnfGlobalToJson(cnfGlobal, json);         // 全局配置 → JSON
    static bool JsonTostruCnfProfile(json, cnfProfile);   // JSON → 方案参数
    static bool StruCnfProfileToJson(cnfProfile, json);   // 方案参数 → JSON

private:
    // ===== 子结构体映射（40+ 个）=====
    static bool layerInfoToJson(...);       // 层信息结构体
    static bool JsonToLayerInfo(...);
    static bool stuViewInfoToJson(...);     // 视信息结构体
    static bool JsonToStuViewInfo(...);
    static bool stuCameraInfoToJson(...);   // 相机信息结构体
    static bool JsonToStuCameraInfo(...);

    // ===== 算法参数映射（20+ 种算法）=====
    static bool JsonTostruGreyColor(...);       // 灰度算法
    static bool struGreyColorToJson(...);
    static bool JsonTostruIntel(...);           // SVM 算法
    static bool struIntelToJson(...);
    static bool JsonTostruHsv(...);             // HSV 算法
    static bool struHsvToJson(...);
    static bool JsonTostruColorSat(...);        // 颜色饱和度算法
    static bool struColorSatToJson(...);
    // ... 还有 20+ 对算法映射 ...
};
```

---

## 三、模式一：基本类型字段的映射

这是最基础的模式，占代码量的 60% 以上。

### JSON → 结构体（读）

```cpp
bool JsonDataConvert::JsonToCnfGlobal(const Json::Value &jsonVal, struCnfGlobal &cnf)
{
    CMyJson::ReadIntKey(jsonVal, "nLang",        cnf.nLang);
    CMyJson::ReadIntKey(jsonVal, "nProfile",     cnf.nProfile);
    CMyJson::ReadIntKey(jsonVal, "nLayerTotal",  cnf.nLayerTotal);
    CMyJson::ReadIntKey(jsonVal, "nChuteTotal",  cnf.nChuteTotal);
    CMyJson::ReadFloatKey(jsonVal, "nEjectDurationMaxValue", cnf.nEjectDurationMaxValue);
    // ... 逐字段，100+ 行 ...
}
```

每行做的事完全一样：

```
JSON 中的 "nLang": 1  ──ReadIntKey──→  cnf.nLang = 1
JSON 中的 "nLayerTotal": 2  ──ReadIntKey──→  cnf.nLayerTotal = 2
```

### 结构体 → JSON（写）

```cpp
void JsonDataConvert::CnfGlobalToJson(const struCnfGlobal &cnf, Json::Value &jsonVal)
{
    CMyJson::WriteIntKey("nLang",       cnf.nLang,       jsonVal);
    CMyJson::WriteIntKey("nProfile",    cnf.nProfile,    jsonVal);
    CMyJson::WriteIntKey("nLayerTotal", cnf.nLayerTotal, jsonVal);
    // ... 逐字段，100+ 行 ...
}
```

读和写的代码几乎完全对称——同一个字段，读用 `ReadIntKey`，写用 `WriteIntKey`。

### 为什么字段名要一致

JSON 中的 key 和 C++ 结构体的成员变量**刻意保持同名**（如 `"nLang"` ↔ `cnf.nLang`），这样映射逻辑清晰、不易出错。如果不同名就需要额外的对照表。

---

## 四、模式二：子结构体数组的映射

当一个字段不是 `int` 而是嵌套的结构体数组时，需要单独写映射函数。

### 示例：层信息结构体

```cpp
// JSON 中的表示：
"struLayerInfo": [
    { "nViewTotal": 2, "nUnitTotal": 8, "nViewName": "前视" },
    { "nViewTotal": 2, "nUnitTotal": 8, "nViewName": "后视" }
]

// C++ 中的定义：
struct stru_layer_info {
    int nViewTotal;
    int nUnitTotal;
    char sViewName[64];
    // ...
};
```

读取时的调用链：

```cpp
// JsonToCnfGlobal 中：
JsonToLayerInfo(MAX_LAYER, jsonVal["struLayerInfo"], cnf.struLayerInfo);
//              ↑           ↑                          ↑
//          数组大小     JSON 中的数组节点          C++ 结构体数组

// JsonToLayerInfo 内部：
bool JsonToLayerInfo(int arraySize, const Json::Value &jsonVal, stru_layer_info *arr)
{
    for (int i = 0; i < arraySize; i++) {
        CMyJson::ReadIntKey(jsonVal[i], "nViewTotal", arr[i].nViewTotal);
        CMyJson::ReadIntKey(jsonVal[i], "nUnitTotal", arr[i].nUnitTotal);
        // ... 逐字段映射 ...
    }
}
```

模式是：**主映射函数调用子结构体映射函数，子结构体映射函数内部又是逐字段的 ReadIntKey/WriteIntKey**。

---

## 五、模式三：字符串的特殊处理

C++ 结构体中有两种字符串表示：

```cpp
struct stu_color_arith {
    char sName[64];      // C 风格定长字符数组
    int nArithEnable;
    // ...
};
```

如果是 `std::string`，直接 `ReadStringKey` / `WriteStringKey` 即可。但这里是 `char[64]`，需要额外处理：

```cpp
// JSON → char[64]
string strsName;
CMyJson::ReadStringKey(jsonVal[i], "sName", strsName);    // 先读到 std::string
memset(struGreyColor[i].sName, 0, sizeof(struGreyColor[i].sName));  // 清空
memcpy(struGreyColor[i].sName, strsName.data(), strsName.length());  // 拷贝进去
```

```cpp
// char[64] → JSON
string strsName = string(struGreyColor[i].sName);  // 先转成 std::string
CMyJson::WriteStringKey("sName", strsName, jsonVal[i]);  // 再写入
```

这是因为 jsoncpp 和 CMyJson 都只认 `std::string`，不认 C 风格 `char[]`，所以需要一个中转。

---

## 六、模式四：多维数组的映射

```cpp
// JsonToCnfGlobal 中：
intVec.clear();
intVec.push_back(MAX_ALARM);    // 期望维度是 {MAX_ALARM}
CMyJson::ReadIntArrayKey(jsonVal, "nAlarmValid", intVec, cnf.nAlarmValid, outArray);
//                                              ↑        ↑
//                                          期望维度   输出缓冲区
```

多维数组的读写完全委托给 CMyJson 的 `ReadIntArrayKey` / `WriteIntArrayKey`，JsonDataConvert 不需要再做额外处理，只需要传正确的维度参数。

---

## 七、193KB 文件的内容分布

```
jsondataconvert.cpp = 193KB

内容分布（估算）：
  顶层映射 (JsonToCnfGlobal / CnfGlobalToJson)            ≈ 400行
  方案映射 (JsonTostruCnfProfile / StruCnfProfileToJson)   ≈ 500行
  层/视/相机/阀门/灯光等子结构体映射 (20+ 对)              ≈ 1500行
  算法参数映射 (20+ 种算法 × 2方向)                        ≈ 1800行
  总行数:                                                    ≈ 4200行

全部是同一模式的重复：
  CMyJson::ReadXxxKey(jsonVal, "fieldName", struct.fieldName);
  CMyJson::WriteXxxKey("fieldName", struct.fieldName, jsonVal);
```

---

## 八、调用链全貌

```
保存全局配置：
  myFlow.saveGlobal()
    → Json::Value jsonVal;
    → JsonDataConvert::CnfGlobalToJson(struCnfg, jsonVal)
        → WriteIntKey("nLang", ...) × N
        → layerInfoToJson(...)          // 子结构体
            → WriteIntKey(...) × M
        → profileIndexToJson(...)       // 另一个子结构体
        → ...
    → CMyJson::SaveJsonFile("/sdcard/cnf/cnf.global.json", jsonVal)

读取全局配置：
  myFlow.getGlobal()
    → Json::Value jsonVal;
    → CMyJson::LoadJsonFile("/sdcard/cnf/cnf.global.json", jsonVal)
        → ifstream + Json::Reader::parse
    → JsonDataConvert::JsonToCnfGlobal(jsonVal, struCnfg)
        → ReadIntKey("nLang", ...) × N
        → JsonToLayerInfo(...)
        → JsonToProfileindex(...)
        → ...
    → struCnfg 所有字段已填充完毕
```

---

## 九、和 CMyJson 的分工对比

| | CMyJson | JsonDataConvert |
|---|---------|----------------|
| 知道 JSON 格式吗 | 知道（通过 jsoncpp） | 不知道 |
| 知道 C++ 结构体定义吗 | 不知道（只认 int/float/string） | 知道 |
| 做的事 | "从 JSON 取一个叫 nLang 的 int" | "struCnfGlobal 的 nLang 字段对应 JSON 的 nLang 键" |
| 层的位置 | 底层工具 | 中间翻译层 |

可以这样理解：

```
CMyJson       = 翻译机（能把 JSON 文本变成 key-value，反之亦然）
JsonDataConvert = 对照表（"nLang 这个 key 的值应该放到 struCnfg.nLang 这个字段"）

两者配合:
  JsonDataConvert 查对照表 → 告诉 CMyJson "读 key=nLang, 类型=int"
  → CMyJson 从 JSON 读到值 → JsonDataConvert 把它填入正确的结构体字段
```

---

## 十、为什么不用自动化方案

C++ 没有原生反射机制（不像 Java/C# 能自动枚举类的所有字段），所以无法写一个"自动把 struct 序列化为 JSON"的通用函数。可选的方案：

| 方案 | 成本 | 本项目用了吗 |
|------|------|------------|
| 手写逐字段映射 | 4200 行重复代码 | ✓ 是的 |
| 宏批量生成 | 需要定义一套宏语法 | 否 |
| 代码生成器 | 需要写生成器脚本 | 否 |
| protobuf 等序列化框架 | 引入新依赖，改变数据定义方式 | 否 |

手写虽然冗长，但**最简单、最可控、编译最快、不引入新依赖**——对工业嵌入式项目来说是正确的选择。
