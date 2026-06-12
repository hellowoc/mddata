# HTTP 协议详解

## 跟 UDP 对比，先搞清楚 HTTP 在哪一层

```
你的程序
    │
    ├── UDP：  你直接构造字节 → sendto() → 对方 recvfrom()
    │          没有格式要求，你们两个约定好字节含义就行
    │
    └── HTTP： 你按 HTTP 格式发文本 → Qt 帮你走 TCP + TLS → 服务器解析
               必须遵守 HTTP 格式（方法行、请求头、请求体）
```

| | UDP | HTTP |
|------|-----|------|
| 传输层 | UDP（不可靠） | TCP（可靠） |
| 格式 | 随便（自定义二进制） | **严格规定**（方法行 + 头 + 体） |
| 连接 | 无连接 | 请求-响应，一次请求一次响应 |
| 谁用 | 工控机 ↔ FPGA | 工控机 ↔ 云端服务器 |

HTTP 跑在 TCP 之上，TCP 保证数据完整到达，HTTP 只规定数据的**格式**。

## 一次 HTTP 请求的全过程

以 `requestTokenUpdate` 为例——向云端申请 OAuth2 Token：

### 第一步：构造请求

你的 Qt 代码做的事：

```cpp
QNetworkRequest request;
request.setUrl("https://cloud.chinaamd.cn:8900/auth/mobile/token/device");
request.setRawHeader("Authorization", "Basic YXBpOmFwaQ==");
request.setRawHeader("TENANT_ID", "1");
request.setHeader(QNetworkRequest::ContentTypeHeader, "application/x-www-form-urlencoded");

QByteArray data;
data.append("mobile=DEVICE@ZKGDDEV47");

QNetworkReply *reply = manage.post(request, data);
```

### 第二步：Qt 内部做的事（你看不到的）

Qt 把上面的代码转成**标准的 HTTP 报文**，通过 TCP 发出去：

```
POST /auth/mobile/token/device HTTP/1.1          ← 方法行
Host: cloud.chinaamd.cn:8900                      ← 请求头
Authorization: Basic YXBpOmFwaQ==
TENANT_ID: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 28
                                                   ← 空行，头结束
mobile=DEVICE@ZKGDDEV47                            ← 请求体
```

### 第三步：服务器收到 → 处理 → 回复

服务器返回的 HTTP 报文：

```
HTTP/1.1 200 OK                                    ← 状态行
Content-Type: application/json                     ← 响应头
Content-Length: 85
                                                   ← 空行
{"access_token":"771ee5f4...","expires_in":3600}   ← 响应体
```

### 第四步：你的代码读响应

```cpp
QByteArray ba = reply->readAll();    // 拿到响应体
// 解析 JSON → 取出 access_token → 存到 g_token
```

## HTTP 报文的结构

**请求报文**（你发给服务器）：

```
方法  路径  HTTP版本         ← 第一行：你要做什么
请求头1: 值1                ← 附带信息
请求头2: 值2
                           ← 空行
请求体                       ← 可选，发送的数据
```

**响应报文**（服务器回给你）：

```
HTTP版本  状态码  状态文字   ← 第一行：结果是什么
响应头1: 值1                ← 附带信息
                           ← 空行
响应体                       ← 真正的数据
```

## HTTP 方法 — GET vs POST

| | GET | POST |
|------|-----|------|
| 用途 | 拿数据（查询、下载） | 发数据（提交、上传） |
| 请求体 | 无 | 有 |
| 参数位置 | URL 后面 `?key=value` | 请求体中 |
| 本工程例子 | `downLoadFile()` — 下载文件 | `requestTokenUpdate()` — 提交认证 |
| Qt 调用 | `manage.get(request)` | `manage.post(request, data)` |

类比：GET 是取快递（告诉快递员取件码，他给你包裹），POST 是寄快递（你把包裹给他，他给你运单号）。

## HTTP 状态码 — 服务器告诉你结果

三位数字，分类：

| 开头 | 含义 | 本工程例子 |
|------|------|-----------|
| 1xx | 信息 | 没用到 |
| 2xx | **成功** | 200 OK、201 Created |
| 3xx | 重定向 | 没用到 |
| 4xx | **客户端错误**（你发的不对） | 401 没权限、404 找不到 |
| 5xx | **服务器错误**（服务器炸了） | 500 内部错误 |

本工程只检查两种结果：

```cpp
int res = reply->attribute(QNetworkRequest::HttpStatusCodeAttribute).value<int>();
if (res == HTTP_STATUS_OK || res == HTTP_STATUS_CREATED)  // 200 或 201 → 成功
    ret = TRUE;
else
    ret = FALSE;
```

## Content-Type — 告诉服务器"我发的是什么格式"

| 类型 | 本工程用途 |
|------|-----------|
| `application/x-www-form-urlencoded` | Token 申请、普通数据提交（`key=value&key=value`） |
| `application/json` | JSON 格式数据 |
| `multipart/form-data` | 文件上传（`upLoadFile`） |

服务器根据这个值决定如何解析请求体——如果发的是 JSON 但你告诉它是表单格式，服务器解析失败。

## HTTPS = HTTP + TLS 加密

HTTP 明文传输，中途任何人截获数据包都能看到内容。HTTPS 在 HTTP 和 TCP 之间加了一层 TLS 加密：

```
HTTP   → 明文："Authorization: Bearer abc123" → 被截获 → 泄露
HTTPS  → 密文："xA3fG9kL..."                 → 被截获 → 解不开
```

本工程的 TLS 配置：

```cpp
QSslConfiguration config = request.sslConfiguration();
config.setPeerVerifyMode(QSslSocket::VerifyNone);  // 跳过域名验证
config.setProtocol(QSsl::TlsV1);                   // 用 TLSv1 协议
request.setSslConfiguration(config);
```

`VerifyNone` 的意思是"不验证服务器的证书是不是正规 CA 签发的"——工控机连的是厂商自建的云端服务器，用的是自签名证书，正规 CA 不签发给这种内部服务器，不跳过验证就连不上。

## 本工程中的 HTTP 请求完整流程

```
1. QNetworkAccessManager manage;          创建管理器
2. QNetworkRequest request;               创建请求
3. request.setUrl(...)                    设 URL
4. request.setRawHeader("Authorization", "Bearer xxx")  设认证头
5. request.setHeader(ContentType, ...)    设内容类型
6. 配置 SSL（跳过证书验证）                 设加密
7. QEventLoop eventLoop;                  创建临时事件循环
8. connect(&manage, finished, &eventLoop, quit);  连接完成信号
9. QNetworkReply *reply = manage.post(request, data);  发请求（异步）
10. eventLoop.exec();                     阻塞等待响应
11. 检查 reply->error() == NoError?       成功？
12. QByteArray ba = reply->readAll();     读响应体
13. reply->deleteLater();                 清理
```

第 10 步 `eventLoop.exec()` 就是**卡住等结果**——跟之前 `myMessageBox::exec()` 一样。套一个临时事件循环，把异步网络请求变成同步等待。但必须注意：**这段代码所在的线程会被卡住**，如果在主线程调 HTTP 请求且网络慢（10 秒超时），主线程会卡 10 秒。正因如此，所有 HTTP 请求都在 `mqttMsgParaseThread`（独立线程）中执行，不影响 UI。
