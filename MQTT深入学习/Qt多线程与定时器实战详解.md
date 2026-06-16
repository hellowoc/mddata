# Qt 多线程与定时器实战详解

Qt 4.8 环境。文档主线：从"为什么需要多线程"出发 → QThread 三种创建方式 → 线程间通信与安全 → QTimer 三种创建方式 → 超时控制实战 → 周期任务实战 → 本工程用法速查。

---

## 第一部分：QThread 多线程

### 主线引入

主线程跑 GUI 事件循环（`QApplication::exec()`）。如果主线程做耗时操作——读大文件、等网络数据、死循环收包——UI 就冻住了。多线程把耗时操作挪到独立线程，主线程只负责 UI 响应。

但 QThread 不是一个"线程类"——它是一个**线程管理器**，用来管理一个操作系统线程。真正的线程函数在 `run()` 里。理解这个区别是理解 QThread 的关键。

---

## 第一章：方式 A — 重写 run()，死循环模式

### 1.1 用法模板

```cpp
// === 头文件 ===
class MyWorkerThread : public QThread
{
    Q_OBJECT
public:
    explicit MyWorkerThread(QObject *parent = nullptr);
    void stop();                          // 外部调用来停止线程

protected:
    void run() override;                  // ★ 线程函数

private:
    volatile bool m_isRunning;           // volatile 防止编译器优化掉

signals:
    void dataReady(QByteArray data);     // 数据准备好了，通知主线程
    void errorOccurred(QString msg);
};

// === 实现 ===
MyWorkerThread::MyWorkerThread(QObject *parent)
    : QThread(parent), m_isRunning(false)
{
}

void MyWorkerThread::run()
{
    m_isRunning = true;

    while (m_isRunning) {
        // 做你的事：收数据、处理、轮询...
        QByteArray data = doSomeWork();

        if (!data.isEmpty()) {
            emit dataReady(data);  // 通知主线程（跨线程，Qt 自动排队）
        }

        msleep(100);  // 短暂休眠，让出 CPU
    }
}

void MyWorkerThread::stop()
{
    m_isRunning = false;   // 请求退出

    if (!wait(5000)) {     // 等最多 5 秒
        terminate();        // 超时强制终止（最后手段）
        wait();
    }
}

// === 主线程使用 ===
MyWorkerThread *worker = new MyWorkerThread;
connect(worker, SIGNAL(dataReady(QByteArray)), this, SLOT(onData(QByteArray)));
connect(worker, SIGNAL(finished()), worker, SLOT(deleteLater()));  // 线程结束自动 delete
worker->start();  // ★ 启动线程 → pthread_create() → run()
```

### 1.2 内部调用链

```
主线程调 start()
  → QThread::start()
    → pthread_create(&thread, NULL, QThreadPrivate::start, this)
      → 新线程执行 QThreadPrivate::start(this)
        → this->run()  ← ★ 你的 run() 在这里执行
```

`pthread_create()` 是 POSIX 标准函数，Linux 内核根据它创建一个新线程（新任务结构体、新栈空间、加入调度器）。

### 1.3 退出机制

| 方法 | 含义 | 适用 |
|------|------|------|
| `m_isRunning = false` | 让 `while(m_isRunning)` 循环退出 | 死循环模式 |
| `wait(timeout)` | 阻塞等待线程结束（pthread_join），可设超时 | 两种模式都需要 |
| `quit()` | 退出事件循环（发送 QEvent::Quit） | 仅 exec() 模式 |
| `terminate()` | 强制杀死线程（pthread_cancel），不推荐 | 最后手段 |

### 1.4 本工程例子

