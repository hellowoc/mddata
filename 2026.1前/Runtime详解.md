# Runtime 详解

---

## 一、什么是 Runtime

**Runtime 是嵌入在 Qt 程序里的"Linux 工具箱"**，把散落在各处的 shell 命令（`df`、`rm`、`cat /proc/mounts`...）和 Qt 底层 API（`QFile::remove()`、`QProcess`...）封装成统一的、有语义的成员函数。

---

## 二、为什么用 Runtime

### 不用 Runtime 的写法（散装 shell 命令满天飞）

```cpp
// 某处：获取 USB 路径
QProcess p1;
p1.start("/bin/bash", QStringList() << "-c" << "cat /proc/mounts |grep /dev/sd |sed -n '$p' |awk '{printf $2}'");
p1.waitForFinished();
QString usbPath = p1.readAll();

// 另一处：删除文件
#ifdef Q_OS_UNIX
system("sync");
#endif
QFile::remove(filePath);

// 又一处：检查磁盘空间
QProcess p2;
p2.start("df /sdcard");
p2.waitForFinished();
// 手动解析 df 输出...
```

### 用 Runtime 的写法

```cpp
QString usbPath = g_Runtime().getUsbPath();
g_Runtime().delFile(filePath);
g_Runtime().getFreeSpace();
```

所有函数调用后面都有一条明确的底层实现链，下面逐个拆解。

---

## 三、单例模式

### 什么是单例

```cpp
// global.cpp
Runtime &g_Runtime()
{
    static Runtime runtime;   // 全局唯一实例
    return runtime;
}
```

- `static Runtime runtime` — 程序生命周期内只构造一次
- 无论在任何 `.cpp` 文件中调用 `g_Runtime()`，拿到的都是**同一个** Runtime 对象
- 全局状态（`g_level`、`g_layerTotal` 等）只有一份，不会出现数据不一致

### 为什么 Runtime 必须是单例

Runtime 内部持有整机共享状态：

```cpp
int g_level;              // 当前权限等级 —— 全程序只有一个真相
int g_layerTotal;         // 机器总层数 —— 不存在"你的层数""我的层数"
QStringList nimageNameVec; // 图像列表缓存 —— 全局一份
```

如果允许 `new Runtime` 随意创建实例，各实例持有不同版本的 `g_level`，业务逻辑就乱了。

---

## 四、全部成员函数逐层拆解

从 Runtime 对外接口 → Qt 库函数 → Linux 系统调用 / shell 命令。

---

### 1. 文件操作类

#### `delFile(filePath)` — 删除文件

```
Runtime::delFile(filePath)
  → QFile file(filePath)          // QFile 构造，只存路径，无 I/O
    → file.remove()               // 触发实际删除
      → ::unlink(filePath)        // POSIX 系统调用，删除 inode
        → Linux 内核：从文件系统中移除目录项
```

---

#### `copyFileToPath(sourceDir, toDir, coverFileIfExist)` — 拷贝文件

```
Runtime::copyFileToPath(src, dst, cover)
  → QFile::exists(src)                       // stat() 系统调用
  → (如果 cover=true 且 dst 已存在)
      → QDir::remove(dst)                    // unlink() 系统调用
  → QFile::copy(src, dst)                    // 内部 open()+read()+write()+close()
    → ::open(src, O_RDONLY)
    → ::open(dst, O_WRONLY | O_CREAT)
    → while(read(fd_src, buf, 4096))
        write(fd_dst, buf, n)
```

---

#### `copyDirToPath(srcPath, dstPath, coverFileIfExist)` — 递归拷贝目录

```
Runtime::copyDirToPath(src, dst, cover)
  → QDir dstDir(dst)
    → dstDir.mkdir(dst)                    // ::mkdir() 系统调用
  → QDir srcDir(src)
    → srcDir.entryInfoList()               // ::opendir() + ::readdir()
  → foreach fileInfo:
      ├── 是目录 → copyDirToPath(子目录)     // 递归
      └── 是文件 → QFile::copy(src, dst)     // open+read+write
                    → (cover=true) QDir::remove(旧文件)  // ::unlink()
```

---

#### `dirExist(dir)` — 确保目录存在

