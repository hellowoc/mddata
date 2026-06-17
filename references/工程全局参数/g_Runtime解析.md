# `g_Runtime()` 解析

## 定义

在 `global.cpp:17-20` 中：

```cpp
Runtime &g_Runtime()
{
    static Runtime runtime;   // 函数内静态局部变量
    return runtime;
}
```

这是经典的 **C++ 单例模式（Meyers Singleton）**——全局只有一个 `Runtime` 实例，首次调用时构造，程序结束时自动析构。[[Runtime详解]]

---

## 类继承关系

`Runtime` 的继承链非常短：

```
QObject              ← Qt 框架的基类
  └── Runtime        ← 本项目自定义，无中间父类
```

没有多层继承。`QObject` 是 Qt 所有对象的根基，提供：
- **信号/槽机制**（signal/slot）
- **对象树内存管理**（父对象销毁时自动销毁子对象）
- **元对象系统**（`Q_OBJECT` 宏支持运行时类型信息）
- 事件处理等

`Runtime` 还用了 `LOG4QT_DECLARE_QCLASS_LOGGER` 宏，给这个类注入了一个 `logger()` 成员，可以直接写日志。

---

## Runtime 的成员函数按职责分类

### 1. 系统命令执行（底层能力）

| 函数 | 实现方式 | 作用 |
|------|---------|------|
| `mySystem(str)` | C 标准库 `system()` | 执行 shell 命令，成功返回 1，并自动 `sync` 刷盘 |
| `mySystemStr(str)` | `QProcess` + `/bin/bash -c` | 执行 shell 命令，**返回 stdout 字符串** |

这两个是 Runtime 最底层的"与 Linux 系统对话"的能力，其他很多函数都建立在这之上。

### 2. USB 存储检测（嵌入式设备的关键外设）

| 函数 | 作用 |
|------|------|
| `checkUsbExist()` | 检查 `/proc/scsi/usb-storage` 是否存在，判断是否有 USB 存储设备 |
| `getUsbPath()` | 扫描 `/media/sda*` `/media/sdb*` 找到 U 盘的挂载路径并返回 |
| `getUsbSpace(dev)` | 调用 `df` 命令获取指定分区的空间信息 |

这就是 `main()` 阶段三里 `g_Runtime().getUsbPath()` 返回的路径来源——它遍历 `/media/sda`、`/media/sda1`...`/media/sdb`... 找到第一个存在的挂载点。

### 3. 文件操作

| 函数 | 作用 |
|------|------|
| `copyFileToPath(src, dst, cover)` | 拷贝单个文件，可选覆盖 |
| `copyDirToPath(src, dst, cover)` | **递归**拷贝整个目录 |
| `delFile(path)` | 删除文件 |
| `dirExist(dir)` | 检查目录是否存在，不存在则创建 |
| `getFileSize(path)` | 返回人类可读的文件大小（如 `2.35M`） |
| `getFileMd5(path)` | 分块读取文件计算 MD5 校验值 |
| `updateTmpImgList()` | 扫描 `/sdcard/bmp/` 下的 `.png` `.bmp`，填充 `nimageNameVec` 列表 |
| `getCurImageLayerAndView()` | 从图像文件名 `层_视_单元` 解析出 layer/view/unit 编号 |

### 4. 配置持久化

| 函数 | 作用 |
|------|------|
| `saveSetting()` | 把 `vendorIndex` 写入 `QSettings`（INI 格式） |
| `save()` | 先清内核缓存 → 调用 `myFlow.saveProfile()` + `myFlow.saveGlobal()` 写 JSON → 同步到 SD 卡 → `sync` 刷盘 |

`save()` 是整个系统"保存配置"的总入口。

### 5. 网络信息

| 函数 | 作用 |
|------|------|
| `GetIpAddress(netName)` | 遍历网络接口（eth0/wlan 等），返回指定接口的 IPv4 地址 |
| `checkIpcMounted(ipcIpAddr)` | 检查摄像头 IPC 的网络文件系统是否以读写模式挂载 |

### 6. 辅助工具

| 函数 | 作用 |
|------|------|
| `fillIn(str, maxLen, c)` | 用指定字符把字符串填充到固定长度（右补齐） |
| `getVerdorName()` | 根据 `g_vendorIndex` 返回品牌名（如 "zk"/"amd"/"ucs" 等） |
| `getStartUpMode()` | 读 `/proc/mounts` 判断从 SD 卡(0)还是 eMMC(1)启动 |
| `getFreeSpace()` | 调用 `getUsbSpace("/sdcard")` 获取 SD 卡剩余空间并转为 MB |
| `updateCornStastic()` | 把玉米产量统计数据追加写入 `/sdcard/ts/corndata.csv` |

---

## Runtime 的核心成员变量

| 变量 | 用途 |
|------|------|
| `g_level` | 当前权限级别（0=客户/1=工程师/2=管理员） |
| `g_level2` | 工程师可见等级（1/2/3级） |
| `g_skinIndex` | 皮肤主题编号 |
| `g_layerTotal` | 机器总层数（从配置文件读取） |
| `g_vendorIndex` | 品牌索引（0=中科 1=amd 2=ucs ...） |
| `appPositionX/Y` | 程序窗口在屏幕上的绝对位置 |
| `nimageNameVec` | `/sdcard/bmp/` 下的所有图像文件路径列表 |
| `m_sampleMap[2]` | 采样数据（前后视两个方向） |
| `vendorNameList` | 品牌名称列表 |
| `arithIndexName` | 整机识别算法名称列表 |

---

## 一句话总结

`Runtime` 就是这个嵌入式系统的**"操作系统抽象层"**——把 Linux 底层操作（shell 命令、文件系统、USB 检测、网络查询）封装成业务层可以直接调用的 C++ 方法。`g_Runtime()` 用单例模式保证全局只有一个实例，任何地方都可以直接拿到它。
