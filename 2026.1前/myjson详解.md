# myjson 文件详解

---

## 一、JSON 是什么

JSON 是一种**纯文本数据格式**，和编程语言无关。它只定义"数据怎么写成字符串"：

```json
{
    "name": "色选机",
    "speed": 120,
    "layers": [1, 2, 3],
    "config": {
        "screenW": 1024,
        "screenH": 768
    }
}
```

规则就 6 条：
- `{}` 包起来的是对象（键值对集合）
- `[]` 包起来的是数组
- 键必须是双引号字符串
- 值可以是：数字、字符串、布尔、null、对象、数组
- 键值对用 `:` 分隔，多个键值对用 `,` 分隔
- 数组元素用 `,` 分隔

任何语言都有库能读 JSON 字符串 → 还原成该语言的数据结构。

---

## 二、jsoncpp 库的核心类型

本项目使用的 JSON 库叫 **jsoncpp**，核心就三个类：

### Json::Value — 万能数据容器

```cpp
Json::Value root;              // 一个空的 JSON 值，可以变成任何类型

root["speed"] = 120;           // 自动变成对象，存入整数
root["name"] = "色选机";        // 存入字符串
root["layers"][0] = 1;        // 自动变成数组，第0个元素 = 1
root["layers"][1] = 2;        // 第1个元素 = 2

// 读取时：
int speed = root["speed"].asInt();        // 取出整数
string name = root["name"].asString();    // 取出字符串
int layer0 = root["layers"][0].asInt();  // 取出数组元素

// 类型检查：
root["speed"].isNull();    // 这个键存在吗？
root["speed"].isInt();     // 值是整数吗？
root["speed"].isString();  // 值是字符串吗？
root["layers"].isArray();  // 值是数组吗？
```

一个 `Json::Value` 对象可以同时是对象、数组、或基本类型，它会根据你怎么用它自动转换。这是 jsoncpp 设计的核心：**一个类型走天下**。

### Json::Reader — 解析器

把 JSON 文本字符串变成 `Json::Value` 对象：

```cpp
Json::Reader reader;                     // 创建解析器
Json::Value root;                        // 准备容器
string jsonStr = "{\"speed\": 120}";     // JSON 文本

bool ok = reader.parse(jsonStr, root);   // 解析 → root 现在包含数据
// ok=true: 成功  ok=false: 格式错误
```

### Json::FastWriter — 写入器

把 `Json::Value` 对象变成 JSON 文本字符串：

```cpp
Json::FastWriter writer;                 // 创建写入器
Json::Value root;
root["speed"] = 120;

string jsonStr = writer.write(root);     // → "{\"speed\":120}\n"
// FastWriter 输出紧凑格式（无缩进）
// 另有 StyledWriter 输出带缩进的漂亮格式
```

---

## 三、CMyJson 类总体结构

```cpp
class CMyJson {
public:
    // ===== 文件操作（2个）=====
    static bool LoadJsonFile(filename, root);    // 读文件 → 解析 → 填入 root
    static bool SaveJsonFile(filename, root);    // root → 写文件

    // ===== 基本类型读写（6个）=====
    static JSON_RETURN_VAL ReadIntKey(root, key, outInt);
    static JSON_RETURN_VAL ReadFloatKey(root, key, outFloat);
    static JSON_RETURN_VAL ReadStringKey(root, key, outStr);
    static JSON_RETURN_VAL WriteIntKey(key, value, root);
    static JSON_RETURN_VAL WriteFloatKey(key, value, root);
    static JSON_RETURN_VAL WriteStringKey(key, value, root);

    // ===== 数组读写（6个）=====
    static JSON_RETURN_VAL ReadIntArrayKey(root, key, dims, outBuf, outArray);
    static JSON_RETURN_VAL ReadFloatArrayKey(root, key, dims, outBuf, outArray);
    static JSON_RETURN_VAL ReadStringArrayKey(root, key, outArray);
    static JSON_RETURN_VAL WriteIntArrayKey(key, dims, buf, root);
    static JSON_RETURN_VAL WriteFloatArrayKey(key, dims, buf, root);
    static JSON_RETURN_VAL WriteStringArrayKey(key, arr, root);

    // ===== 工具（1个）=====
    static int GetArraySize(json_val);

private:
    // 多维数组递归读写辅助（6个）
    static int  _RecursionReadArrayDimension(...);
    static void _RecursionReadArrayData(...);
    static void _RecursionReadFloatArrayData(...);
    static bool _ArrayDimensionIsEqual(...);
    static void _RecursionWriteArrayData(...);
    static void _RecursionWriteFloatArrayData(...);
};
```

