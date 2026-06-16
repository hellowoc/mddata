# HTTP 多部件格式 — QHttpMultiPart 和 QHttpPart

## 数据结构

### QHttpPart

一个 `QHttpPart` 包含两个字段：

```cpp
class QHttpPart {
    QMap<QString, QString> headers;   // 头部信息（Content-Disposition、Content-Type 等）
    QByteArray body;                  // 文本内容（setBody 写入）
    QIODevice *bodyDevice;            // 流式设备，用于大文件（setBodyDevice 写入）
};
```

**与 `QNetworkRequest::setHeader` 的区别**：`QNetworkRequest::setHeader` 设置的是整个 HTTP 请求的头部（如 `Authorization`、`Content-Type`）。`QHttpPart::setHeader` 设置的是 multipart 中**这一个 Part 自己的头部**，仅描述这个 Part 的内容。

### QHttpMultiPart

```cpp
class QHttpMultiPart {
    QList<QHttpPart> parts;    // 内部维护一个 QList，存放所有 Part
};
```

发送时 Qt 遍历 `parts` 链表，将每个 `QHttpPart` 按 `multipart/form-data` 格式序列化，拼接成完整的 HTTP 请求体。

## 方法

### QHttpPart

| 方法 | 作用 |
|------|------|
| `setHeader(QNetworkRequest::ContentDispositionHeader, QVariant(...))` | 设置"这个 Part 叫什么名字、是否是文件" |
| `setHeader(QNetworkRequest::ContentTypeHeader, QVariant(...))` | 设置"这个 Part 的内容类型"（可选，文件用 `application/octet-stream`） |
| `setBody(QByteArray)` | 设置文本内容，数据直接写入 `body` 字段 |
| `setBodyDevice(QIODevice*)` | 设置流式设备（QFile），数据从磁盘流式读取 |

### QHttpMultiPart

| 方法 | 作用 |
|------|------|
| `append(QHttpPart)` | 将 Part 加入 `parts` 链表末尾 |
| `setBoundary(QByteArray)` | 自定义分界符（默认 Qt 自动生成） |

## `append()` 的作用

```cpp
// append 前：
multiPart.parts = []                              // 空链表
textPart_key.header = "name=\"key\""              // Part 有值，但不在容器中
textPart_key.body   = "设备编号"

// append 后：
multiPart.append(textPart_key);
multiPart.parts = [textPart_key]                  // 加入链表
```

`manager.post(request, multiPart)` 时，Qt 遍历 `multiPart.parts`，对链表中每个 `QHttpPart` 执行序列化——取出其 `header` 和 `body`/`bodyDevice`，拼接成：

```
--boundary\r\n
Content-Disposition: form-data; name="key"\r\n
\r\n
设备编号\r\n
--boundary\r\n
Content-Disposition: form-data; name="file"; filename="image.png"\r\n
\r\n
[文件二进制内容]\r\n
--boundary--\r\n
```

## 文本 Part 的构造

```cpp
QHttpPart textPart_key;    // 构造空 Part（headers 空，body 空）

// headers["Content-Disposition"] = "form-data; name=\"key\""
textPart_key.setHeader(QNetworkRequest::ContentDispositionHeader, 
    QVariant("form-data; name=\"key\""));

// body = "设备编号"
textPart_key.setBody(QString("设备编号").toAscii());
```

## 文件 Part 的构造

```cpp
QHttpPart filePart;

// headers["Content-Disposition"] = "form-data; name=\"file\"; filename=\"image.png\""
filePart.setHeader(QNetworkRequest::ContentDispositionHeader, 
    QVariant("form-data; name=\"file\"; filename=\"" + info.fileName() + "\""));

// bodyDevice = QFile 对象（流式读取，不一次性加载到内存）
QFile *file = new QFile(fileName);
file->open(QIODevice::ReadOnly);
filePart.setBodyDevice(file);

// 将 QFile 挂到 multiPart 下，multiPart 析构时自动 delete file
file->setParent(multiPart);
```

## 本工程中 upLoadOssFile 的 9 个 Part

```
multiPart.parts = [
    textPart_key,                   // key — 文件在 OSS 上的路径
    textPart_policy,                // policy — 上传策略（base64 编码的 JSON）
    textPart_success_action_status, // success_action_status — 期望成功状态码 "200"
    textPart_date,                  // x-amz-date — 签名时间
    textPart_signature,             // x-amz-signature — 签名（hex 字符串）
    textPart_algorithm,             // x-amz-algorithm — 签名算法
    textPart_credential,            // x-amz-credential — 凭证信息
    textPart_contenttype,           // Content-Type — 文件类型
    filePart                        // file — 文件内容（QFile 流式读取）
];
```

前 8 个用 `setBody()` 直接写文本（几十字节），最后一个用 `setBodyDevice(QFile*)` 流式读磁盘文件（可能几 MB）。全部 `append()` 到 `multiPart` 后，`manager.post(request, multiPart)` 发送。
