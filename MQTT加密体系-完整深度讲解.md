# MQTT 加密体系 — 完整深度讲解

文档主线：本工程 MQTT 需要 TLS 加密——先把 TLS 依赖的四个底层概念挨个讲清（对称加密、非对称加密、哈希、数字签名），再用这四个概念拼出 TLS 握手全流程，然后把证书体系讲透，最后回到本工程 `mqttsrv.cpp:93-95` 三行代码逐行拆解。加密基础 → TLS 协议 → 证书 → OpenSSL → 本工程代码，这条线贯穿全文。

---

## 第一章：加密基础

在理解 TLS 之前，需要先理解 TLS 使用的四个基础构件。

### 1.1 对称加密 — 同一把钥匙

```
加密过程：
  明文 "Hello World" + 密钥 "mysecret12345678"
    → AES-256-CBC 加密
    → 密文 (一串乱码字节，如 0xE3 0xA2 0x1B ...)

解密过程：
  密文 + 同样的密钥 "mysecret12345678"
    → AES-256-CBC 解密
    → 明文 "Hello World"
```

AES（Advanced Encryption Standard）的算法原理简述：

```
AES 把明文分成 128 位（16 字节）的块，对每个块执行：
  1. 用密钥扩展出多轮"轮密钥"
  2. 对数据块执行：字节替换(查S盒) → 行移位 → 列混淆 → 轮密钥异或
  3. 根据密钥长度不同，执行不同轮数：
     AES-128：10 轮
     AES-256：14 轮
  4. 每轮都在"混淆"数据，经过足够多轮后，密文和明文之间的统计关系完全消除

安全性基础：在不知道密钥的情况下，尝试所有可能的密钥（2^128 种 ≈ 3.4×10^38）
即使每秒尝试 10 亿亿次，也需要几十亿年。
```

核心特征：加密和解密用同一把密钥，速度快——加密 1MB 数据耗时微秒级。

致命缺陷：密钥怎么安全地传给对方？如果不加密传输密钥，密钥被截获就白加密了。

### 1.2 非对称加密 — 两把不同的钥匙

```
生成密钥对：
  随机选两个大质数 p 和 q（如各 1024 位）
  计算 n = p × q
  通过数学推导得到 e（公钥指数）和 d（私钥指数）
  公钥 = (n, e)，可以公开
  私钥 = (n, d)，必须保密

加密（用公钥）：
  明文 M → M^e mod n → 密文 C
  用公钥(n, e)加密后的内容，只能被私钥(n, d)解开

解密（用私钥）：
  密文 C → C^d mod n → 明文 M
```

RSA 的数学原理简述：

```
基础依赖：欧拉定理
  a^φ(n) ≡ 1 (mod n)

选 e 和 d 满足：e × d ≡ 1 (mod φ(n))
  即 e × d = k × φ(n) + 1

加密：C = M^e mod n
解密：C^d = (M^e)^d = M^(e×d) = M^(k×φ(n)+1) = M^(k×φ(n)) × M ≡ 1^k × M = M (mod n)

关键：已知 n 和 e，无法高效算出 d——需要知道 φ(n)。
而 φ(n) = (p-1)(q-1)，需要先分解 n 得到 p 和 q。
大整数质因数分解至今没有高效算法——这就是 RSA 的安全基础。
```

核心特征：
- **公钥加密 → 私钥解密**：实现"只有私钥持有者能看"的加密通信
- **私钥加密 → 公钥解密**：实现"证明这条消息确实来自私钥持有者"的数字签名
- 速度慢——比对称加密慢 100-1000 倍。不适合加密大量数据。

**对称 vs 非对称的分工**：TLS 用非对称加密安全传输一个"临时密钥"，然后用这个临时密钥切换到对称加密传输业务数据。各取所长。

### 1.3 哈希函数 — 单向指纹

