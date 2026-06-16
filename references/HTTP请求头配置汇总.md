# HTTP 请求头配置汇总

本工程 `bus/myhttpfileclient.cpp` 中所有 `request.setHeader()` 和 `request.setRawHeader()` 的配置。

## request 级别的头

### setRawHeader — 自定义头

| 头名称             | 值                    | 作用                                                                             | 出现位置                                                                                        |
| --------------- | -------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `Authorization` | `Basic YXBpOmFwaQ==` | HTTP Basic Auth。`YXBpOmFwaQ==` 是用户名密码 `api:api` 的 Base64 编码。用于还没有 Token 时的身份验证 | `requestToken`、`requestTokenUpdate`                                                         |
| `Authorization` | `Bearer {g_token}`   | Bearer Token 认证。`g_token` 是已获取的 OAuth2 令牌，服务器通过它识别设备身份                         | `upLoadFile`、`upLoadData`、`requestOssUploadSign`、`requestModelLableList`、`requestModelList` |
| `TENANT_ID`     | `1`                  | 租户编号。云端服务器按租户隔离数据。本工程写死 `"1"`                                                  | 除 `downLoadFile` 和 `canWebsiteVisit` 外的所有函数                                                 |
| `client_id`     | `7BIp1rTdiM...`      | OAuth2 客户端 ID，Base64 编码的固定凭证。与 `client_secret` 配对                              | 仅 `requestToken`                                                                            |
| `client_secret` | `7BIp1rTdiM...`      | OAuth2 客户端密钥                                                                   | 仅 `requestToken`                                                                            |
| `User-Agent`    | `MyOwnBrowser 1.0`   | 标识 HTTP 客户端的类型，服务器可据此区分请求来源                                                    | `downLoadFile`                                                                              |
| `Range`         | `bytes=已下载-`         | HTTP 断点续传头。告诉服务器从指定字节偏移之后开始传输                                                  | `httpDownloadManager`                                                                       |

### setHeader — 标准 HTTP 头

| 头常量 | 值 | 作用 | 出现位置 |
|------|------|------|---------|
| `ContentTypeHeader` | `application/x-www-form-urlencoded` | 请求体格式为表单键值对 `key=value&key=value` | `requestToken`、`requestTokenUpdate` |
| `ContentTypeHeader` | `application/json` | 请求体格式为 JSON | `upLoadData`、`requestVPNCertificate`、`requestOssUploadSign`、`requestModelList` |

## Part 级别的头

这些是 `QHttpPart.setHeader()` 设置的，只描述 multipart 中**单个 Part** 的内容：

| 头常量 | 值 | 作用 |
|------|------|------|
| `ContentDispositionHeader` | `form-data; name="key"` | Part 的字段名 |
| `ContentDispositionHeader` | `form-data; name="file"; filename="xxx.png"` | Part 包含一个文件 |
| `ContentTypeHeader` | `application/octet-stream` | Part 内容是二进制流 |

## 层级关系

```
HTTP 请求
├── request.setHeader / setRawHeader    ← 整个请求的头
│   ├── Authorization: Basic/Bearer     ← 身份认证
│   ├── ContentTypeHeader: json/form    ← 请求体整体格式
│   ├── TENANT_ID: 1                    ← 数据隔离
│   └── Range: bytes=...                ← 断点续传
│
└── 请求体
    ├── QByteArray                       ← 对应 Content-Type: json/form
    └── QHttpMultiPart                   ← 对应 multipart/form-data
        └── QHttpPart.setHeader()        ← 单个 Part 的 Content-Disposition
```

## 固定配置（每个请求）

```
SSL 跳过证书验证 + TLSv1
   ↓
QTimer 10 秒超时
   ↓
eventLoop.exec() 阻塞等待
```
