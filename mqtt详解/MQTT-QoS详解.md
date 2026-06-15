# MQTT QoS 详解 — 三级消息可靠性保证

## 先理解背景：MQTT 消息的"送达路径"

```
发布者 ───PUBLISH───→ Broker ───PUBLISH───→ 订阅者
```

QoS 控制的是**每一段路径上的可靠性**，发布者→Broker 和 Broker→订阅者是独立协商的——发布者用 QoS 2 发给 Broker，订阅者可以用 QoS 0 从 Broker 收，互不影响。

---

## QoS 0：最多一次

```
发布者 → Broker: PUBLISH(data)  
         然后就没了。不等确认，不重发，不检查。
```

**数据流**：消息发出后，发送方**立即丢弃**这份数据。不缓存，不跟踪，不关心 Broker 收到没。

**代价**：零。没有额外的网络往返，没有内存占用缓存消息。

**丢失场景**：TCP 链路本身可能丢包（虽然 TCP 有重传，但极端情况下连接断开+重连，这条消息就彻底没了）。

**用于**：高频传感器数据。温度每秒上报一次——丢一次，下一秒新的温度值就到了，不在乎。

---

## QoS 1：至少一次

```
发布者 → Broker: PUBLISH(PacketId=123, data)
         Broker 把 data 存下来
发布者 ← Broker: PUBACK(123)
         收到 PUBACK 后，发布者才删除缓存的 data

如果发布者超时没收到 PUBACK → 重发 PUBLISH(DUP=1, PacketId=123, data)
如果 Broker 收到 DUP=1 的 PUBLISH → 发现 PacketId=123 已处理 → 只回 PUBACK，不重复存储
```

**数据结构变化**：发送方必须维护一个**待确认消息表**：

```
发送方内存中的结构：
  ┌──────────┬──────────┬──────────┬──────────┐
  │ PacketId │ 消息体    │ 发送时间  │ 重发次数  │
  ├──────────┼──────────┼──────────┼──────────┤
  │   123    │ "50:20:" │ 14:30:01 │    0     │  ← 等 PUBACK 中
  │   124    │ "1:0:1:" │ 14:30:02 │    0     │
  └──────────┴──────────┴──────────┴──────────┘
```

Broker 收到 PUBLISH 后：先存消息 → 立即回 PUBACK → 然后转发给订阅者。

发布者收到 PUBACK 后：从待确认表中删除 PacketId=123 → 释放内存。

**"至少一次"的含义**：如果 PUBACK 丢了，发布者会重发 PUBLISH → Broker 收到**可能重复的消息**。Broker 用 PacketId 去重（收到重复的 PacketId 只回 PUBACK 不再存储），但订阅者收到的可能是两条相同的消息（因为 Broker→订阅者也用 QoS 1，也存在同样的 PUBACK 丢失导致重发）。

**代价**：发送方需要缓存消息直到收到 PUBACK。对于本工程，publish 调完后消息体还在内存中（mosquitto 库内部管理）。

---

## QoS 2：恰好一次

最复杂的四步握手。**"恰好一次"不是技术上真的不能重复——而是通过四次握手确保了"发送方知道接收方知道发送方知道消息已被处理"，从而安全地从两端删除缓存。**

```
步骤1: 发布者 → Broker:  PUBLISH(PacketId=123, data)
       发布者缓存 data，等 PUBREC

步骤2: 发布者 ← Broker:  PUBREC(123)
       Broker 缓存 data（还没转发给订阅者！）
       Broker 的意思是："我收到了，存好了，但我还没处理"

步骤3: 发布者 → Broker:  PUBREL(123)
       发布者收到 PUBREC，知道 Broker 已安全存储
       发 PUBREL = "现在你可以放心处理/转发了"
       发布者此时可以删 data 了（但保留 PacketId 用于最后一步确认）

步骤4: 发布者 ← Broker:  PUBCOMP(123)
       Broker 处理完了（转发了），发布者删除 PacketId 记录
```

### 为什么要分 PUBREC 和 PUBREL 两步？直接 PUBLISH → PUBACK 不就完了？

关键区别在于：**"Broker 收到了数据"和"Broker 认为自己有责任处理这条数据"是两件事**。

```
QoS 2 场景：发布者发了 PUBLISH，Broker 回了 PUBREC，然后网络断了。

发布者视角：
  - 没收到 PUBREC → 重发 PUBLISH（Broker 用 PacketId 去重）
  - 收到了 PUBREC，没收到 PUBCOMP → 重发 PUBREL（不是重发 data，只是说"请完成处理"）
  
Broker 视角：
  - 发了 PUBREC 后，Broker 承担了"必须最终处理这条消息"的责任
  - 即使发布者掉线，Broker 仍然会转发给订阅者
  - 发布者重连后发 PUBREL，Broker 回 PUBCOMP，干净清理
```

### 状态机视角（发送方）

```
         send PUBLISH
  IDLE ──────────────→ WAIT_FOR_PUBREC
                          │
              recv PUBREC │
  WAIT_FOR_PUBREC ───────→ WAIT_FOR_PUBCOMP
                           send PUBREL
                          │
              recv PUBCOMP│
  WAIT_FOR_PUBCOMP ───────→ IDLE (删除缓存)
```

**代价**：两次额外往返（PUBLISH→PUBREC→PUBREL→PUBCOMP vs QoS 0 的一次 PUBLISH 即结束），对本工程 publish 到 `"MqttServerClient"` 每次多 3 个往返（PUBREC/PUBREL/PUBCOMP），在工厂内网延迟可忽略（RTT < 1ms），跨互联网约增加几十毫秒。

---

## 三级对比

```
QoS 0:  PUBLISH ──────────────────────────→  没了
        发送方不缓存，不确认

QoS 1:  PUBLISH ──────────────────────────→  收到，回 PUBACK
        发送方缓存到收 PUBACK

QoS 2:  PUBLISH ──→ 收到，回 PUBREC ──→ 收到 PUBREC，发 PUBREL ──→ 收到 PUBREL，处理完，回 PUBCOMP
        发送方缓存 data 到收 PUBREC，缓存 PacketId 到收 PUBCOMP
        Broker 从收 PUBLISH 开始承担责任
```

| | QoS 0 | QoS 1 | QoS 2 |
|------|------|------|------|
| 确认次数 | 0 | 1 (PUBACK) | 3 (PUBREC+PUBREL+PUBCOMP) |
| 发送方缓存 | 不缓存 | 缓存到收 PUBACK | 缓存 data 到收 PUBREC，缓存 PacketId 到收 PUBCOMP |
| 重复可能性 | 无（丢了就丢了） | 可能重复（PUBACK 丢失导致重发） | 不重复 |
| 额外网络往返 | 0 | 1 | 3 |
| 本工程用法 | 不使用 | 不使用 | **onCmdReturn 回包用 QoS 2** |

---

## 本工程为什么 publish 用 QoS 2

```cpp
// mqttsrv.cpp:1439
int rc = myMqttThread->test->publish(NULL, "MqttServerClient",
                                      str.length(), str.c_str(), 2);
//                                                          ↑ QoS=2
```

`onCmdReturn` 是给手机回传命令执行结果——手机必须收到这个回包。如果丢了，手机显示"命令已发送"但永远不知道执行成功了没。QoS 2 保证恰好送达一次——即使网络断了一下，mosquitto 内部会自动完成 PUBREC→PUBREL→PUBCOMP 重试，最终一定送达。