```
SHA256 的特性：

输入："Hello World"
→ 输出：a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e
       (32 字节，256 位，16 进制表示是 64 个字符)

输入："Hello World."（最后加了一个点）
→ 输出：99b2dfbaa6ebae3d783bd455827f77f5f2e0e9e33891b8462d26793ed6c84a3b
       (完全不同的输出，尽管输入只差一个字符)
```

三个核心特性：

1. **定长输出**：无论输入是 1 字节还是 1GB，输出永远是 32 字节（SHA256）
2. **单向不可逆**：从哈希值无法倒推出原文
3. **防碰撞**：实际上无法找到两个不同输入产生同一个哈希值

用途：
- **数据完整性校验**：文件下载完成后算哈希和原始哈希对比
- **数字签名的输入**：先对消息算哈希，再对哈希值签名（比直接对长消息签名快）

### 1.4 数字签名 — 哈希 + 非对称加密

```
签名过程（持有私钥的人）：
  要签名的消息 → SHA256(消息) → 哈希值 H
  H + 私钥 → RSA 私钥加密 → 签名值 S
  把 (消息 + 签名值 S) 发出去

验证过程（任何人都能验证，用公钥）：
  收到的消息 → SHA256(消息) → 哈希值 H1
  收到的签名 S + 公钥 → RSA 公钥解密 → 哈希值 H2
  比较 H1 == H2？
    == → 两个事实同时成立：
         1. 消息没有被篡改（因为 H1 是从收到的消息算的，和 H2 匹配说明没改）
         2. 消息确实来自私钥持有者（只有私钥能产生可以被公钥解开的签名）
    != → 消息被篡改，或者签名是伪造的
```

---

前面讲清了四个基础构件。TLS 协议就是把这四个东西串起来：用非对称加密交换对称密钥，用哈希做数据校验，用数字签名验证书，最后用对称加密传输业务数据。

---

## 第二章：TLS 协议 — 握手与加密传输

### 2.1 TLS 在 TCP 和 MQTT 之间的位置

```
┌─────────────────────────────┐
│  MQTT 报文（应用层）          │  ← 你的 JSON {"cmdCode":"CMD_Feed",...}
├─────────────────────────────┤
│  TLS 记录层                  │  ← ★ 加密发生在这里
│  (加密/解密 MQTT 报文)        │     明文 MQTT → 加密 → TLS 帧发出
│                              │     收到 TLS 帧 → 解密 → MQTT 明文
├─────────────────────────────┤
│  TCP 字节流                  │  ← TCP 看到的只是加密后的乱码
├─────────────────────────────┤
│  IP / 以太网                  │
└─────────────────────────────┘
```

本工程的端口：**1884**。标准 MQTT 端口是 1883（明文），1884 是加了 TLS 的 MQTT。

### 2.2 密码套件 — 决定用什么算法

TLS 握手时双方协商使用哪个"密码套件"。一个密码套件定义了三个算法的组合：

```
TLS_RSA_WITH_AES_256_CBC_SHA
  │   │    │    │    │   │
  │   │    │    │    │   └── 哈希算法：SHA（用于 HMAC 消息认证）
  │   │    │    │    └────── 对称加密模式：CBC（Cipher Block Chaining）
  │   │    │    └─────────── 对称加密算法：AES-256
  │   │    └──────────────── 加密类型：WITH
  │   └───────────────────── 密钥交换 + 身份认证：RSA
  └───────────────────────── 协议层：TLS

含义：
  密钥交换用 RSA（服务器公钥加密会话密钥）
  数据传输用 AES-256-CBC（对称加密）
  消息认证用 SHA（防篡改）
```

### 2.3 TLS 握手 — 完整流程