```
Runtime::dirExist(dir)
  → QDir newdir(dir)
    → newdir.exists()                     // ::stat() 系统调用
    → (不存在) newdir.mkpath(dir)         // ::mkdir() 递归创建所有父目录
```

---

#### `getFileSize(filePath)` — 人类可读的文件大小

```
Runtime::getFileSize(filePath)
  → QFileInfo file(filePath)
    → file.size()                          // ::stat() → st_size 字段
  → 纯 CPU 运算：除以 1024 换算为 B/K/M/G
  → 返回如 "2.35M"
```

---

#### `getFileMd5(filePath)` — 文件 MD5

```
Runtime::getFileMd5(filePath)
  → QFile localFile(filePath)
    → localFile.open(QFile::ReadOnly)      // ::open(path, O_RDONLY)
  → QCryptographicHash ch(QCryptographicHash::Md5)
  → while 文件未读完:
      → localFile.read(4096)               // ::read() 系统调用
      → ch.addData(buf)                    // 纯 CPU（MD5 算法运算）
  → localFile.close()                      // ::close()
  → ch.result() → QByteArray::toHex()      // CPU
```

---

#### `updateTmpImgList()` — 扫描图像目录

```
Runtime::updateTmpImgList()
  → QDir dir("/sdcard/bmp/")
    → dir.setNameFilters({"*.png","*.bmp"})
    → dir.entryInfoList(Files, Time)       // ::opendir() + ::readdir() + ::stat()
  → 遍历结果填入 nimageNameVec（纯内存操作）
```

---

### 2. 系统命令类

#### `mySystem(str)` — 执行 shell 命令（不关心输出）

```
Runtime::mySystem("rm /sdcard/cnf -rf")
  → system("rm /sdcard/cnf -rf")            // C 标准库
    → ::fork()                             // Linux 系统调用：创建子进程
    → 子进程: ::exec("/bin/sh", "-c", "rm /sdcard/cnf -rf")
        → /bin/sh 内部执行 rm 命令
          → rm 内部调用 ::unlink() 逐个删除文件
    → 父进程: ::waitpid()                   // 等待子进程结束
    → WIFEXITED(status) / WEXITSTATUS(status) // 解析退出状态
  → (成功) system("sync")                  // ::sync() 系统调用——刷页缓存到磁盘
```

**`system()` 内部等价于：**

```
system(cmd) 的内部伪代码:
  pid = fork()
  if pid == 0:           // 子进程
    exec("/bin/sh", "-c", cmd)
  else:                   // 父进程
    waitpid(pid, &status)
    return status
```

---

#### `mySystemStr(str)` — 执行 shell 命令（捕获 stdout）

```
Runtime::mySystemStr("cat /proc/mounts |grep sdcard")
  → QProcess process
    → process.start("/bin/bash", {"-c", "cat /proc/mounts |grep sdcard"})
      → ::fork() + ::exec("/bin/bash", "-c", cmd)
    → process.waitForFinished()
      → ::waitpid()
    → process.readAll()                      // ::read() 读取管道中的 stdout
```

`mySystem` 和 `mySystemStr` 的区别：

| | `mySystem` | `mySystemStr` |
|---|---|---|
| 底层 | C `system()` | Qt `QProcess` |
| 获取输出 | 不获取 | 返回 stdout 字符串 |
| sync 刷盘 | 自动 sync | 不 sync |
| 适用场景 | 删文件、改权限、刷盘 | 查询信息（读 /proc 等） |

---

#### `getUsbPath()` — 检测 U 盘挂载路径

```
Runtime::getUsbPath()
  → 先尝试方式1:
    QProcess → bash -c "cat /proc/mounts |grep /dev/sd |sed -n '$p' |awk '{printf $2}'"
    // /proc/mounts 是内核导出的挂载信息虚拟文件
    // 如果返回包含 "/media/sd" → 直接返回

  → 再尝试方式2 (遍历常见挂载点):
    循环 i=0..4:
      QFileInfo::exists("/media/sda" + QString::number(i))
      // ::stat("/media/sda", ...)
      // ::stat("/media/sda1", ...)
      // ::stat("/media/sda2", ...)
      // ...

    循环 i=0..4:
      QFileInfo::exists("/media/sdb" + QString::number(i))
      // ::stat("/media/sdb", ...)
      // ::stat("/media/sdb1", ...)
      // ...
```

