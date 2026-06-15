# HTTP 普通请求体 — QByteArray

## 与 QHttpMultiPart 的区别

`QByteArray` 用于**纯文本请求体**，数据全部加载到内存中一次性发送。`QHttpMultiPart` 用于**混合内容**（文本 + 文件），文件部分通过 `QFile` 流式读取。两者的选择取决于 `Content-Type`：

| Content-Type | 请求体格式 | 使用 Qt 类型 |
|-------------|-----------|-------------|
| `application/x-www-form-urlencoded` | `key=value&key=value` | `QByteArray` |
| `application/json` | `{"key":"value"}` | `QByteArray` |
| `multipart/form-data` | `--boundary\r\n...\r\n--boundary--` | `QHttpMultiPart` |

## 数据结构

```cpp
class QByteArray {
    char *data;       // 指向堆上分配的内部缓冲区
    int size;         // 当前有效数据长度
};
```

`append()` 将新数据追加到内部缓冲区末尾。`data()` 返回 `char*` 指针，可传给 `memcpy`、`manager.post()` 等需要原始字节指针的函数。

## 本工程中的三种构造方式

### 方式一：append 拼接（表单格式）

```cpp
// requestToken
QByteArray array;
array.append("grant_type=client_credentials");
array.append("&scope=server");
array.append("&code=xiewj-test");
array.append("&model=RZ+8-64X");
array.append("&version=1.1.0");

// 最终 array 内部：
// "grant_type=client_credentials&scope=server&code=xiewj-test&model=RZ+8-64X&version=1.1.0"
```

每次 `append()` 将 `char*` 内容拷贝到 `data` 缓冲区末尾，`size` 同步增长。`Content-Type` 设置为 `application/x-www-form-urlencoded`，服务器收到的请求体会按 `&` 分割键值对、按 `=` 提取 key 和 value。

### 方式二：QString 转换（表单格式）

```cpp
// requestTokenUpdate
QString username = "mobile=DEVICE@" + g_connectName;
QByteArray array;
array.append(username);
// 或直接构造：
QByteArray array(username.toLocal8Bit());
```

`toLocal8Bit()` 将 `QString`（UTF-16 内部编码）转为本地 8 位编码的 `QByteArray`。

### 方式三：QString 转 QByteArray（JSON 格式）

```cpp
// upLoadData
QByteArray updata = QString().fromStdString(data).toLocal8Bit();
// data 是 std::string，先转 QString，再转 QByteArray
```

## 发送

```cpp
QByteArray array;
array.append("...");

QNetworkReply *reply = manager.post(request, array);
// Qt 内部读取 array.data() 和 array.size()，写入 TCP socket
```

`manager.post(request, array)` 内部：
1. 读取 `array.data()` 指针
2. 读取 `array.size()` 得到字节数
3. 自动填写 `Content-Length` 头 = size
4. 将 `data` 指向的字节序列作为 TCP 数据发送

## 接收

```cpp
QByteArray ba = reply->readAll();
```

`readAll()` 内部从 `QNetworkReply` 的接收缓冲区读取所有已到达字节，构造一个新的 `QByteArray` 返回。`data` 指向堆上新分配的缓冲区，`size` 为实际接收字节数。

## QByteArray 与 std::string 互转

```cpp
// QByteArray → std::string
QByteArray ba;
string s = QString(ba).toStdString();

// std::string → QByteArray
string s;
QByteArray ba = QString().fromStdString(s).toLocal8Bit();
```

转换路径：`QByteArray ↔ QString ↔ std::string`。`QString` 作为中间格式处理编码转换。

## QByteArray 与原始指针互转

```cpp
// 从指针构造
char *ptr = ...;
QByteArray array(ptr);        // 调用 strlen(ptr) 确定长度

// 获取内部指针（只读）
const char *data = array.data();     // 返回内部缓冲区的指针
int len = array.size();              // 返回有效数据长度

// 获取内部指针（可写，不常用）
char *data = array.data();
```
