# JSON 解析教程 — jsoncpp

## 本工程使用的库

`3rdparty/jsoncpp/` — C++ JSON 解析库。核心三个类：

| 类 | 作用 |
|------|------|
| `Json::Reader` | 解析器，读取 JSON 字符串，构建 DOM 树 |
| `Json::Value` | DOM 树节点，可以是 Object、Array、String、Int 等任意 JSON 类型 |
| `Json::FastWriter` | 序列化器，将 DOM 树写回 JSON 字符串 |

## 基础模式：解析单层 JSON Object

适用场景：云端返回一个简单的键值对结构。

### JSON 示例

```json
{
    "access_token": "771ee5f4-1fc7-46c1-9583-36bddb551ceb",
    "token_type": "bearer",
    "expires_in": 3600
}
```

### C++ 代码（`requestTokenUpdate` 中）

```cpp
// 1. HTTP 响应 → std::string
QByteArray ba = reply->readAll();
string rcv = QString(ba).toStdString();

// 2. 创建解析器和目标容器
Json::Reader reader;
Json::Value root;

// 3. 解析。reader.parse() 返回 bool：true=成功，false=JSON 格式错误
if (reader.parse(rcv, root))
{
    // 4. 按 key 取出字段
    Json::Value val = root["access_token"];
    g_token = val.asString();           // 转为 std::string

    val = root["token_type"];
    string tokenType = val.asString();

    val = root["expires_in"];
    g_token_expires = val.asInt();      // 转为 int
}
```

### 解析后 root 的内存结构

```
root (type=Object)
├── key "access_token" → StringValue "771ee5f4-..."
├── key "token_type"   → StringValue "bearer"
├── key "expires_in"   → IntValue    3600
└── key "scope"       → StringValue "server"
```

### root["key"] 的行为

`Json::Value` 的 `operator[]` 根据当前节点的类型有不同的行为：

| 当前节点类型 | `root["xxx"]` 含义 |
|-------------|---------------------|
| Object | 按字符串 key 查找子节点，返回该子节点的引用 |
| Array | 按整数下标访问，如 `root[0]` |
| String / Int / Null | 行为未定义，调用可能导致运行时错误 |

`reader.parse()` 成功后 root 为 Object 类型，之后用 `root["xxx"]` 按 key 访问。

### 类型提取方法

| 方法 | 返回值 | 适用类型 |
|------|------|---------|
| `.asString()` | `std::string` | String |
| `.asInt()` | `int` | Int |
| `.asDouble()` | `double` | 浮点数 |
| `.asBool()` | `bool` | Bool |
| `.isString()` | `bool` | 判断当前节点是否为 String |
| `.isInt()` | `bool` | 判断当前节点是否为 Int |
| `.isArray()` | `bool` | 判断当前节点是否为 Array |
| `.isObject()` | `bool` | 判断当前节点是否为 Object |
| `.size()` | `unsigned int` | Array 元素个数 或 Object 键值对个数 |

---

## 进阶模式：解析两级嵌套 JSON

适用场景：云端返回一个数组，每个数组元素内部又包含子数组。

### JSON 示例

```json
[
  {
    "value": "颜色",
    "labels": [
      { "value": "红", "labelType": "color", "id": "001" },
      { "value": "蓝", "labelType": "color", "id": "002" }
    ]
  },
  {
    "value": "形状",
    "labels": [
      { "value": "圆", "labelType": "shape", "id": "003" }
    ]
  }
]
```

### C++ 代码（模型标签解析）

#### 外层数组遍历

```cpp
// root["cmd"] 返回一个 Json::Value，其类型为 Array
Json::Value cmdArray = root["cmd"];    
int size = cmdArray.size();           // Array 的元素个数

for (int i = 0; i < size; i++)
{
    // cmdArray[i] 取第 i 个元素（类型为 Object）
    Json::Value row = cmdArray[i];

    // row["value"] 取该 Object 中 key 为 "value" 的字段
    if (row["value"].isString())      // 先判断类型，避免运行时错误
    {
        QString name = QString::fromStdString(row["value"].asString());
    }
```

#### 内层数组遍历