```cpp
// mqttsrv.cpp:54 — mqttThread
void mqttThread::run()
{
    // ... TLS 连接 + subscribe ...

    while (1) {                    // 死循环
        rc = test->loop();         // mosquitto 网络循环
        if (rc) {                  // 出错 → 重连
            test->reconnect();
            test->subscribe(NULL, topic);
        }
        msleep(100);
    }
}

// mynetwork.cpp:1089 — MyNetWorkThread
void MyNetWorkThread::run()
{
    while (isrunning) {            // volatile 标志位控制退出
        switch (currenttype) {
        case CAPTURE_SINGLE_WAVE:
            // select + recvfrom 收波形...
            break;
        }
    }
}
```

---

## 第二章：方式 B — 重写 run()，exec() 事件循环模式

### 2.1 为什么需要事件循环

死循环模式下，子线程不能使用 QTimer、不能用 Qt 的信号自动排队机制（没有事件循环来分发信号）。如果子线程需要接收来自其他线程的信号，或者需要定时器，就必须跑事件循环。

### 2.2 用法模板

```cpp
class MyEventThread : public QThread
{
    Q_OBJECT
public:
    explicit MyEventThread(QObject *parent = nullptr);
    void stop();

protected:
    void run() override;

signals:
    void workFinished(QString result);
};

void MyEventThread::run()
{
    // 1. 初始化（在子线程中创建 QObject、QUdpSocket 等）
    m_socket = new QUdpSocket;
    m_socket->bind(QHostAddress::Any, 12345);
    connect(m_socket, SIGNAL(readyRead()), this, SLOT(onData()));

    // 2. 启动事件循环
    exec();  // ★ 进入事件循环，在此阻塞。事件循环分发信号、定时器、socket 通知
    //    直到 quit() 被调用才返回
}

void MyEventThread::stop()
{
    quit();    // 退出 exec()
    wait();    // 等线程结束
}
```

### 2.3 事件循环内部做了什么

```
exec()
  → QEventLoop::exec()
    → while (!d->exit) {
        processEvents();         // 处理积压事件
        │  ├── 检查 socket fd（通过 select/poll）→ 可读 → 发射 readyRead
        │  ├── 检查定时器是否到期 → 发射 timeout()
        │  ├── 检查信号队列 → 执行排队的槽函数
        │  └── 检查 QEvent::Quit → 退出循环
        │
        if (没有事件要处理)
            poll(..., timeout);  // 休眠等事件，有超时（最近的定时器到期时间）
      }
```

### 2.4 本工程例子

```cpp
// myudpthread.cpp — SendThread
void SendThread::run()
{
    // 创建 BackgroudObj（QUdpSocket + readyRead 信号连接）
    m_MySocket = new BackgroudObj(this, 8082, this);
    // BackgroudObj 内部：udpSocket->bind(...)
    //   connect(udpSocket, SIGNAL(readyRead()), ...)

    // 创建 FastNetObj（另一个 QUdpSocket）
    m_MySocket_2 = new FastNetObj(this, 20000, this);

    exec();  // ★ 进入事件循环
    // QUdpSocket 的 readyRead 信号在这个事件循环中被分发
    // 外部 pushUdpCmdData() 入队的数据在这里被发送
}
```

### 2.5 两种 run() 模式对比

| | 死循环模式 | exec() 模式 |
|------|---------|----------|
| 核心语句 | `while(isrunning)` | `exec()` |
| 适合场景 | 持续高速收数据 | 偶尔有命令需要收发 |
| QTimer 可用？ | 不可用（需自己用 select 超时替代） | 可用 |
| 信号槽排队 | emit 的槽如果需要事件循环分发则不行 | 完全支持 |
| 退出方式 | `isrunning = false` | `quit()` |
| 本工程 | MyNetWorkThread, mqttThread | SendThread |

---

## 第三章：方式 C — moveToThread()

### 3.1 和重写 run() 的本质区别

重写 run()：QThread 的子类本身就是"线程要做的事"。
moveToThread()：把任意 QObject 移到一个独立的 QThread 中——QThread 只是一个空的事件循环容器，业务逻辑完全在 QObject 里。

### 3.2 用法模板

