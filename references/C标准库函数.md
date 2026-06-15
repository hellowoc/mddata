# C 标准库函数

本工程中高频使用的 C 语言标准库函数。这些不是 Qt 的，而是 C 语言自带的。

---

## memcpy(dst, src, n)

**头文件**：`<cstring>` 或 `<string.h>`

**签名**：`void *memcpy(void *dst, const void *src, size_t n);`

**作用**：从 `src` 地址拷贝 `n` 个字节到 `dst` 地址。**按字节拷贝，不管数据类型。**

**类比**：复印机——把原件的内容一字不差地复制到另一张纸上。

**本工程中的典型用法**：

```cpp
// myudpthread.cpp:79 — 从收到的包中取出 12 字节包头
udp_header head;                          // 目标：栈上的结构体
memcpy(&head, datagram.data(), headSize); // 从 QByteArray 拷 12 字节到 head

// myudpthread.cpp:788-789 — 拼包：头 + 数据
memcpy(send_buf.data(), &header, headSize);       // 先拷 12 字节头
memcpy(send_buf.data() + headSize, buf, dataLen); // 再拷数据，从偏移 12 开始

// myudpthread.cpp:86 — 取出包头后的数据部分
memcpy(buf, datagram.data() + headSize, nPacketLen); // 跳过 12 字节包头
```

---

## memset(ptr, value, n)

**头文件**：`<cstring>` 或 `<string.h>`

**签名**：`void *memset(void *ptr, int value, size_t n);`

**作用**：把 `ptr` 指向的 `n` 个字节全部设为 `value`。通常 `value = 0`，即**清空一段内存**。

**类比**：用修正液涂掉一张纸上的所有内容。

**本工程中的典型用法**：

```cpp
// myudpthread.cpp:35 — 数组清零
memset(camBadPoiontRet, 0, sizeof(camBadPoiontRet));

// mynetwork.cpp:2408 — 结构体清零
memset(&addr_local, 0, sizeof(addr_local));

// 为什么需要清零？C/C++ 中，栈上的局部变量初始值是随机的（垃圾值）。
// 不清零的话，结构体的某些字段可能是乱码。
```

---

## malloc(n) / free(ptr)

**头文件**：`<cstdlib>` 或 `<stdlib.h>`

**签名**：
- `void *malloc(size_t n);`
- `void free(void *ptr);`

**作用**：`malloc` 在**堆**上分配 `n` 字节内存，返回指针。`free` 释放之前分配的内存。

**为什么不用 `new/delete`**：这行代码出现在 2015 年左右的 C++ 工程中，当时混合使用 C 和 C++ 分配方式很常见。

```cpp
// myudpthread.cpp:84 — 包头后数据部分的长度 nPacketLen 是运行时才知道的
unsigned char* buf = (unsigned char*)malloc(nPacketLen);  // 分配 nPacketLen 字节
memset(buf, 0, nPacketLen);     // 清零
// ... 使用 buf ...
free(buf);                       // 用完释放！
```

注意：如果 `malloc` 了但忘了 `free`，造成**内存泄漏**——程序占用的内存越来越大。

---

## printf(format, ...)

**头文件**：`<stdio.h>` 或 `<cstdio>`

**签名**：`int printf(const char *format, ...);`

**作用**：向终端输出格式化字符串。调试时用。

**本工程中的典型用法**：

```cpp
// myudpthread.cpp:98
printf("\n222 recv[%d][%d] cmd[%02x] data[", view, unit, nCmd);
// %d  → 十进制整数
// %02x → 十六进制，至少 2 位，不足补 0

// mynetwork.cpp:2385
printf("OK: Obtain Socket Despcritor sucessfully.\n");
// \n → 换行
```

**`qDebug()` vs `printf`**：`qDebug()` 是 Qt 的调试输出（自动加换行、可重定向到文件），`printf` 是 C 标准库的（直接输出到 stdout）。本工程混用两者。

---

## sprintf(buf, format, ...)

**头文件**：`<stdio.h>`

**签名**：`int sprintf(char *buf, const char *format, ...);`

**作用**：和 `printf` 一样，但结果写入 `buf` 字符串而不是输出到终端。

```cpp
// myudpthread.cpp:899-900
char cLog[100] = {0};
sprintf(cLog, "send udp packet id 255 error (layer: %d, sorter: %d) !!!", layer, sorter);
// 结果："send udp packet id 255 error (layer: 1, sorter: 2) !!!"
```

---

## sizeof(x)

**是什么**：C/C++ 的**编译时运算符**（不是函数），返回类型或变量占用的字节数。

```cpp
sizeof(int)           → 4    (32位系统中 int 占 4 字节)
sizeof(udp_header)    → 14   (12字节字段 + 2字节对齐填充)
sizeof(camBadPoiontRet) → 整个数组占的字节数
```

用在 `memcpy`/`memset`/`malloc` 中告诉函数要操作多少字节。

---

## ofstream (C++ 文件写入流)

**头文件**：`<fstream>`

**是什么**：C++ 的文件写入类。把数据写入磁盘文件。

**本工程中的典型用法**：

```cpp
// myudpthread.cpp:65
ofstream fout;                                 // 创建文件流对象
fout.open("/sdcard/ipc_debug.txt", ios::app);  // 打开文件（追加模式）
fout << "some text" << endl;                   // 写入内容
fout.close();                                  // 关闭文件
```