```cpp
    // row["labels"] 返回另一个 Array
    Json::Value labels = row["labels"];
    int labelCount = labels.size();     // 子数组元素个数

    for (unsigned int j = 0; j < labels.size(); j++)
    {
        Json::Value label = labels[j];  // 子数组的第 j 个元素

        // 取内部 Object 的各个字段
        QString v = QString::fromStdString(label["value"].asString());
        QString t = QString::fromStdString(label["labelType"].asString());
        QString id = QString::fromStdString(label["id"].asString());
    }
}
```

### 嵌套 JSON 对应的内存结构

```
root (type=Array, size=2)
├── [0] (type=Object)
│   ├── key "value"  → String "颜色"
│   └── key "labels" → Array (size=2)
│       ├── [0] (type=Object)
│       │   ├── key "value"     → String "红"
│       │   ├── key "labelType" → String "color"
│       │   └── key "id"        → String "001"
│       └── [1] (type=Object)
│           ├── key "value"     → String "蓝"
│           ├── key "labelType" → String "color"
│           └── key "id"        → String "002"
│
└── [1] (type=Object)
    ├── key "value"  → String "形状"
    └── key "labels" → Array (size=1)
        └── [0] (type=Object)
            ├── key "value"     → String "圆"
            ├── key "labelType" → String "shape"
            └── key "id"        → String "003"
```

---

## 数据转换链：JSON 字符串 → C struct

JSON 解析结果（`std::string`）需要写入 C 语言固定长度 `char[]` 结构体。完整转换路径：

```
Json::Value
  ↓ .asString()
std::string                           // jsoncpp 内部的字符串类型
  ↓ QString::fromStdString()          // std::string → Qt 字符串（支持 Unicode）
QString
  ↓ .toUtf8()                         // QString 内部 UTF-16 → UTF-8 字节流
QByteArray
  ↓ .data()                           // 获取底层字节缓冲区首地址
const char*
  ↓ memcpy(dst, src, strlen(src))     // 按字节拷贝到固定长度 char[] 中
char[固定长度]
```

### 每一步的原因

| 转换 | 为什么需要 |
|------|-----------|
| `.asString()` → `std::string` | jsoncpp 的 `Json::Value` 取值必须通过 asXxx() 方法 |
| `QString::fromStdString()` | `std::string` 没有编解码能力，QString 处理多语言和编码转换 |
| `.toUtf8()` → `QByteArray` | QString 内部是 UTF-16，C 结构体的 `char[]` 需要 UTF-8 |
| `.data()` → `const char*` | `memcpy` 需要源地址指针 |
| `memcpy(dst, src, len)` | C 结构体 `char[]` 是定长数组，不能用 `=` 赋值，只能按字节拷贝 |

---

## 常用模式速查

### 读取单个字符串字段

```cpp
QString value = QString::fromStdString(root["fieldName"].asString());
```

### 读取单个整数字段

```cpp
int value = root["fieldName"].asInt();
```

### 遍历数组

```cpp
Json::Value arr = root["arrayName"];
for (unsigned int i = 0; i < arr.size(); i++)
{
    Json::Value item = arr[i];
}
```

### 判断字段是否存在

```cpp
if (!root["fieldName"].isNull())    // 字段存在且非 null
{
    // 安全访问
}
```

### 判断字段类型

```cpp
if (root["fieldName"].isString())   // 是字符串
if (root["fieldName"].isInt())      // 是整数
if (root["fieldName"].isArray())    // 是数组
if (root["fieldName"].isObject())   // 是对象
```

### 写入 C char[] 结构体三步

```cpp
QString qstr = QString::fromStdString(jsonVal.asString());
QByteArray ba = qstr.toUtf8();
memcpy(struCnfg.xxxName, ba.data(), strlen(ba.data()));
```

---

## 与 HTTP 请求中的 JSON 构造对比

解析是"读取"，反方向是 `onCmdReturn()` 中的构造：

```cpp
Json::Value root;                          // 构造空 Object
root["uuid"]    = uuid;                    // 逐字段赋值
root["cmdCode"] = cmd;
root["result"]  = QString::number(errorcode).toStdString();

Json::FastWriter info;                     // 序列化器
std::string str = info.write(root);         // Object → JSON 字符串

myMqttThread->test->publish(NULL, "topic", str.length(), str.c_str());
```