**原理**：Linux 内核在 `cat /proc/mounts` 中列出了所有已挂载的文件系统，U 盘设备名一般是 `/dev/sda` 或 `/dev/sdb`，挂载点通常在 `/media/sda` 或 `/media/sdb`。

---

#### `checkUsbExist()` — 检查是否有 USB 存储设备

```
Runtime::checkUsbExist()
  → QDir::exists("/proc/scsi/usb-storage")
    → ::stat("/proc/scsi/usb-storage")
```

`/proc/scsi/usb-storage` 是 Linux 内核导出的目录，USB 存储设备插入后内核会创建它。目录存在 = USB 存储控制器被驱动识别。

---

#### `getUsbSpace(dev)` — 获取分区空间信息

```
Runtime::getUsbSpace("/sdcard")
  → QProcess::start("df /sdcard")
    → ::fork() + ::exec("df", "/sdcard")
  → QProcess::waitForFinished(3000)
  → 逐行 readAll() 解析 df 输出
```

**df 输出示例：**
```
Filesystem     1K-blocks  Used Available Use% Mounted on
/dev/mmcblk0p1  7743744 2048512 5294656  28% /sdcard
```

函数解析这一行字符串，返回各列的列表。

---

#### `getFreeSpace()` — 获取 /sdcard 剩余空间

```
Runtime::getFreeSpace()
  → getUsbSpace("/sdcard")              // QProcess → df
  → 取第4列（Available）
  → 除以 1024.0 换算为 MB
  → 写入 struCnfg.nFreeSpace
```

---

#### `getStartUpMode()` — 判断从 SD 卡还是 eMMC 启动

```
Runtime::getStartUpMode()
  → QProcess → bash -c "cat /proc/mounts |grep sdcard |sed -n '$p' |awk '{printf $1}'"
    → ::fork() + ::exec("/bin/bash", "-c", "...")
  → 解析 stdout:
    /dev/mmcblk0p1 → 0 (SD 卡启动)
    /dev/mmcblk0p2 或 /dev/mmcblk1p2 → 1 (eMMC 启动)
    其他 → -1 (异常)
```

---

#### `checkIpcMounted(ipcIpAddr)` — 检查 IPC 相机是否 NFS 挂载

```
Runtime::checkIpcMounted("192.168.1.100")
  → QProcess → bash -c "cat /proc/mounts |grep 192.168.1.100 |sed -n '$p' |awk '{printf $4}'"
  → 解析挂载属性:
    "rw" → true (读写挂载成功)
    其他 → false
```

---

### 3. 网络信息类

#### `GetIpAddress(netName)` — 获取网卡 IP

```
Runtime::GetIpAddress("eth0")
  → QNetworkInterface::allInterfaces()     // netlink socket → 内核网络栈
  → 遍历每个网络接口，找到名字匹配的
  → QNetworkAddressEntry::ip()             // 返回 IPv4 地址字符串
  → 如 "192.168.1.50"
```

---

### 4. 配置持久化类

#### `saveSetting()` — 保存单一设置

```
Runtime::saveSetting()
  → QSettings setting(CFG_APPSet, QSettings::IniFormat)
    → setting.setValue("vendorIndex", g_vendorIndex)
      → 写入 INI 格式的配置文件到磁盘
        → ::open() + ::write() + ::close()
```

---

#### `save()` — 保存全部配置（JSON + SD 卡同步）

```
Runtime::save()
  → g_Runtime().mySystem("sysctl -w vm.drop_caches=3 > /dev/zero")
    // 先清内核缓存，确保读到最新数据

  → myFlow.saveProfile()           // JSON 写入 profile 参数到 /sdcard/cnf/
    → CMyJson::SaveJsonFile()      // Json::Value → ofstream → ::write()

  → myFlow.saveGlobal()            // JSON 写入全局参数到 /sdcard/cnf/
    → CMyJson::SaveJsonFile()      // Json::Value → ofstream → ::write()

  → (如果 SD 卡存在)
    → QDir::mkdir("/media/mmcblk0p1/cnf")   // ::mkdir()
    → system("cp /sdcard/cnf/* -fr /media/mmcblk0p1/cnf/")
      // fork+exec cp 命令，备份到 SD 卡

  → system("sync")                 // ::sync() 强制刷盘
```