```
客户端（工控机）                              服务端（cloud.chinaamd.cn:1884）
    │                                               │
    │── 1. ClientHello ────────────────────────→  │
    │    TLS 版本: TLSv1                             │
    │    支持的密码套件列表                            │
    │    随机数: client_random (28 字节)              │
    │                                               │
    │←── 2. ServerHello ─────────────────────────│
    │    TLS 版本: TLSv1 (选定)                      │
    │    选定密码套件: TLS_RSA_WITH_AES_256_CBC_SHA  │
    │    随机数: server_random (28 字节)              │
    │                                               │
    │←── 3. Certificate ─────────────────────────│
    │    服务器证书链 (X.509 格式)                    │
    │    包含: 域名、公钥、颁发者(CA)、有效期、签名    │
    │                                               │
    │    4. 客户端验证证书:                           │
    │       - 用 root.crt 中的 CA 公钥验证签名       │
    │       - 检查有效期                              │
    │       - 检查域名 (tls_insecure_set 跳过的步骤)  │
    │                                               │
    │←── 5. ServerHelloDone ─────────────────────│
    │    "我的部分完了"                               │
    │                                               │
    │    6. 客户端生成 pre_master_secret             │
    │       (48 字节随机数)                           │
    │                                               │
    │── 7. ClientKeyExchange ─────────────────→  │
    │    pre_master_secret 用服务器公钥加密后发出      │
    │    只有服务器私钥能解开                          │
    │                                               │
    │    8. 双方各自计算会话密钥:                     │
    │       master_secret = PRF(                     │
    │           pre_master_secret,                   │
    │           "master secret",                     │
    │           client_random + server_random        │
    │       )                                        │
    │       从 master_secret 派生:                   │
    │       - 客户端写密钥 (加密客户端→服务端的数据)    │
    │       - 服务端写密钥 (加密服务端→客户端的数据)    │
    │       - 客户端写 MAC 密钥 (防篡改)              │
    │       - 服务端写 MAC 密钥                       │
    │                                               │
    │── 9. ChangeCipherSpec ──────────────────→  │
    │    "之后所有消息都用协商好的密钥加密"            │
    │                                               │
    │── 10. Finished (加密的) ─────────────────→  │
    │    第一条用新密钥加密的消息                      │
    │                                               │
    │←── 11. ChangeCipherSpec ──────────────────│
    │←── 12. Finished (加密的) ─────────────────│
    │                                               │
    │===== 加密通道建立，后续通信对称加密 ========│
```

**三个随机数的作用**：client_random + server_random + pre_master_secret。如果只靠 pre_master_secret（客户端单方面生成），万一客户端随机数生成器有缺陷（嵌入式设备常见问题），密钥就是弱的。三个来源混合，只要任意一个来源是真随机的，最终密钥就是安全的。

### 2.4 TLS 记录层 — 加密后的数据帧结构

握手完成后，每条 MQTT 报文都被 TLS 记录层加密成这样的帧：

```
┌──────────────────────────────────────────────────────────┐
│ TLS Record Header (5B) │ Encrypted Data (MQTT 报文, 密文) │
│                         │                                │
│ Byte 0: 23             │  明文 MQTT PUBLISH 报文           │
│    (Application Data)  │  ↓ AES-256-CBC 加密              │
│ Byte 1-2: TLS 版本      │  ↓ 变成看不懂的乱码              │
│    (0x03 0x01 = TLSv1) │                                │
│ Byte 3-4: 加密数据长度   │                                │
└──────────────────────────────────────────────────────────┘
```

---

TLS 握手过程中，客户端收到服务器证书后必须"验证证书"。这个验证是怎么做的、信任是怎么建立的——这是下面要讲的证书体系。

---

## 第三章：证书与 CA 体系

### 3.1 X.509 证书里有什么

X.509 是证书的标准格式。一个证书文件包含：

```
Certificate:
    Version: 3
    Serial Number: 0A:1B:2C:...           ← CA 分配的唯一序列号
    Signature Algorithm: sha256WithRSAEncryption
    Issuer:                                ← 谁颁发的
        CN = Root CA
        O  = ChinaAMD
        C  = CN
    Validity:                              ← 有效期
        Not Before: 2020-01-01 00:00:00
        Not After:  2030-01-01 00:00:00
    Subject:                               ← 证书持有人
        CN = cloud.chinaamd.cn             ← 域名
        O  = ChinaAMD
    Subject Public Key Info:               ← 公钥
        RSA Public Key: (2048 bit)
        Modulus (n): 0x00:b3:8a:... (256 字节)
        Exponent (e): 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:                       ← CA 的签名
        0x4f:8a:1c:3d:... (256 字节)
```