```cpp
// === 工作类（纯粹的 QObject，不继承 QThread） ===
class Worker : public QObject
{
    Q_OBJECT
public:
    explicit Worker(QObject *parent = nullptr);

public slots:
    void doWork();              // 将在子线程中执行
    void startMonitoring();     // 启动循环监控

signals:
    void progressUpdated(int percent);
    void workFinished(QString result);

private:
    QTimer *m_timer;            // 这个定时器在子线程中
};

Worker::Worker(QObject *parent) : QObject(parent)
{
    m_timer = new QTimer(this);
    connect(m_timer, SIGNAL(timeout()), this, SLOT(startMonitoring()));
}

void Worker::doWork()
{
    // 这段代码在子线程中执行
    for (int i = 0; i < 100; i++) {
        // 耗时操作...
        emit progressUpdated(i + 1);
    }
    emit workFinished("完成");
}

void Worker::startMonitoring()
{
    // 每 2 秒在子线程中触发一次
}

// === 主线程使用 ===
QThread *thread = new QThread;
Worker *worker = new Worker;               // 不设 parent！
worker->moveToThread(thread);              // ★ 把 worker 移到子线程

// worker 的信号 → 主线程更新 UI
connect(worker, SIGNAL(progressUpdated(int)), this, SLOT(onProgress(int)));
connect(worker, SIGNAL(workFinished(QString)), this, SLOT(onFinished(QString)));

// 主线程的信号 → worker 的槽在子线程执行
connect(this, SIGNAL(startWork()), worker, SLOT(doWork()));

// 线程结束时清理
connect(thread, SIGNAL(finished()), worker, SLOT(deleteLater()));
connect(thread, SIGNAL(finished()), thread, SLOT(deleteLater()));

thread->start();  // ★ 启动子线程的事件循环
emit startWork(); // worker.doWork() 将在子线程执行

// 退出：
thread->quit();
thread->wait();
```

### 3.3 关键注意点

1. **Worker 不能设 parent**：`moveToThread()` 要求对象没有 parent，否则 Qt 报错
2. **槽函数在哪个线程执行**：连接方式为 `AutoConnection`（默认）时，emit 线程 ≠ worker 所在线程 → 槽在 worker 所在线程执行
3. **QTimer 在 moveToThread 后的行为**：如果在 `moveToThread()` 之前创建了 QTimer，之后 `moveToThread()`，定时器在子线程触发
4. **deleteLater()**：`thread->finished` → `worker->deleteLater()` → `thread->deleteLater()`，否则内存泄漏

### 3.4 三种方式的应用场景对比

| 方式 | 适用场景 | 何时不适用 |
|------|---------|-----------|
| 重写 run() 死循环 | 持续高速 I/O（收 UDP 包、串口数据） | 需要用 QTimer 或复杂信号分发 |
| 重写 run() + exec() | 偶尔收发命令，需要 QTimer | 高频数据（事件循环开销大） |
| moveToThread() | 复用已有 QObject，希望线程和业务逻辑分离 | 简单一次性任务（杀鸡用牛刀） |

---

## 第四章：QtConcurrent 与 QThreadPool

### 4.1 QtConcurrent::run() — 零管理的一次性后台任务

```cpp
#include <QtConcurrentRun>

// 把一个函数扔到全局线程池执行
QFuture<QString> future = QtConcurrent::run(longRunningFunction, arg1, arg2);

// 监控完成
QFutureWatcher<QString> *watcher = new QFutureWatcher<QString>(this);
connect(watcher, SIGNAL(finished()), this, SLOT(onTaskDone()));
watcher->setFuture(future);

// 取结果
void onTaskDone() {
    QString result = watcher->future().result();  // 阻塞取结果（此时任务已完成）
}
```

### 4.2 QThreadPool + QRunnable — 控制并发数

```cpp
class MyTask : public QRunnable
{
public:
    void run() override {
        // 在线程池的某个线程中执行
        // 注意：QRunnable 不是 QObject，不能发信号
    }
};

// 使用
QThreadPool::globalInstance()->setMaxThreadCount(4);  // 最多 4 个并发线程
MyTask *task = new MyTask;
task->setAutoDelete(true);  // run() 结束后自动 delete
QThreadPool::globalInstance()->start(task);
```