---

### 5. 工具类

#### `fillIn(str, maxLen, c)` — 字符串补齐

```
Runtime::fillIn("hello", 10, ' ')
  → QString 纯内存操作
  → 无系统调用
  → 返回 "hello     "
```

---

#### `getVerdorName()` — 获取品牌名

```
Runtime::getVerdorName()
  → vendorNameList[g_vendorIndex]    // QList 下标访问，纯内存
  → 无系统调用
```

---

#### `updateCornStastic()` — 追加玉米产量统计到 CSV

```
Runtime::updateCornStastic()
  → ::fopen("/sdcard/ts/corndata.csv", "a+")   // 追加模式打开
  → ::fprintf(fp, "%s,%s,%s,%s,%s\n", ...)     // 写入一行 CSV
  → ::fclose(fp)                                 // 关闭
```

---

### 6. 总调用关系图

```
业务层 (widget / manager)
│
│  g_Runtime().xxx()
│
▼
┌─────────────────────────────────────────────────┐
│                  Runtime (单例)                   │
│                                                   │
│  delFile()     copyDirToPath()   getFileMd5()    │
│  copyFile()    dirExist()        getFileSize()   │
│  mySystem()    mySystemStr()     getUsbPath()    │
│  save()        saveSetting()     GetIpAddress()  │
│  getFreeSpace()  getStartUpMode()  fillIn()     │
│  updateCornStastic()  checkIpcMounted()  ...    │
└─────────────────────────────────────────────────┘
│                    │                    │
│    Qt 封装层       │    C 标准库        │   纯内存
│                    │                    │
▼                    ▼                    ▼
QFile::remove()    system()           QString 拼接
QFile::copy()      ::fopen()          数学运算
QFileInfo::size()  ::fprintf()
QDir::exists()     ::fclose()
QDir::mkdir()
QProcess::start()
QSettings::setValue()
QNetworkInterface::
  allInterfaces()
│                    │
▼                    ▼
       Linux 系统调用层
│
├── ::unlink()       删除文件
├── ::stat()         获取文件元信息
├── ::open()         打开文件
├── ::read()         读文件
├── ::write()        写文件
├── ::close()        关闭文件
├── ::mkdir()        创建目录
├── ::fork()         创建子进程
├── ::exec()         执行新程序
├── ::waitpid()      等待子进程
├── ::sync()         刷页缓存
├── ::opendir()      打开目录
├── ::readdir()      读取目录项
├── ::rename()       重命名文件
└── /proc 虚拟文件系统  查询内核状态
```

---

## 五、单例的使用方式

### 获取方式

```cpp
Runtime &g_Runtime()          // 全局函数
{
    static Runtime runtime;   // Meyers 单例
    return runtime;
}
```

### 调用方式

```cpp
// 任何 .cpp 文件，包含 global.h 后即可调用：
g_Runtime().delFile(path);
g_Runtime().getUsbPath();
g_Runtime().mySystem("reboot");
```

### 为什么是引用返回而不是指针

```cpp
Runtime &g_Runtime()  // 返回引用 —— 保证不会是 NULL，调用者不需要判空
```

这是惯例——单例不能被销毁，所以返回引用比返回指针（可能为 NULL）语义更准确。

---

## 六、Runtime 的设计价值总结

| 价值 | 说明 |
|------|------|
| **统一入口** | 所有系统操作走 Runtime，不会出现散装 shell 命令 |
| **平台隔离** | `#ifdef Q_OS_UNIX` 藏在 Runtime 内部，调用者不感知 |
| **错误处理** | `mySystem` 检查 fork/exit status，调用者不需要重复处理 |
| **自动 sync** | 每次写操作后自动刷盘，防止嵌入式设备断电丢数据 |
| **状态集中** | 全局状态（权限、层数、图像列表）存在单例里，全程序一致 |
| **可测试** | 如果换成 mock Runtime，可以脱离真实硬件测试业务逻辑 |