**签名是怎么生成的**：

```
颁发者(CA)：
  1. 把证书中"除了 Signature Value 之外的所有字段"串起来
  2. 对这串数据计算 SHA256 → 哈希值
  3. 用 CA 自己的私钥对哈希值加密 → 这就是 Signature Value
  4. 把 Signature Value 填入证书
```

**验证时怎么做**：

```
客户端：
  1. 把证书中除了 Signature Value 之外的字段串起来
  2. 计算 SHA256 → 哈希值 H1
  3. 用 CA 的根证书中的公钥，对 Signature Value 解密 → 哈希值 H2
  4. 比较 H1 == H2？
     == → 签名有效，证书内容没被篡改，确实由该 CA 签发
     != → 证书被篡改过，或者不是该 CA 签发的
```

### 3.2 信任链

```
┌──────────────────────────────────────────────┐
│ root.crt (CA 根证书)                          │
│   Subject: CN = Root CA                      │
│   公钥: pub_root                              │
│   签名: 自签名（Root CA 用自己的私钥签自己）    │
│                                              │
│   ★ 根证书必须预置在设备上，它是信任的起点       │
│   ★ 因为根证书是自签名的，不能被外部验证         │
│   ★ 所以你用 tls_set("./root.crt") 把它加载进去 │
└──────────────┬───────────────────────────────┘
               │ CA 用自己的私钥签名
               ▼
┌──────────────────────────────────────────────┐
│ 中间 CA 证书（可选）                           │
│   Subject: CN = Intermediate CA              │
│   Issuer:  CN = Root CA                      │
│   公钥: pub_intermediate                      │
│   签名: 用 Root CA 的私钥签名                  │
└──────────────┬───────────────────────────────┘
               │ 中间 CA 用自己的私钥签名
               ▼
┌──────────────────────────────────────────────┐
│ 服务器证书                                     │
│   Subject: CN = cloud.chinaamd.cn            │
│   Issuer:  CN = Intermediate CA (或 Root CA) │
│   公钥: pub_server                            │
│   签名: 用中间 CA 的私钥签名                    │
└──────────────────────────────────────────────┘
```

### 3.3 本工程为什么 tls_insecure_set(true)

`tls_set("./root.crt")` 只做了**签名验证**——验证服务器证书确实是由 root.crt 对应的 CA 机构签发的。

但还有一步叫**域名验证**：检查证书上的 CN（Common Name）或 SAN（Subject Alternative Name）是否等于你实际连接的地址（`cloud.chinaamd.cn`）。

如果设备通过内网 IP `192.168.1.100` 连接到本地 Broker，而这个 Broker 用的证书上的 CN 是 `cloud.chinaamd.cn`——严格验证下会失败，因为 IP ≠ 域名。

`tls_insecure_set(true)` 跳过域名验证：接受这个证书，只要 CA 签名是对的。

---

知道证书怎么验证了，接下来看 OpenSSL 这个库和本工程代码中是怎么使用这些机制的。

---

## 第四章：OpenSSL — 谁在干活

### 4.1 两个 .so 的分工

```
libssl.so      → TLS 协议实现（握手、记录层、SSL_connect/SSL_read/SSL_write）
libcrypto.so   → 加密算法实现（AES、RSA、SHA256、大数运算、随机数生成）

libssl.so 依赖 libcrypto.so——TLS 协议调用加密算法
```

TLS 是一个协议规范（纸上的文档）。OpenSSL 把这个规范写成 60 万行 C 代码。其他实现：GnuTLS（GNU）、mbedTLS（ARM 嵌入式常用）、wolfSSL（物联网）。

