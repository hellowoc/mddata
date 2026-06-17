# HTTP 响应数据处理

## HTTP 返回的是什么

**字节流，没有固定格式。** HTTP 协议只规定报文结构（状态行+头+空行+体），响应体是什么格式由服务器的 `Content-Type` 头决定：

```
Content-Type: application/json          → 体是 JSON 字符串
Content-Type: application/octet-stream  → 体是二进制数据（文件）
Content-Type: text/html                 → 体是 HTML 网页
Content-Type: application/xml           → 体是 XML
```

## 为什么用 QByteArray 接收

HTTP 传的是**字节**，不是 QString，不是 int，不是结构体。`QByteArray` 是 Qt 里装字节的容器，不假设编码、不假设格式，原样存下来。

```cpp
QByteArray ba = reply->readAll();    // 所有 HTTP 请求都是这一行
```

## 接收之后的五种处理方式

### 方式一：JSON 解析（最常见）

```cpp
// requestToken、requestTokenUpdate、upLoadData
QByteArray ba = reply->readAll();
string rcv = QString(ba).toStdString();       // QByteArray → QString → std::string

Json::Reader reader;
Json::Value root;
reader.parse(rcv, root);                      // 按 JSON 解析
g_token = root["access_token"].asString();    // 取字段
```

### 方式二：直接写入文件（下载）

```cpp
// downLoadFile
QByteArray ba = reply->readAll();
myfile.write(ba);       // 不解析，原样写入磁盘
myfile.close();
```

### 方式三：只看状态码（网络检测）

```cpp
// canWebsiteVisit
int res = reply->attribute(HttpStatusCode).value<int>();
if (res == 200) return true;    // 不读响应体，只关心状态码
return false;
```

### 方式四：XML 解析（OSS 上传回包）

```cpp
// OSS 上传
QByteArray ba = reply->readAll();
QDomDocument doc;
doc.setContent(ba);                      // 按 XML 解析
// 从 <Location> 标签取出文件 URL
```

### 方式五：debug 打印

```cpp
QByteArray ba = reply->readAll();
qDebug() << QString(ba);    // 不解析，打印出来看看
```

## 总结

```
HTTP 返回字节流 → QByteArray 原样接收 → 根据 Content-Type 决定处理方式
                                      ├── JSON  → jsoncpp 解析
                                      ├── 文件  → QFile 写入磁盘
                                      ├── XML   → QDomDocument 解析
                                      └── 不管  → 只看状态码
```