全部方法是 **static**（静态方法），意味着**不需要创建 CMyJson 实例就能调用**。

---

## 四、返回值枚举

```cpp
enum JSON_RETURN_VAL {
    JSON_SUCCESS         = 0,   // 操作成功，值已写入输出参数
    JSON_FAILED_INVALID  = 1,   // 键不存在（key 在 JSON 中没有）
    JSON_FAILED_TYPE     = 2,   // 键存在但类型不对（要 int 但存的是 string）
    JSON_FAILED_ABORT    = 3,   // 操作抛异常（文件损坏、内存不足等）
    JSON_FAILED_ARRAY_NUM= 4    // 数组维度不匹配（期望 2×3，实际是 3×2）
};
```

调用方根据返回值决定如何处理：

```cpp
JSON_RETURN_VAL ret = CMyJson::ReadIntKey(root, "nLang", nLang);
if (ret == JSON_SUCCESS) {
    // nLang 已正确赋值，可以用了
} else if (ret == JSON_FAILED_INVALID) {
    // 配置文件中没有这个键 → 用默认值
    nLang = 1;
}
```

---

## 五、文件操作

### SaveJsonFile — 保存为 JSON 文件

```cpp
static bool CMyJson::SaveJsonFile(const char* filename, const Json::Value& root)
{
    Json::FastWriter writer;        // ① 创建写入器
    std::string strOut = writer.write(root);  // ② Json::Value → JSON 文本字符串
    
    ofstream ofs(filename, ios_base::out | ios_base::binary);  // ③ 创建输出文件流
    if (!ofs) return false;         // ④ 打开失败 → 返回 false
    
    ofs << strOut;                  // ⑤ 把字符串写入文件
    ofs.close();                    // ⑥ 关闭文件
    return true;
}
```

**① `Json::FastWriter writer`**
jsoncpp 的写入器，负责把内存中的 `Json::Value` 树结构转成 JSON 文本字符串。FastWriter 输出紧凑格式（无缩进、无换行），文件更小。

**② `writer.write(root)`**
`write()` 的参数是 `Json::Value&`，返回 `std::string`。递归遍历整棵 JSON 树，输出符合 JSON 规范的字符串。如：

```
Json::Value 内部: { "nLang": 1, "name": "test" }
writer.write() → "{\"nLang\":1,\"name\":\"test\"}\n"
```

**③ `ofstream ofs(filename, flags)`**
C++ 标准库的输出文件流（output file stream）。

| 参数 | 值 | 含义 |
|------|-----|------|
| `filename` | `const char*` | 文件路径，如 `"/sdcard/cnf/cnf.global.json"` |
| `ios_base::out` | 打开方式：写入 | 文件不存在则创建，存在则清空 |
| `ios_base::binary` | 二进制模式 | 不做换行符转换（Linux 下无影响，Windows 下防止 `\n` 变 `\r\n`） |

`if (!ofs)` — 如果文件无法打开（路径不存在、权限不够、磁盘满），`ofs` 转换为 `false`。

**④ `ofs << strOut`** — C++ 流操作符，把字符串内容写入文件。底层调用 `::write()` 系统调用。

**⑤ `ofs.close()`** — 关闭文件，释放文件描述符。

---

### LoadJsonFile — 从文件加载 JSON

```cpp
static bool CMyJson::LoadJsonFile(const char* filename, Json::Value& root)
{
    Json::Reader reader;        // ① 创建解析器
    ifstream ifs(filename, ios_base::in | ios_base::binary);  // ② 创建输入文件流
    if (!ifs) return false;     // ③ 打开失败
    
    root.clear();               // ④ 清空旧数据
    return reader.parse(ifs, root);  // ⑤ 解析 → 填入 root
}
```

**① `Json::Reader reader`** — jsoncpp 的解析器。`reader.parse(输入流, 输出容器)` 是核心调用。

**② `ifstream ifs(filename, flags)`** — C++ 标准库的输入文件流（input file stream）。`ios_base::in` 表示只读模式。

**③ `root.clear()`** — 清空 `root` 中原有的数据，防止旧值和解析结果混在一起。

