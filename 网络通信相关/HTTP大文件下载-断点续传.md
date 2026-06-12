# HTTP 大文件下载 — 断点续传

## 两种下载模式对比

本工程实现了两种 HTTP 文件下载：

| | downLoadFile (MyHttpFileClient) | startDownload (httpDownloadManager) |
|------|------------|--------------|
| 接收方式 | `eventLoop.exec()` 阻塞等，`readAll()` 一次性读全部 | `readyRead` 信号驱动，每次收到多少就写入磁盘多少 |
| 内存占用 | 整个文件加载到内存（`QByteArray`） | 边收边写磁盘，内存中只暂存当前接收的片段 |
| 断点续传 | 不支持 | 支持（`Range` 头 + 文件 `Append` 打开） |
| 进度通知 | 无 | `downloadProgress` 信号实时发出 |
| 适用场景 | 小文件（几 KB 到几 MB） | 大文件（固件、模型，几十 MB 到几百 MB） |

## 信号驱动的接收流程

```
TCP 数据到达 → 内核通知 Qt → QNetworkReply 发射信号

  数据到达              全部接收完成              传输过程中
    readyRead()           finished()              downloadProgress()
       ↓                      ↓                        ↓
  downloadReadyRead()    downloadFinished()     downloadProgress()
   write(file)            MD5 校验 + 通知          通知进度百分比
```

整个过程不依赖任何 `QTimer`。信号发射频率由三个因素决定：
1. 网络速度（TCP 包到达速率）
2. Qt 内部缓冲区大小（默认 64KB，缓冲区满才通知）
3. 事件循环处理速度

## 断点续传的三要素

### 1. 客户端记录已下载字节数

```cpp
m_bytesCurrentReceived = QFileInfo(myfile).size();
// 读取本地文件的当前大小——这个就是已下载的字节数
```

### 2. 客户端发送 HTTP Range 头

```cpp
if (m_isBreakPointOk){
    QString strRange = QString("bytes=%1-").arg(m_bytesCurrentReceived);
    m_Request.setRawHeader("Range", strRange.toLatin1());
}
// myfile 已有 2048 字节 → bytes=2048-
// 服务器收到后：跳过前 2048 字节，从第 2049 字节开始发送
```

Range 头是 HTTP 协议标准（IETF RFC 7233）。`bytes=N-` 表示从第 N 字节开始发送到文件末尾。服务器必须支持此标准。

### 3. 客户端以追加模式打开文件

```cpp
myfile.open(QIODevice::WriteOnly | QIODevice::Append);
//                                         ↑
// Append 模式：新数据追加到文件已有内容之后，不覆盖
```

`Range` 让服务器从断点开始发送，`Append` 让本地文件从断点开始写入。两者配合——服务器发来的数据正好拼接到本地文件已有的内容之后。

## 不支持断点续传时的行为

`m_isBreakPointOk == false` 时，`Range` 头不被设置，服务器从第 0 字节开始发送完整文件。此时如果本地文件已有部分内容，`Append` 模式会导致文件内容错乱（旧内容 + 从头开始的完整新内容拼接在一起）。

`startDownload()` 在开头检测到 `m_bytesCurrentReceived <= 0` 时（非续传场景），主动删除旧文件：

```cpp
if (m_bytesCurrentReceived <= 0)
{
    removeFile(TMPFILE);            // 删除旧临时文件
    removeFile(MODEL_FILE_PATH + "...");  // 删除旧模型文件
}
```

## MD5 校验（downloadFinished 中）

```cpp
if (m_Reply->error() == QNetworkReply::NoError) {
    if (g_Runtime().getFileMd5(TMPFILE) == unit.md5.toUpper())
        res = SUCCESS;
}
```

下载完成后计算本地文件的 MD5 值，与云端在下载任务中提供的 MD5 比较。匹配 → 文件完整无误。不匹配 → 传输中数据损坏 → 返回 `FILEERROR`。

## 信号链路（从底层到槽）

```
Linux 内核 TCP 栈
  → 数据到达 → 标记 socket 可读
    → Qt 事件分发器 (QAbstractEventDispatcher)
      → QNetworkReply 内部从 socket 读取字节
        → QNetworkReply 内部缓冲区累积数据
          → 发射 readyRead() → downloadReadyRead()
          → 更新接收计数器 → 发射 downloadProgress() → downloadProgress()
    → TCP 收到 FIN / 全部接收
      → 发射 finished() → downloadFinished()
```

没有定时器参与——TCP 层数据到达触发中断，经内核协议栈处理后转换为 Qt 事件。信号是数据驱动而非时间驱动。
