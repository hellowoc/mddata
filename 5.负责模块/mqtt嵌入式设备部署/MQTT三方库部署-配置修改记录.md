# MQTT 三方库部署 — 配置修改记录

## 当前状态

工程版本：`~/桌面/v7.0.9/qtrun-aiuse/`（亦可用于 `~/桌面/mqtt-config/`）
MQTT 功能被四处禁用，已全部解封。

---

## 一、mqttsrv.h — 类定义解封

**文件**：`zksort/bus/mqttsrv.h`
**位置**：第 77 行
**操作**：`#if 0` → 删除，改为标记注释。删除对应的多余 `#endif`（第 298 行）

```cpp
// [zcy 2026.06.15] mqtt config
// 为了适配 MQTT 的第三方支持和各项文件依赖所进行的改动
#ifdef Q_OS_UNIX
class mqttsrv : public mosqpp::mosquittopp { ... };
class mqttThread : public QThread { ... };
class mqttMsgParaseThread : public QThread { ... };
extern mqttThread *myMqttThread;
extern mqttMsgParaseThread *myMqttMsgParaseThread;
#endif
```

---

## 二、mqttsrv.cpp — 全局变量解封

**文件**：`zksort/bus/mqttsrv.cpp`
**位置**：第 4 行
**操作**：`#if 0` → 删除。删除文件末尾对应的多余 `#endif`（第 4991 行）

```cpp
// [zcy 2026.06.15] mqtt config
// 为了适配 MQTT 的第三方支持和各项文件依赖所进行的改动
#ifdef Q_OS_UNIX
mqttThread *myMqttThread;
mqttMsgParaseThread *myMqttMsgParaseThread;
#endif
```

---

## 三、globalflow.cpp — 线程创建/启动代码恢复

**文件**：`zksort/global/globalflow.cpp`
**位置**：第 10846-10850 行
**操作**：取消注释，恢复 MQTT 线程创建和条件启动

```cpp
// [zcy 2026.06.15] mqtt config
// 为了适配 MQTT 的第三方支持和各项文件依赖所进行的改动
#ifdef Q_OS_UNIX
    myHttpFileClient = new MyHttpFileClient;
    myMqttThread = new mqttThread;
    myMqttMsgParaseThread = new mqttMsgParaseThread;
    if (ipForInterface == "162.254.129.100"){
        myMqttThread->start();
        myMqttMsgParaseThread->start();
    }
#endif
```

---

## 四、zksort.pro — 链接 mosquitto 库

**文件**：`zksort/zksort.pro`
**位置**：第 589 行
**操作**：取消注释，改用 `3rdparty/mqtt/` 下的本地库，添加 OpenSSL

```diff
- #LIBS += -L$$PWD/../3rdparty/mqtt -lmosquittopp -lmosquitto
+ # [zcy 2026.06.15] mqtt config: 启用 MQTT 库链接
+ LIBS += -L$$PWD/../3rdparty/mqtt -lmosquittopp -lmosquitto -lssl -lcrypto
```

---

## 五、mosquitto 1.6.15 库编译部署

**问题**：系统 `libmosquitto-dev` 为 2.0.11，已移除 C++ wrapper（`mosquittopp`），与工程不兼容。

**解决**：下载 mosquitto 1.6.15 源码 → 编译 x86_64 → 放入 `3rdparty/mqtt/`

```bash
# 下载编译
curl -L -o /tmp/mosquitto-1.6.15.tar.gz https://mosquitto.org/files/source/mosquitto-1.6.15.tar.gz
cd /tmp && tar xzf mosquitto-1.6.15.tar.gz && cd mosquitto-1.6.15
make WITH_TLS=yes -j$(nproc)

# 复制到工程
cp lib/libmosquitto.so.1 3rdparty/mqtt/
cp lib/cpp/libmosquittopp.so.1 3rdparty/mqtt/
cd 3rdparty/mqtt && ln -sf libmosquitto.so.1 libmosquitto.so && ln -sf libmosquittopp.so.1 libmosquittopp.so
```

**产物**：
- `3rdparty/mqtt/libmosquitto.so.1`（493KB）
- `3rdparty/mqtt/libmosquittopp.so.1`（36KB）
- 对应符号链接 `.so`

---

## 六、运行时配置

### CFG_APPSet.ini
**路径**：`/sdcard/cfg/CFG_APPSet.ini`

```ini
[General]
connectName=ZKGDDEV47
testIp=cloud.chinaamd.cn
```

### root.crt（TLS 证书）
**路径**：`/app/root.crt`

从 Broker 提取：`openssl s_client -connect cloud.chinaamd.cn:1884`

### LD_LIBRARY_PATH 更新
`qt_run.sh` 新增 `3rdparty/mqtt` 路径：
```bash
export LD_LIBRARY_PATH=.../uikits/lib/unix:.../3rdparty/mqtt
```

---

## 七、依赖检查清单

| 检查项 | 状态 |
|------|------|
| `bus/mosquitto.h` 头文件 | ✅ 已有（mosquitto 1.x） |
| `bus/mosquittopp.h` 头文件 | ✅ 已有（C++ wrapper） |
| `3rdparty/mqtt/` 库文件 | ✅ x86_64 .so 已编译 |
| `libssl-dev` | ✅ 已安装 |
| `/sdcard/cfg/CFG_APPSet.ini` | ✅ 已创建 |
| `/app/root.crt` | ✅ 已提取 |
| ARM 交叉编译 mosquitto | ❌ 待处理（缺 ARM OpenSSL） |

---

## 八、编译验证

```bash
qmake zksort.pro && make -j$(nproc)   # 无错误
nm dist/zksort | grep mosquitto       # 符号已链接
ldd dist/zksort | grep mosquitto      # 依赖正确
timeout 5 ./zksort                    # 运行无崩溃
```

---

## 修改记录

| 时间 | 步骤 | 变更 |
|------|------|------|
| 2026-06-15 | 步骤1 | 安装 libmosquitto-dev 2.x → 发现不兼容 |
| 2026-06-15 | 步骤1 | 下载编译 mosquitto 1.6.15 x86_64 |
| 2026-06-15 | 步骤2 | 解封 mqttsrv.h 类定义 |
| 2026-06-15 | 步骤3 | 解封 mqttsrv.cpp 全局变量 |
| 2026-06-15 | 步骤4 | 恢复 globalflow.cpp 线程创建 |
| 2026-06-15 | 步骤5 | 取消注释 zksort.pro 链接配置 |
| 2026-06-15 | 步骤6 | 创建 CFG_APPSet.ini + root.crt |
| 2026-06-15 | 步骤7 | 更新 qt_run.sh LD_LIBRARY_PATH |
| 2026-06-15 | 编译 | 编译通过，运行正常 |