### 4.2 核心概念：SSL_CTX 和 SSL

```
SSL_CTX *ctx = SSL_CTX_new(TLSv1_client_method());
// SSL_CTX 是"TLS 连接的模板"
// 包含：信任的 CA 证书列表、TLS 版本配置、证书验证级别
// 一个进程通常只需要一个 SSL_CTX，所有连接共享

SSL *ssl = SSL_new(ctx);
// SSL 是"一条具体的 TLS 连接"
// 绑定到一个 socket fd
// 包含：这条连接的会话密钥、读写缓冲区、握手状态

SSL_set_fd(ssl, sockfd);   // 把 SSL 和 TCP socket 绑定
SSL_connect(ssl);           // 执行 TLS 握手（发 ClientHello → 收 ServerHello → ...）
SSL_write(ssl, buf, len);  // 加密写入：buf → 加密 → TCP send
SSL_read(ssl, buf, len);   // 解密读取：TCP recv → 解密 → buf
```

mosquitto 的 `tls_set()` / `tls_opts_set()` / `tls_insecure_set()` 内部就是在操作这个 `SSL_CTX`，然后在 `connect()` 建立 TCP 连接后创建 `SSL` 对象并调 `SSL_connect()` 完成握手。

---

## 第五章：本工程 TLS 代码逐行拆解

### 5.1 三行核心代码

```cpp
// mqttsrv.cpp:93-95
test->tls_opts_set(1, "tlsv1", NULL);
test->tls_insecure_set(true);
rc = test->tls_set("./root.crt", NULL);
```

### 5.2 第一行：tls_opts_set — 设置 TLS 版本和证书验证级别

```cpp
// mosquittopp.h:78
int tls_opts_set(int cert_reqs, const char *tls_version, const char *ciphers);

// 本工程调用：
test->tls_opts_set(1, "tlsv1", NULL);
//                  ↑    ↑       ↑
//       cert_reqs=1    TLSv1   不指定加密套件，用默认

// cert_reqs 的取值：
//   0 = SSL_VERIFY_NONE  — 不验证服务器证书。相当于裸 TCP，TLS 形同虚设
//   1 = SSL_VERIFY_PEER  — 验证服务器证书。这是正确的做法
//
// tls_version 取值：
//   "tlsv1"  — TLS 1.0（本工程使用，较老，仍广泛用于嵌入式设备）
//   "tlsv1.1"
//   "tlsv1.2" — 当前主流
//   "tlsv1.3" — 最新，2018年
//
// 如果 tls_version 传 NULL，OpenSSL 用默认版本（取决于编译时的 OpenSSL 版本）
```

本工程用 TLSv1 是因为嵌入式 Linux 板上的 OpenSSL 版本较老（`openssl 1.0.1` 以下只支持 TLSv1），不支持 TSLv1.2。TLSv1 在理论上存在已知漏洞（如 BEAST 攻击），但在工业内网环境中实际风险极低。

### 5.3 第二行：tls_insecure_set — 跳过域名验证

```cpp
// mosquittopp.h:79
int tls_insecure_set(bool value);

// 本工程调用：
test->tls_insecure_set(true);
//                      ↑
//              true = 不验证域名 → 证书 CN 不必匹配连接地址
//              false = 严格验证域名 → 必须匹配

// 内部对应：
// OpenSSL 参数 SSL_VERIFY_PEER（来自 tls_opts_set）验证了 CA 签名有效性
// 域名验证是额外的步骤，OpenSSL 默认不自动做——需要应用层自己调用
// X509_check_host() 或用 SSL_set1_host() + SSL_set_verify(SSL_VERIFY_PEER)
//
// mosquitto 默认做了域名验证。tls_insecure_set(true) 关闭这个默认行为。
```

**为什么设为 true（不安全设置）**：

