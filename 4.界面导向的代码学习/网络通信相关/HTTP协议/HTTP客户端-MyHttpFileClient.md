# HTTP 客户端 — MyHttpFileClient

## 文件

`CameraTest(aiuse)/zksort/bus/myhttpfileclient.h` + `myhttpfileclient.cpp`

## 全局单例

```cpp
extern MyHttpFileClient *myHttpFileClient;
// 程序启动时在 globalflow.cpp 中 new
```

## 两个公开成员变量

| 变量 | 类型 | 作用 |
|------|------|------|
| `g_token` | `string` | OAuth2 Bearer Token，所有需要鉴权的 HTTP 请求都带它 |
| `g_token_expires` | `int` | Token 有效期（秒），过期后需续期 |
| `g_connectName` | `QString` | 设备编号（如 "ZKGDDEV47"），从 INI 文件读取 |

## 全部函数

### 构造函数

从 `QSettings` 读取 `connectName`（设备注册编号）。`g_token` 初始为空。

### Token 认证

| 函数 | 作用 | 调用时机 |
|------|------|---------|
| `requestToken(url)` | 首次申请 Token（client_id + client_secret） | 程序启动 |
| `requestTokenUpdate(url)` | 用设备编号续期 Token | Token 过期，mqttMsgParaseThread 主循环 |

流程：HTTP POST → 云端返回 JSON `{"access_token":"xxx","expires_in":3600}` → 解析存入 `g_token`、`g_token_expires`。

### 文件操作

| 函数 | 作用 | 方式 |
|------|------|------|
| `downLoadFile(url, path)` | 下载文件到本地 | HTTP GET，一次性全量（不支持断点） |
| `upLoadFile(url, path)` | 上传本地文件 | HTTP POST，`multipart/form-data` |
| `upLoadData(url, data)` | 上传字符串数据 | HTTP POST，`application/x-www-form-urlencoded` |

请求头带 `Authorization: Bearer {g_token}`。

### VPN

| 函数 | 作用 |
|------|------|
| `requestVPNCertificate(url)` | 向云端申请 VPN 证书文件 |

### 阿里云 OSS

| 函数 | 作用 |
|------|------|
| `requestOssUploadSign(url, data)` | 向云端请求 OSS 上传临时签名 |
| `upLoadOssFile(fileName)` | 用签名直传文件到 OSS |

### AI 模型

| 函数 | 作用 |
|------|------|
| `requestModelLableList(url)` | 获取模型标签/类别列表 |
| `requestModelList(urlpath, data)` | 按条件搜索模型列表 |

### 网络检测

| 函数 | 作用 |
|------|------|
| `canWebsiteVisit(url)` | 测试 URL 是否可达（HTTP GET + 5 秒超时） |

## httpDownloadManager — 断点续传下载器

| 函数 | 作用 |
|------|------|
| `setDownloadInfo(UpdateUnit)` | 设置下载信息（URL、MD5、版本） |
| `setBreakPointSupport(bool)` | 启用断点续传 |
| `startDownload()` | 开始下载（带 `Range: bytes=已下载-` 头） |
| `stopDownload()` / `stopWork()` | 暂停 / 停止 |
| `reset()` | 重置 |
| `sigDownloadFinished(int)` | 信号：下载完成 |
| `sigDownloadProcess(qint64, qint64)` | 信号：下载进度 |
| `sigDownloadError()` | 信号：下载出错 |

## 本工程中的 HTTP 调用链

```
系统启动
  → requestToken() / requestTokenUpdate()   获取 Token

mqttMsgParaseThread 主循环
  → Token 过期 → requestTokenUpdate()       续期

用户操作
  → IPSetWidget "申请证书" → requestVPNCertificate()

自动任务
  → uploadLogFile() → upLoadFile()          日志上传
  → uploadRealTimePara() → upLoadData()     实时参数上报
```