**④ `reader.parse(ifs, root)`**
- `ifs` — 输入流，`Reader` 从流中逐字符读取 JSON 文本
- `root` — 输出，解析成功后 `root` 变成 JSON 数据的树结构
- 返回值：`true` 解析成功，`false` JSON 格式错误（如括号不匹配、缺逗号等）

---

## 六、基本类型读取 — 以 ReadIntKey 为例

```cpp
JSON_RETURN_VAL CMyJson::ReadIntKey(
    const Json::Value& root,    // 从哪里读（已解析好的 JSON 树）
    const string& keyName,      // 找哪个键（如 "nLang"）
    int& outInt)                // 读到的值放这里
{
    try {
        // ① 通过键名取子节点
        Json::Value key = root[keyName];
        
        // ② 检查键是否存在
        if (key.isNull())
            return JSON_FAILED_INVALID;
        
        // ③ 检查类型是否正确
        if (key.isInt()) {
            outInt = key.asInt();
            return JSON_SUCCESS;
        }
        return JSON_FAILED_TYPE;
    }
    catch (...) {
        return JSON_FAILED_ABORT;
    }
}
```

**① `root[keyName]`** — jsoncpp 的重载操作符。如果 `root` 是对象且有这个键，返回对应的 `Json::Value`。如果键不存在，返回一个"空值"（`isNull()` 为 `true`）。

**② `key.isNull()`** — 判断这个 `Json::Value` 是否是空值。jsoncpp 中用 `isNull()` 判断键是否存在。

**③ `key.isInt()` / `key.asInt()`** — 类型判断和提取：

| 判断方法 | 提取方法 | 说明 |
|---------|---------|------|
| `isInt()` | `asInt()` | 整数 |
| `isDouble()` | `asDouble()` | 浮点数 |
| `isString()` | `asString()` | 字符串 |
| `isBool()` | `asBool()` | 布尔 |
| `isArray()` | `size()` + `[i]` | 数组 |
| `isObject()` | `[key]` | 对象 |
| `isNull()` | — | 空值 |

**④ `try-catch(...)`** — 捕获一切异常。如果 JSON 树结构损坏或内存不足，jsoncpp 会抛异常，catch 住后返回 `JSON_FAILED_ABORT`，防止整个程序崩溃。

---

## 七、基本类型写入 — 以 WriteIntKey 为例

```cpp
JSON_RETURN_VAL CMyJson::WriteIntKey(
    const string& keyName,      // 键名
    int intData,                // 要写入的值
    Json::Value& root)          // 写入到哪里
{
    try {
        root[keyName] = intData;   // 一行搞定
        return JSON_SUCCESS;
    }
    catch (...) {
        return JSON_FAILED_ABORT;
    }
}
```

`root[keyName] = intData` — jsoncpp 自动处理：如果 `root` 中已有这个键 → 覆盖旧值；如果还没这个键 → 自动创建；如果 `root` 原来不是对象类型 → 自动转为对象。

---

## 八、多维数组读取 — ReadIntArrayKey（最复杂的部分）

### 问题背景

JSON 中数组可以是多维的：

```json
{
    "matrix": [
        [1, 2, 3],
        [4, 5, 6]
    ]
}
```

这是一个 2×3 的二维数组。需要：
1. 自动检测数组的维度
2. 把嵌套的 JSON 数组展平成一段连续的 C++ int 数组
3. 校验维度是否和期望的一致

### 主函数

```cpp
JSON_RETURN_VAL CMyJson::ReadIntArrayKey(
    const Json::Value& root,          // JSON 树
    const string& keyName,            // 如 "matrix"
    const vector<int>& inDimensions,  // 期望的维度，如 {2, 3}
    int* outBuffer,                   // 成功时：把连续数据写到这里
    INT_ARRAY_KEY& outArray)          // 失败时：把实际维度和数据放这里
{
    try {
        outArray.Dimensions.clear();
        outArray.Datas.clear();
        
        Json::Value key = root[keyName];
        if (key.isNull())
            return JSON_FAILED_INVALID;
        if (!key.isArray())
            return JSON_FAILED_TYPE;

        // 步骤A：递归检测实际维度
        int DimensionCount = _RecursionReadArrayDimension(key, outArray.Dimensions);
        // outArray.Dimensions 现在存了实际维度，如 {2, 3}
        
        // 步骤B：计算总元素数
        int data_total = 1;
        while (DimensionCount--)
            data_total *= outArray.Dimensions[DimensionCount];
        // 2 × 3 = 6 个元素

        // 步骤C：维度匹配？
        if (_ArrayDimensionIsEqual(inDimensions, outArray.Dimensions)) {
            // 匹配 → 数据直接填入调用者提供的缓冲区
            _RecursionReadArrayData(key, data_total, outBuffer);
            return JSON_SUCCESS;
        } else {
            // 不匹配 → 把实际数据填入 outArray.Datas 返回，让调用者知道实际情况
            int* pIntData = new int[data_total];
            _RecursionReadArrayData(key, data_total, pIntData);
            for (int i = 0; i < data_total; i++)
                outArray.Datas.push_back(*(pIntData + i));
            delete[] pIntData;
            return JSON_FAILED_ARRAY_NUM;
        }
    }
    catch (...) {
        return JSON_FAILED_ABORT;
    }
}
```