### 适用场景判断

```
问自己三个问题：
1. 这个任务只需要执行一次？               → QtConcurrent::run()
2. 有大量小任务需要排队执行、控制并发？    → QThreadPool + QRunnable
3. 需要持续运行、需要信号槽/定时器？       → QThread（前面三种方式）
```

---

## 第五章：线程间通信与线程安全

### 5.1 信号槽跨线程 — 四种连接方式[[connect信号槽]]

```cpp
// Qt::AutoConnection（默认）
connect(sender, SIGNAL(sig()), receiver, SLOT(slt()));
// 行为：
//   emit 线程 == receiver 所在线程 → DirectConnection（直接调用槽）
//   emit 线程 != receiver 所在线程 → QueuedConnection（排队到事件循环）

// Qt::DirectConnection — 强制在 emit 线程直接调用
connect(sender, SIGNAL(sig()), receiver, SLOT(slt()), Qt::DirectConnection);
// ★ 注意：如果 receiver 在 GUI 线程，emit 在子线程 → 槽函数在子线程执行 → 不能操作 UI

// Qt::QueuedConnection — 强制排队
connect(sender, SIGNAL(sig()), receiver, SLOT(slt()), Qt::QueuedConnection);

// Qt::BlockingQueuedConnection — 排队 + emit 线程等槽执行完
connect(sender, SIGNAL(sig()), receiver, SLOT(slt()), Qt::BlockingQueuedConnection);
// ★ 注意：receiver 不能在 emit 线程中，否则死锁。emit 线程会阻塞直到槽执行完
```

### 5.2 QMutex / QMutexLocker — 互斥锁

```cpp
QMutex m_lock;

// 手动加解锁（不推荐 — 容易忘记 unlock）
m_lock.lock();
// ... 临界区代码 ...
m_lock.unlock();

// RAII 风格（推荐）
{
    QMutexLocker locker(&m_lock);  // 构造时 lock()
    // ... 临界区代码 ...
}  // 离开作用域，locker 析构 → unlock()
```

`QMutexLocker` 的 RAII 保证即使临界区抛异常、return、break，锁一定会释放。

### 5.3 QReadWriteLock — 读写锁

```cpp
QReadWriteLock rwLock;

// 读操作（允许多个线程同时读）
{
    QReadLocker locker(&rwLock);
    int value = sharedData;  // 只读
}

// 写操作（独占）
{
    QWriteLocker locker(&rwLock);
    sharedData = newValue;   // 写操作，独占访问
}
```

### 5.4 QWaitCondition — 条件变量

```cpp
QMutex mutex;
QWaitCondition cond;
bool dataReady = false;

// 线程 A（生产者）
{
    QMutexLocker locker(&mutex);
    data = produceData();
    dataReady = true;
    cond.wakeOne();          // 唤醒一个等待的线程
}

// 线程 B（消费者）
{
    QMutexLocker locker(&mutex);
    while (!dataReady) {     // ★ 用 while 不是 if（防止虚假唤醒）
        cond.wait(&mutex);   // 释放 mutex 并挂起，被唤醒后重新获取 mutex
    }
    consumeData(data);
}
```

### 5.5 QAtomicInt — 无锁原子操作

```cpp
QAtomicInt counter;

counter.ref();        // 原子自增，返回新值
counter.deref();      // 原子自减，返回新值
counter.fetchAndAddAcquire(5);  // 原子加 5

// 适用场景：简单的计数器、标志位，不需要互斥锁的开销
```

### 5.6 本工程的安全模式