1. 工厂环境可能用内网 IP 连接 Broker（`192.168.1.100`），不是用域名
2. 如果用域名 `cloud.chinaamd.cn`，IP 直连时证书上的域名和连接地址不匹配
3. CA 密钥通常离线保管在安全硬件中，伪造证书需要在云端服务器侧配合，不是一个攻击者能单方面做到的

### 5.4 第三行：tls_set — 加载 CA 根证书

```cpp
// mosquittopp.h:77
int tls_set(const char *cafile,         // CA 根证书文件路径（PEM 格式）
            const char *capath=NULL,    // CA 证书目录（和 cafile 二选一）
            const char *certfile=NULL,  // 客户端证书文件（双向认证时用）
            const char *keyfile=NULL,   // 客户端私钥文件（双向认证时用）
            int (*pw_callback)(...)=NULL);  // 私钥的密码回调

// 本工程调用：
rc = test->tls_set("./root.crt", NULL, NULL, NULL, NULL);
//                  ↑              ↑     ↑     ↑     ↑
//            CA 证书文件路径    不用目录 不用客户端证书 不用私钥 不用密码

// ★ 所有后面的参数都是 NULL 意味着：
//    - 没有客户端证书 → 使用"单向认证"（只验证服务器身份，不验证客户端身份）
//    - 没有私钥 → 不需要密码
```

**单向认证 vs 双向认证**：

```
单向认证（本工程）：
  客户端验证服务器证书 → 确认服务器的身份
  服务器不验证客户端 → 任何能 TCP 连上的人都能建立 TLS 连接
  MQTT 层面再用 OAuth2 Token 验证客户端身份

双向认证：
  客户端验证服务器证书 + 服务器验证客户端证书 → 双方确认身份
  每个设备需要自己的客户端证书和私钥 → 证书管理复杂
  没有 OAuth2 的情况下这才是完全安全的，本工程用 OAuth2 替代了这一层
```

### 5.5 三行代码在 run() 中的调用时序

```cpp
// mqttsrv.cpp:54-175 — mqttThread::run()

void mqttThread::run()
{
    sleep(3);

    // ... ping 检查网络 ...

    mosqpp::lib_init();  // ★ 1. 初始化 mosquitto 库（调一次）

    // ★ 2. 配置 TLS（必须在 connect 之前调用）
    test->tls_opts_set(1, "tlsv1", NULL);    // 选 TLSv1 + 验证证书
    test->tls_insecure_set(true);             // 跳域名验证
    rc = test->tls_set("./root.crt", NULL);   // 加载 CA 根证书

    // ★ 3. 连接 Broker
    rc = test->connect("cloud.chinaamd.cn", 1884, 50);
    //   connect() 内部：
    //     socket() → TCP connect()
    //     → SSL_CTX_new() → SSL_new() → SSL_set_fd()
    //     → SSL_connect() → TLS 握手（验证证书在此完成）
    //     → 握手成功 → TCP 连接升级为 TLS 连接

    // ★ 4. 正常使用（loop 内部自动加密/解密）
    while (1) {
        rc = test->loop();
        //  内部：select() → recv() 收 TLS 帧 → OpenSSL 解密 → MQTT 报文 → on_message()
        //       on_message 中收到的是明文 JSON
        //       发送：publish(明文) → OpenSSL 加密 → send() 发 TLS 帧
        msleep(100);
    }
}
```

### 5.6 调试技巧：如何临时关闭 TLS

当你在本地搭建 mosquitto Broker 做测试时，TLS 是障碍。临时关闭方法：

```cpp
// mqttsrv.cpp:93-95 — 修改或注释掉 TLS 相关三行

// 方式1：注释掉 TLS 配置
// test->tls_opts_set(1, "tlsv1", NULL);
// test->tls_insecure_set(true);
// test->tls_set("./root.crt", NULL);
// test->connect("localhost", 1883, 50);  // 端口改回 1883

// 方式2：用 #if 0 包起来
#if 1
    test->tls_opts_set(1, "tlsv1", NULL);
    test->tls_insecure_set(true);
    test->tls_set("./root.crt", NULL);
    test->connect("cloud.chinaamd.cn", 1884, 50);
#else
    test->connect("localhost", 1883, 50);  // 不带 TLS 的本地测试
#endif
```