### 步骤A：递归检测数组维度

```cpp
int CMyJson::_RecursionReadArrayDimension(
    const Json::Value& key,        // 当前正在检查的 JSON 数组节点
    vector<int>& ArrayDimension)   // 输出：各维度的大小
{
    int number = GetArraySize(key);  // 当前数组有多少元素
    if (number > 0) {
        ArrayDimension.push_back(number);  // 记录这一维的大小
        // 拿第0个元素继续往下探 → 看是不是还是数组
        return _RecursionReadArrayDimension(key[0], ArrayDimension);
    }
    else
        return ArrayDimension.size();  // 到底了，返回总维度数
}
```

递归过程示意：

```
JSON: [[1,2,3], [4,5,6]]

第1次调用: key = [[1,2,3], [4,5,6]]
  → number = 2 (有2个元素)
  → ArrayDimension = [2]
  → 递归: key[0] = [1,2,3]

第2次调用: key = [1,2,3]
  → number = 3 (有3个元素)
  → ArrayDimension = [2, 3]
  → 递归: key[0] = 1

第3次调用: key = 1 (不是数组了)
  → number = 0 (GetArraySize 对非数组返回0)
  → return 2  (维度总数)
```

**`GetArraySize(json_val)`** — 工具函数：
```cpp
int CMyJson::GetArraySize(const Json::Value& json_val)
{
    return (json_val.isArray() ? json_val.size() : 0);
    //      ↑ 是数组？              ↑ 取元素个数    ↑ 不是数组 → 返回0
}
```

### 步骤B：计算总元素数

```cpp
int data_total = 1;
while (DimensionCount--)
    data_total *= outArray.Dimensions[DimensionCount];
// DimensionCount=2: data_total = 1 × 3 × 2 = 6
```

### 步骤C：递归提取数据到连续缓冲区

```cpp
void CMyJson::_RecursionReadArrayData(
    const Json::Value& key,    // 当前 JSON 数组节点
    int data_total,            // 这个子树有多少个叶子元素
    int* outData)              // 输出缓冲区
{
    int number = GetArraySize(key);  // 当前层有多少元素
    if (number > 0) {
        bool bret = key[0].isArray();  // 第0个元素还是数组吗？
        for (int i = 0; i < number; i++) {
            if (bret) {
                // 是数组 → 继续递归，每段数据量 = data_total / number
                _RecursionReadArrayData(key[i], data_total / number, 
                                        outData + i * data_total / number);
            }
            else {
                *(outData + i) = key[i].asInt();  // 到底了 → 直接取值
            }
        }
    }
}
```

递归过程：

```
调用: _RecursionReadArrayData([[1,2,3],[4,5,6]], 6, outData)
  number=2, bret=true
  i=0: 递归 → _RecursionReadArrayData([1,2,3], 3, outData+0)
          number=3, bret=false
          i=0: outData[0] = 1
          i=1: outData[1] = 2
          i=2: outData[2] = 3
  i=1: 递归 → _RecursionReadArrayData([4,5,6], 3, outData+3)
          number=3, bret=false
          i=0: outData[3] = 4
          i=1: outData[4] = 5
          i=2: outData[5] = 6

结果: outData = [1, 2, 3, 4, 5, 6]
```

### 维度校验

```cpp
bool CMyJson::_ArrayDimensionIsEqual(
    const vector<int>& number1,   // 期望维度 {2, 3}
    const vector<int>& number2)   // 实际维度 {2, 3}
{
    if (number1.size() != number2.size())   // 维度数不同 → false
        return false;
    for (int i = 0; i < (int)number1.size(); i++)
        if (number1[i] != number2[i])        // 某维大小不同 → false
            return false;
    return true;
}
```

