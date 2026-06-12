# Qt 核心类型

本工程中使用频率最高的 Qt 基础类型和类。Qt 是 C++ 框架，这些是对 C++ 标准库的补充或封装。

---

## QByteArray

**头文件**：`<QByteArray>`

**是什么**：Qt 的字节数组。可以理解为一个**自动管理内存的 `unsigned char[]` 数组**，不需要手动 `malloc/free`。

**为什么用它**：UDP 通信中传输的是二进制数据（字节流），`QByteArray` 是 Qt 中最适合存放原始字节的容器。

**本工程中的典型用法**：

```cpp
// 1. 创建并指定大小
QByteArray datagram;
datagram.resize(len);             // 分配 len 个字节

// 2. 用 memcpy 往里面填数据
QByteArray send_buf;
send_buf.resize(headSize + dataLen);
memcpy(send_buf.data(), &header, headSize);  // .data() 返回内部 char* 指针

// 3. 传给 UDP socket 发送
udpSocket->writeDatagram(send_buf, QHostAddress(ip), port);

// 4. 读取其中的字节
if (datagram.at(0) != 0xa5)      // .at(0) 读第 0 个字节
    continue;

// 5. 转为十六进制字符串（调试用）
QString hex = datagram.toHex();
```

**关键方法**：
| 方法 | 作用 |
|------|------|
| `resize(n)` | 分配 n 字节空间 |
| `data()` | 返回内部数据的 `char*` 指针，用于 `memcpy` |
| `at(i)` | 读取第 i 个字节（返回 `char`） |
| `size()` | 返回字节数 |
| `toHex()` | 转为十六进制字符串（如 "A55A00C0..."） |
| `prepend(str)` | 在头部插入数据 |

---

## QMap<Key, Value>

**头文件**：`<QMap>`

**是什么**：Qt 的键值对容器（类似 Python 的 dict）。通过 key 查找 value，查找速度是 O(log n)。

**为什么用它**：把 board 地址（`board_h`）映射为 view 编号。收到 UDP 包时从包头取出 `board_h`，再反向查出 view。

**本工程中的典型用法**：

```cpp
// 定义与初始化（myudpthread.cpp:26-33）
QMap<int, int> m_viewIndexMap;
m_viewIndexMap.insert(0, 0x02);   // view 0 的 board 地址是 0x02
m_viewIndexMap.insert(1, 0x04);   // view 1 的 board 地址是 0x04
m_viewIndexMap.insert(2, 0x03);
// ...

// 正向查找：view → board（插入时的顺序）
// board = m_viewIndexMap.value(view);  // 或 m_viewIndexMap[view]

// 反向查找：board → view（processTheDatagram 中用到）
view = m_viewIndexMap.key(head.board_h);  // 0x02 → 0, 0x04 → 1
```

| 方法 | 作用 |
|------|------|
| `insert(key, value)` | 插入键值对 |
| `value(key)` / `operator[](key)` | 通过 key 查 value |
| `key(value)` | 反向查找：通过 value 查第一个匹配的 key |

---

## QMutex + QMutexLocker

**头文件**：`<QMutex>`, `<QMutexLocker>`

**是什么**：互斥锁。保证同一时刻只有一个线程访问某块代码/数据。

**为什么用它**：多个线程可能同时读写同一个变量（如 `m_count`），不加锁会导致数据错乱。

**本工程中的典型用法**：

```cpp
// 声明锁
QMutex m_mutex;

// 使用时加锁
{
    QMutexLocker lock(&m_mutex);   // 构造时自动 lock()
    // ... 临界区代码（同一时刻只有一个线程能执行到这里）...
}   // lock 离开作用域析构，自动 unlock()

// 与手动 lock/unlock 的对比：
m_mutex.lock();
m_count++;
m_mutex.unlock();   // 如果中间 return 了，锁永远不释放 → 死锁
// 而 QMutexLocker 离开作用域自动释放，不会忘记。
```

---

## QString

**头文件**：`<QString>`

**是什么**：Qt 的字符串类。比 C 的 `char[]` 强大得多——自动管理内存、支持 Unicode、丰富的格式化方法。

**本工程中的典型用法**：

```cpp
// 拼接字符串
QString serverIp = QString("192.168.1.%1").arg(nIpAddr[4]);  // %1 被替换为 nIpAddr[4]

// 数字转字符串
QString::number(100)  → "100"

// 格式化浮点数
QString::number(3.14159, 'f', 1)  → "3.1"

// 取子串
ipStr0.right(3)   // 取右边 3 个字符
ipStr0.mid(2)     // 从第 2 个字符开始取到最后
ipStr0.left(2)    // 取左边 2 个字符

// 转 C 字符串
qPrintable(qstr)  // 用于 printf/qDebug 输出
qstr.toLatin1().constData()  // 转为 const char*
```

---

## QThread

**头文件**：`<QThread>`

**是什么**：Qt 的线程基类。**继承它，重写 `run()`，用 `start()` 启动。**

**本工程中的典型用法**：

```cpp
// 定义线程类
class WorkerThread : public QThread
{
    Q_OBJECT
protected:
    void run() { /* 线程要干的活 */ }
};

// 使用
WorkerThread *t = new WorkerThread;
t->start();        // 创建系统线程 → 调用 run()（非阻塞，立即返回）
// 主线程继续做自己的事...

// 停止
t->quit();         // 让 exec() 退出（仅对有事件循环的线程有效）
t->wait();         // 阻塞等待，直到线程死透
```

| 方法 | 作用 |
|------|------|
| `start()` | 创建系统线程 → 调用 `run()`。非阻塞 |
| `run()` | 纯虚函数，**重写它**，线程的入口 |
| `exec()` | 进入 Qt 事件循环（在 `run()` 中调用） |
| `quit()` | 让 `exec()` 退出 |
| `wait()` | 阻塞等待线程结束 |
| `msleep(n)` | 当前线程休眠 n 毫秒 |

---

## QTimer

**头文件**：`<QTimer>`

**是什么**：定时器。隔一段时间发射 `timeout()` 信号。

**本工程中的典型用法**：

```cpp
// 主线程中——每秒刷新界面
QTimer *timer = new QTimer(this);
connect(timer, SIGNAL(timeout()), this, SLOT(updateDisplay()));
timer->start(1000);   // 每 1000ms 触发一次 timeout()

// 子线程中（需要事件循环支持）
QTimer *timer = new QTimer();   // 注意：不能传 this（run() 不是 QObject）
connect(timer, SIGNAL(timeout()), this, SLOT(onTick()));
timer->start(1000);
```

---

## QObject 和 Q_OBJECT 宏

**头文件**：`<QObject>`

**是什么**：几乎所有 Qt 类的基类。提供信号/槽、事件处理、对象树内存管理。

**本工程中**：所有界面控件（QWidget）、全局单例（MyUdp）、线程（QThread）都继承自 QObject。

`Q_OBJECT` 宏必须写在类定义的最开头。它让 moc（Meta-Object Compiler）生成信号/槽所需的元代码。**忘记写这个宏 → 信号/槽不工作 → 编译报错。**