```cpp
// 模式1：QMutex 保护共享队列
// mqttsrv.h — recvMsgList
QList<string> recvMsgList;
QMutex m_paraseLock;

// mqttThread（生产者）：
void on_message(...) {
    QMutexLocker lock(&m_paraseLock);
    recvMsgList.append(strRcv);
}

// mqttMsgParaseThread（消费者）：
{
    QMutexLocker lock(&m_paraseLock);
    msg = recvMsgList[0];
    recvMsgList.removeAt(0);
}

// 模式2：全局结构体读写分离（无锁）
// 子线程写 struGsh，主线程定时器读 — 时间差可接受，不加锁
```

---

## 第六章：QTimer 定时器

### 6.1 方式 A：QTimer 对象 — 最常用

```cpp
// === 循环定时器 ===
QTimer *timer = new QTimer(this);
connect(timer, SIGNAL(timeout()), this, SLOT(onTick()));
timer->start(1000);   // 每 1000ms 触发一次
// timer->stop();     // 停止

// === 单次定时器 ===
QTimer *onceTimer = new QTimer(this);
onceTimer->setSingleShot(true);
connect(onceTimer, SIGNAL(timeout()), this, SLOT(onTimeout()));
onceTimer->start(5000);  // 5 秒后触发一次，然后自动停止

// === 查询状态 ===
if (timer->isActive()) { /* 正在运行 */ }
timer->setInterval(2000);  // 修改间隔
```

### 6.2 方式 B：QTimer::singleShot() — 一次性延迟

```cpp
// 不需要创建 QTimer 对象
QTimer::singleShot(3000, this, SLOT(onDelayedAction()));
// 3 秒后 onDelayedAction() 执行，执行一次

// 配合 lambda（Qt5+，本工程 Qt4.8 不可用，仅作参考）
// QTimer::singleShot(3000, []() { qDebug() << "3 seconds passed"; });

// 适用场景：
// - 延迟执行（如启动后 3 秒检查网络状态）
// - 超时控制（发起操作 + singleShot 设定超时时间）
```

### 6.3 方式 C：QObject::startTimer() — 底层 API

```cpp
class MyClass : public QObject
{
    Q_OBJECT
protected:
    void timerEvent(QTimerEvent *event) override;  // ★ 重写这个

private:
    int m_timerId1;
    int m_timerId2;
};

void MyClass::startTimers()
{
    m_timerId1 = startTimer(1000);   // 每 1000ms 触发 timerEvent
    m_timerId2 = startTimer(5000);   // 每 5000ms 触发 timerEvent
}

void MyClass::timerEvent(QTimerEvent *event)
{
    if (event->timerId() == m_timerId1) {
        // 定时器 1 到期
    } else if (event->timerId() == m_timerId2) {
        // 定时器 2 到期
    }
}

void MyClass::stopTimers()
{
    killTimer(m_timerId1);
    killTimer(m_timerId2);
}
```

QTimer 底层就是用 `startTimer()` + `timerEvent()` 实现的——QTimer 收到 `QTimerEvent` 后发射 `timeout()` 信号。

### 6.4 QTimer 和线程的关系 — 最容易踩的坑

```
QTimer 依赖事件循环。在哪个线程创建 QTimer，就必须在那个线程有事件循环。

主线程：QApplication::exec() 提供了事件循环 → QTimer 直接可用
子线程（exec() 模式）：exec() 提供事件循环 → QTimer 可用
子线程（moveToThread）：QThread 内部有 exec() → QTimer 可用
子线程（重写 run() 死循环模式）：没有事件循环 → QTimer 不会触发！
```

---

## 第七章：实战场景 — 三种最常见需求

### 7.1 场景 A：主线程做事一半，开子线程处理别的

