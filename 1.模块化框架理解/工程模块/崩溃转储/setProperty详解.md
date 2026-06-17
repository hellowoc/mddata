# `qtlogProper.setProperty` 详解

## 调用处

```cpp
qtlogProper.setProperty("log4j.rootLogger", "DEBUG, A1");
```

`qtlogProper` 是 `Log4Qt::Properties` 类型的局部变量。

---

## Properties 是什么

```cpp
// 3rdparty/log4qt/include/helpers/properties.h:50
class LOG4QT_EXPORT Properties : public QHash<QString, QString>
{
public:
    inline void setProperty(const QString &rKey, const QString &rValue)
    {   insert(rKey, rValue);   }
};
```

```
Properties
  └── QHash<QString, QString>      ← Qt 的哈希表模板类
        └── 内部是哈希数组（bucket array）
```

**`Properties` 本质上就是一个 `QHash<QString, QString>`**。它没有增加任何数据成员，只增加了一些辅助方法。`setProperty` 等价于 `QHash::insert()`。

---

## `QHash::insert(key, value)` 做了什么

```cpp
// Qt 源码中 QHash::insert 的核心逻辑（简化版）

void QHash<Key, T>::insert(const Key &key, const T &value)
{
    // 1. 对 key 做哈希计算，找到对应的桶
    uint hash = qHash(key);          // "log4j.rootLogger" → 一个 uint
    int bucket = hash % numBuckets;   // 取模落到具体桶

    // 2. 在桶的链表中查找是否已存在相同的 key
    Node *node = findNode(bucket, key);
    if (node) {
        node->value = value;          // key 已存在 → 替换 value
    } else {
        // key 不存在 → 创建新节点，插入链表
        Node *newNode = new Node(key, value);
        newNode->next = buckets[bucket];
        buckets[bucket] = newNode;
        ++size;

        if (size > numBuckets * loadFactor)
            rehash();                 // 负载过高 → 扩容
    }
}
```

所以 `setProperty("log4j.rootLogger", "DEBUG, A1")` 就是往哈希表里塞了一个键值对。

---

## 该项目塞了哪些键值对

```
qtlogProper 哈希表内容:

┌──────────────────────────────────────────────┬──────────────────────────────────────┐
│ key                                          │ value                                │
├──────────────────────────────────────────────┼──────────────────────────────────────┤
│ "log4j.rootLogger"                           │ "DEBUG, A1"                          │
│ "log4j.appender.A1"                          │ "org.apache.log4j.DailyRollingFile.."│
│ "log4j.appender.A1.appendFile"               │ "true"                               │
│ "log4j.appender.A1.immediateFlush"           │ "true"                               │
│ "log4j.appender.A1.datePattern"              │ "'.'yyyy-MM-dd"                      │
│ "log4j.appender.A1.file"                     │ "/sdcard/logs/log.txt"               │
│ "log4j.appender.A1.layout"                   │ "org.apache.log4j.PatternLayout"     │
│ "log4j.appender.A1.layout.ConversionPattern" │ "%d [%t] %p %c%x - %m%n"            │
│ "log4j.appender.A1.layout.header"            │ "程序启动"                           │
│ "log4j.appender.A1.layout.footer"            │ "程序结束"                           │
└──────────────────────────────────────────────┴──────────────────────────────────────┘
```

---

## 这些键值对后来怎么被消费

这些键值对指向了一套**层次化配置体系**，靠 key 的命名约定来表达：

```
log4j.rootLogger          = 根 Logger 的级别和关联 Appender
log4j.appender.A1         = 定义一个名叫 A1 的 Appender（= 它的类型）
log4j.appender.A1.xxx     = A1 的配置属性（文件、刷新模式等）
log4j.appender.A1.layout  = A1 用的 Layout 类型
log4j.appender.A1.layout.xxx = Layout 的配置属性（格式串）
```

`PropertyConfigurator::configure()` 解析时按前缀匹配：

```cpp
// 解析时的伪代码
void configure(rootLogger) {
    // 读 "log4j.rootLogger" → "DEBUG, A1"
    // → 设置 rootLogger 级别为 DEBUG
    // → 去找 "log4j.appender.A1" 开头的配置

    // 读 "log4j.appender.A1" → "org.apache.log4j.DailyRollingFileAppender"
    // → 反射/工厂模式创建 DailyRollingFileAppender 实例

    // 读所有 "log4j.appender.A1.*" 项
    // → "appendFile" → 调用 appender->setAppendFile(true)
    // → "immediateFlush" → 调用 appender->setImmediateFlush(true)
    // → "file" → 调用 appender->setFile("/sdcard/logs/log.txt")

    // 读 "log4j.appender.A1.layout" → 创建 PatternLayout 实例

    // 读 "log4j.appender.A1.layout.ConversionPattern" → "%d [%t] %p..."
    // → 调用 layout->setConversionPattern("%d [%t] %p...")
}
```

---

## 为什么要用 Properties + setProperty 而不是写配置文件

**原因一：文件路径是运行时决定的。** 日志文件路径在不同平台（Windows/Linux）不同，在代码中动态计算后通过 setProperty 写入，比静态配置文件灵活。

**原因二：嵌入式设备上少一个文件更可靠。** 没有 `log4j.properties` 文件就少一个文件要管理、不会被误删、少一次文件 I/O。

**原因三：等价于 Java 的 log4j.properties。** 命名规则、层级关系、解析逻辑完全一致。Java 下的配置：

```properties
log4j.rootLogger=DEBUG, A1
log4j.appender.A1=org.apache.log4j.DailyRollingFileAppender
```

和 C++ 代码：

```cpp
qtlogProper.setProperty("log4j.rootLogger", "DEBUG, A1");
qtlogProper.setProperty("log4j.appender.A1", "org.apache.log4j.DailyRollingFileAppender");
```

语义完全等价。

---

## 小结

```
qtlogProper.setProperty("key", "value")
    │
    └── Properties::setProperty()              // 内联函数
          └── QHash::insert(key, value)         // 哈希表插入
                └── qHash(key) → 桶 → Node

// 配置阶段结束后：
PropertyConfigurator::configure(qtlogProper)
    └── 遍历哈希表
          ├── 找 "log4j.rootLogger"             → 配置根 Logger
          ├── 找 "log4j.appender.A1"            → 创建 Appender
          ├── 找 "log4j.appender.A1.*"           → 设置属性
          └── 找 "log4j.appender.A1.layout.*"    → 创建并配置 Layout
```

`setProperty` 只是一个哈希表 insert 操作，意义不在本身，而在它参与构建的 **"key 命名约定 → 配置对象"** 的映射体系。