---

## 第六章：OAuth2 + Base64（补充：认证 ≠ 加密）

MQTT 用 TLS 做了加密，但服务器还需要知道"这条连接是谁"。TLS 只验证服务器身份，不验证客户端身份。OAuth2 填补了这一点。

### 6.1 Base64 — 编码，不是加密

```
加密：依赖密钥。第三方没有密钥无法解锁。
编码：没有密钥。任何人拿到编码后的数据都能解码——因为编码/解码规则是公开的。

"api:api" → Base64 编码 → "YXBpOmFwaQ=="
"YXBpOmFwaQ==" → Base64 解码 → "api:api"
             ↑ 不需要密钥，谁都能解
```

Base64 的算法：每 3 个字节（24位）拆成 4 个 6 位组，每个 6 位组对应一个可打印字符（A-Z, a-z, 0-9, +, /），末尾用 `=` 补齐。

用途：HTTP Basic Auth 的 `Authorization` 头——`用户名:密码` 用 Base64 编码后发出。不是安全措施，只是让特殊字符能安全通过 HTTP 头传输。

### 6.2 OAuth2 Client Credentials 流程

```
设备（工控机）                             云服务器（cloud.chinaamd.cn:8900）
    │                                             │
    │── POST /auth/mobile/token/device ───────→ │
    │   Host: cloud.chinaamd.cn:8900              │
    │   Authorization: Basic YXBpOmFwaQ==        │
    │     (= Base64("api:api"))                  │
    │   Content-Type: application/x-www-form-urlencoded │
    │   Body: mobile=DEVICE@ZKGDDEV47            │
    │                                             │
    │←── 200 OK ──────────────────────────────│
    │   {                                         │
    │     "access_token": "eyJhbGciOi...",       │
    │     "token_type": "bearer",                 │
    │     "expires_in": 604800                    │
    │   }                                         │
    │                                             │
    │   之后所有业务请求带这个 token：              │
    │── POST /api/upload ────────────────────→  │
    │   Authorization: Bearer eyJhbGciOi...      │
```

token 本身是一个加密的 JWT（JSON Web Token），里面编码了设备 ID 和过期时间。服务器收到后解密验证，确认"这个请求来自 ZKGDDEV47"。

本工程 token 续期逻辑（`mqttMsgParaseThread::run()`）：

```cpp
// mqttsrv.cpp:265-272
if (myHttpFileClient->g_token_expires < struGsh.nCounter - m_nextWeek) {
    m_nextWeek = struGsh.nCounter;
    myHttpFileClient->g_token_expires = 0;
    myHttpFileClient->requestTokenUpdate(URL_TOKEN_REQUEST);  // 重新申请 token
}
```

每周自动续期一次，确保设备长期在线时 token 不会过期。

---

## 附录：本工程加密相关文件索引

| 文件 | 行号 | 内容 |
|------|------|------|
| `zksort/bus/mqttsrv.cpp:93-95` | — | TLS 三行核心配置 |
| `zksort/bus/mqttsrv.cpp:54-175` | — | `mqttThread::run()` 完整 TLS + 连接流程 |
| `zksort/bus/mqttsrv.cpp:178-358` | — | `mqttMsgParaseThread::run()` OAuth2 Token 续期 |
| `zksort/bus/myhttpfileclient.cpp:177-267` | — | `requestTokenUpdate()` OAuth2 设备认证实现 |
| `zksort/bus/mosquitto.h` | 996-1026 | `mosquitto_tls_set()` / `tls_insecure_set()` C API 完整注释 |
| `CameraTest(aiuse)/3rdparty/mqtt/` | — | 预编译 mosquitto .so 文件 |