```cpp
// 需求：用户点击按钮后，主线程开始一段计算，同时开子线程去网络请求

// === MainWindow.h ===
class MainWindow : public QMainWindow
{
    Q_OBJECT
public:
    explicit MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

private slots:
    void onStartButtonClicked();
    void onRemoteResultReady(QString result);
    void onRemoteError(QString msg);

private:
    QThread *m_remoteThread;
    RemoteWorker *m_remoteWorker;
};

// === MainWindow.cpp ===
MainWindow::MainWindow(QWidget *parent) : QMainWindow(parent)
{
    m_remoteThread = new QThread(this);
    m_remoteWorker = new RemoteWorker;   // 不设 parent
    m_remoteWorker->moveToThread(m_remoteThread);

    connect(m_remoteWorker, SIGNAL(resultReady(QString)), 
            this, SLOT(onRemoteResultReady(QString)));
    connect(m_remoteWorker, SIGNAL(errorOccurred(QString)),
            this, SLOT(onRemoteError(QString)));
    connect(m_remoteThread, SIGNAL(finished()), m_remoteWorker, SLOT(deleteLater()));

    m_remoteThread->start();  // 子线程事件循环启动，等待任务
}

void MainWindow::onStartButtonClicked()
{
    // 主线程中
    label->setText("处理中...");

    // 主线程继续做自己的事
    doLocalCalculation_part1();

    // ★ 同时让子线程去干活（发信号给 worker，槽在子线程执行）
    QMetaObject::invokeMethod(m_remoteWorker, "doRemoteRequest",
                              Qt::QueuedConnection,
                              Q_ARG(QString, "http://192.168.1.100/api/data"));

    // 主线程继续（不阻塞）
    doLocalCalculation_part2();
}

void MainWindow::onRemoteResultReady(QString result)
{
    // ★ 这个槽在主线程执行（worker emit → AutoConnection → 主线程事件循环分发）
    label->setText("远程结果：" + result);
}

// === RemoteWorker.h ===
class RemoteWorker : public QObject
{
    Q_OBJECT
public:
    explicit RemoteWorker(QObject *parent = nullptr);

public slots:
    void doRemoteRequest(QString url);   // 在子线程执行

signals:
    void resultReady(QString result);
    void errorOccurred(QString msg);
};
```

### 7.2 场景 B：超时控制 — 操作必须在 N 秒内完成

```cpp
// 方案 1：QTimer::singleShot（适合异步操作）

class TimeoutController : public QObject
{
    Q_OBJECT
public:
    void startOperationWithTimeout(int timeoutMs)
    {
        m_timedOut = false;

        // 设超时定时器
        QTimer::singleShot(timeoutMs, this, SLOT(onTimeout()));

        // 发起操作
        startAsyncOperation();
    }

private slots:
    void onTimeout()
    {
        m_timedOut = true;
        emit errorOccurred("操作超时");
        cancelOperation();
    }

    void onOperationFinished(QString result)
    {
        if (!m_timedOut) {
            emit success(result);
        }
    }

private:
    bool m_timedOut;
};

// ────────────────────────────────────────────

// 方案 2：QElapsedTimer（适合同步循环中的超时检测）

#include <QElapsedTimer>

bool receiveWithTimeout(int sockfd, char *buf, int totalSize, int timeoutMs)
{
    QElapsedTimer timer;
    timer.start();

    int received = 0;
    while (received < totalSize) {
        if (timer.elapsed() > timeoutMs) {
            return false;  // 超时
        }

        int ret = recv(sockfd, buf + received, totalSize - received, 0);
        if (ret > 0) {
            received += ret;
        } else if (ret < 0) {
            return false;  // 错误
        }
    }
    return true;  // 收齐了，未超时
}

// ────────────────────────────────────────────

// 方案 3（本工程方案）：select() 的 timeval 参数

// mynetwork.cpp:617 — 就是方案 2 的变体，用 select 替代 QElapsedTimer
int checkUdpSocketStatues(int seconds)
{
    struct timeval timeout;
    timeout.tv_sec = seconds;
    timeout.tv_usec = 500000;    // 精确到微秒
    int ret = select(sockfd + 1, &rds, NULL, NULL, &timeout);
    return (ret > 0) ? 1 : -1;
}
```

### 7.3 场景 C：每隔一段时间做一件事