---

## 九、多维数组写入

```cpp
JSON_RETURN_VAL CMyJson::WriteIntArrayKey(
    const string& keyName,              // 键名
    const vector<int>& inDimensions,    // 维度，如 {2, 3}
    const int* bufData,                 // 连续存储的数据 {1,2,3,4,5,6}
    Json::Value& root)
{
    try {
        root.removeMember(keyName);     // 删除旧值（如果存在）
        Json::Value key;
        _RecursionWriteArrayData(inDimensions, bufData, key);  // 递归写入
        root[keyName] = key;            // 挂到 root 上
        return JSON_SUCCESS;
    }
    catch (...) {
        return JSON_FAILED_ABORT;
    }
}
```

递归写入辅助：

```cpp
void CMyJson::_RecursionWriteArrayData(
    const vector<int>& inDimensions,  // [2, 3]
    const int* bufData,               // [1,2,3,4,5,6]
    Json::Value& key)
{
    int DimensionCount = inDimensions.size();  // 2
    int data_total = 1;
    for (int i = 0; i < DimensionCount; i++)
        data_total *= inDimensions[i];         // 6

    int number = inDimensions[0];  // 第一维大小 = 2
    for (int i = 0; i < number; i++) {
        if (1 != DimensionCount) {
            // 还有更深维度 → 切掉第一维，递归
            vector<int> vec(inDimensions.begin() + 1, inDimensions.end());
            // vec = [3]
            _RecursionWriteArrayData(vec, bufData + i * data_total / number, key[i]);
            //                            ↑ bufData + 0*3 或 bufData + 1*3
        }
        else {
            key.append(*(bufData + i));  // 到底了 → 直接 append
        }
    }
}
```

逻辑和读取的递归完全对称——把连续的 C++ 数组按维度信息逐层包装成嵌套的 jsoncpp `Json::Value` 数组。

---

## 十、OUT 宏和类型别名

```cpp
#define OUT
```

这是一个空宏，纯粹给读代码的人看的标记——说明这个参数是输出参数。如：

```cpp
static JSON_RETURN_VAL ReadIntKey(root, key, int& outInt OUT);
//                                               └── 标记：这是输出参数
```

```cpp
struct INT_ARRAY_KEY {
    vector<int> Dimensions;   // 如 {2, 3}
    vector<int> Datas;        // 如 {1,2,3,4,5,6}
};
```

---

## 十一、全部函数的分类总结

```
CMyJson 提供的能力：

文件层面:
  LoadJsonFile  ← ifstream + Json::Reader::parse
  SaveJsonFile  → Json::FastWriter::write + ofstream

标量读写:
  ReadIntKey    ← root[key].isNull? isInt? asInt
  ReadFloatKey  ← root[key].isNull? isDouble? asDouble
  ReadStringKey ← root[key].isNull? isString? asString
  WriteIntKey   → root[key] = intData
  WriteFloatKey → root[key] = floatData
  WriteStringKey→ root[key] = strData

数组读写:
  ReadIntArrayKey   ← 递归探维度 + 递归取数据 + 维度校验
  ReadFloatArrayKey ← 同上（float 版本）
  ReadStringArrayKey← 一维循环取值
  WriteIntArrayKey  → 递归按维度包装 + 写入
  WriteFloatArrayKey→ 同上（float 版本）
  WriteStringArrayKey→ 一维循环 append

工具:
  GetArraySize → json_val.isArray() ? json_val.size() : 0
```

全部 static，不实例化，不存状态。就是 jsoncpp 的一层类型安全封装。

---

## 十二、为什么要封装而不是直接调 jsoncpp

一个例子说明：

```cpp
// 直接调 jsoncpp — 每个调用处都要写：
Json::Value key = root["nLang"];
if (!key.isNull() && key.isInt()) {
    nLang = key.asInt();
} else {
    nLang = 1;  // 默认值
}
// 50 个调用处 → 50 份同样的防御代码

// 封装后 — 调用处只需一行：
CMyJson::ReadIntKey(root, "nLang", nLang);
// 内部已经包含了判空、判类型、try-catch
```

封装的核心价值：**防御代码只写一次，所有调用处受益。**