```cpp
// === 标准模式：QTimer 循环 ===
class PeriodicTask : public QObject
{
    Q_OBJECT
public:
    void start()
    {
        m_timer = new QTimer(this);
        connect(m_timer, SIGNAL(timeout()), this, SLOT(onTick()));
        m_timer->start(2000);   // 每 2 秒
    }

private slots:
    void onTick()
    {
        // 每 2 秒执行一次
        checkSystemStatus();
        updateStatistics();
    }

private:
    QTimer *m_timer;
};

// === 子线程中的周期任务（moveToThread 模式） ===
QThread *monitorThread = new QThread;
MonitorWorker *monitor = new MonitorWorker;        // 不设 parent
monitor->moveToThread(monitorThread);
connect(monitorThread, SIGNAL(started()), monitor, SLOT(startMonitoring()));
monitorThread->start();
// MonitorWorker::startMonitoring() 在子线程中创建 QTimer → 在子线程触发

// === 重写 run() 死循环模式中的周期执行 ===
void MyThread::run()
{
    while (isrunning) {
        doPeriodicWork();
        msleep(2000);   // 休眠 2 秒，替代 QTimer
    }
}
```

---

## 第八章：本工程用法速查

| 需求 | 工程中的实现 | 模式 | 文件位置 |
|------|------------|------|---------|
| 持续收 UDP 波形/图像 | `MyNetWorkThread::run()` + `while(isrunning)` | 死循环 | `mynetwork.cpp:1089` |
| 持续收高速抓图 | `MyFastNetWorkThread::run()` | 死循环 | `myfastnetwork.cpp` |
| IPC 命令收发 | `SendThread::run()` + `exec()` | 事件循环 | `myudpthread.cpp` |
| MQTT 网络保活 | `mqttThread::run()` + `while(1)` | 死循环 | `mqttsrv.cpp:54` |
| MQTT 消息解析 | `mqttMsgParaseThread::run()` + `for(;;)` | 死循环 | `mqttsrv.cpp:222` |
| 状态轮询（时钟/清灰/报警） | `updateStatusThread` | 死循环 | `myautofeed.cpp` 等 |
| UI 每秒刷新 | `QTimer` + `timeout` 信号 | 主线程定时器 | `machineinfowidget.cpp` |
| 超时控制（收波形） | `select()` 的 `timeval` 参数 | 内核级超时 | `mynetwork.cpp:617` |
| 超时控制（采图等结果） | `QThread::wait(timeout)` | Qt 超时等待 | `mqttsrv.cpp:1149` |
| 超时控制（VPN 连接） | `QElapsedTimer` 等效（手动循环计数） | 自建超时 | `mqttsrv.cpp:1044-1067` |
| 信号跨线程通知 UI | `emit SignalWaveRecvFinished()` | AutoConnection | `mynetwork.cpp:1200` |
| 消息队列线程安全 | `QMutexLocker` + `recvMsgList` | 互斥锁 | `mqttsrv.cpp:170` |

---

## 附录：Qt 类的线程安全性速查

| 类 | 可在非 GUI 线程使用？ | 注意事项 |
|-----|-----|------|
| QWidget 及所有子类 | 不可 | 只能在 GUI 线程创建和操作 |
| QPixmap | 不可 | 用 QImage 替代子线程操作 |
| QImage | 可读，不可写 | 写操作需在主线程 |
| QString, QByteArray | 可 | 隐式共享，线程安全 |
| QList, QMap, QVector | 可 | 隐式共享，但并发写需加锁 |
| QMutex, QMutexLocker | 可 | 本身就是线程同步工具 |
| QThread | 可 | 管理器本身线程安全 |
| QTimer | 可 | 但依赖所在线程的事件循环 |
| QUdpSocket, QTcpSocket | 可 | 在子线程使用时需注意事件循环 |
| QSettings | 可 | 并发读写同一个 INI 文件需加锁 |
| QProcess | 可 | 但 `waitForFinished()` 在 GUI 线程会阻塞 UI |
