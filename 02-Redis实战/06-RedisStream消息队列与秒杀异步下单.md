# Redis Stream 消息队列：秒杀订单的异步处理

> 这篇博客对应我学过的《Redis实战篇》优惠券秒杀 + Redis Stream 部分。

## 一句话定位

**Redis Stream 是 Redis 5.0+ 提供的轻量级消息队列**，用于秒杀场景下的"下单"与"实际写库"解耦——消费者用 `XREADGROUP` 读取消息，按 ACK 机制保证不丢消息。

---

## 一、秒杀场景的两大难题

### 1.1 库存超卖

1000 张券，1 万个用户抢 → 必须保证只卖出 1000 张。

**经典坑**：

```java
// ❌ 先查后改，必超卖！
int stock = queryStock(id);       // 查到 1
if (stock > 0) {
    updateStock(id, stock - 1);    // 改完还是 1
}
```

**根因**：查和改不是原子的，中间窗口被并发插足。

### 1.2 一人单（一人只能抢一张）

```sql
-- 错误：同一用户并发 100 个请求，每个都查到"没抢过"，都插进订单
SELECT * FROM voucher_order WHERE user_id = ? AND voucher_id = ?;
INSERT INTO voucher_order (...) VALUES (...);
```

---

## 二、Lua 脚本：把秒杀判断写成原子操作

把"判断资格 + 扣库存"放在 Redis 一条 Lua 脚本里执行 → **单线程 + 串行 = 原子**：

```lua
-- KEYS[1]: stock key, KEYS[2]: order key
-- ARGV[1]: userId

-- 1. 判断是否已下单
if (redis.call('sismember', KEYS[2], ARGV[1]) == 1) then
    return 2;   -- 已抢过
end

-- 2. 扣库存
local stock = tonumber(redis.call('get', KEYS[1]))
if (stock == nil or stock <= 0) then
    return 1;   -- 库存不足
end
redis.call('decr', KEYS[1])

-- 3. 记录用户已下单
redis.call('sadd', KEYS[2], ARGV[1])
return 0;       -- 成功
```

```java
Long result = stringRedisTemplate.execute(
    new DefaultRedisScript<>(script, Long.class),
    Arrays.asList("seckill:stock:" + voucherId, "seckill:order:" + voucherId),
    userId.toString()
);

if (result == 0) { /* 抢到 */ }
else if (result == 1) { /* 库存没了 */ }
else if (result == 2) { /* 你抢过了 */ }
```

**为什么 Lua 能解决超卖**：Redis 是单线程执行命令的，而 Lua 脚本在 Redis 里跑相当于一个"超级命令"——脚本内的所有 Redis 操作是连续执行的，期间不会有其他客户端插入。

---

## 三、Stream 消息队列：异步下单

### 3.1 为什么需要异步

Lua 脚本只能保证"判断+扣库存"原子，但**订单写 DB 还是要走 MySQL**——这一步很慢，且容易拖垮数据库。

**解法**：把订单信息塞进消息队列，立刻返回"抢券成功"；后台异步线程消费队列，真正写 DB。

```
用户请求 → Lua 脚本（秒级判断）→ 塞订单到 Stream → 立刻返回"成功"
                                                    ↓
                                       异步消费者 XREADGROUP 读取
                                                    ↓
                                            BeanUtil.toBean 转 Order
                                                    ↓
                                                写 MySQL
```

### 3.2 Redis Stream 三种消息队列对比

| 队列 | 特点 | 适用 |
| ---- | ---- | ---- |
| **Pub/Sub** | 简单发布订阅 | 允许丢消息（聊天、广播） |
| **List + BRPOP** | 阻塞读，模拟队列 | 不能 ACK，断电丢消息 |
| **Stream（XREADGROUP）** | 持久化 + 消费者组 + ACK | ✅ 秒杀订单（不丢消息） |

### 3.3 Stream 关键命令

```bash
# 1. 发消息
XADD stream.orders * userId 1001 voucherId 2001

# 2. 创建消费者组
XGROUP CREATE stream.orders g1 0

# 3. 消费者读消息（阻塞读新消息）
XREADGROUP GROUP g1 consumer1 COUNT 1 BLOCK 2000 STREAMS stream.orders >

# 4. 确认处理完毕（ACK）
XACK stream.orders g1 <message-id>

# 5. 查询 Pending List（未确认的消息）
XPENDING stream.orders g1
```

### 3.4 Java 代码实现

```java
// 生产端：发消息到 Stream
stringRedisTemplate.opsForStream().add("stream.orders",
    Map.of("userId", userId.toString(),
           "voucherId", voucherId.toString()));

// 消费端：阻塞读 + 异步处理
while (running) {
    // 1. 读消息（COUNT 1 BLOCK 2000）
    List<MapRecord<String, Object, Object>> records = stringRedisTemplate
        .opsForStream()
        .read(Consumer.from("g1", "consumer1"),
              StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
              StreamOffset.create("stream.orders", ReadOffset.lastConsumed()));

    if (records == null || records.isEmpty()) continue;

    // 2. 转 Bean（Hutool BeanUtil）
    MapRecord<String, Object, Object> record = records.get(0);
    Map<Object, Object> values = record.getValue();
    VoucherOrder order = BeanUtil.fillBeanWithMap(values, new VoucherOrder(), true);

    // 3. 写 DB
    handleVoucherOrder(order);

    // 4. ACK
    stringRedisTemplate.opsForStream().acknowledge("stream.orders", "g1", record.getId());
}
```

### 3.5 为什么用 `BeanUtil.fillBeanWithMap`

`Map → Bean` 的转换不能强转（Map 不是 Bean），需要按字段名反射拷贝。Hutool `BeanUtil.fillBeanWithMap` 一行搞定：

```java
VoucherOrder order = BeanUtil.fillBeanWithMap(map, new VoucherOrder(), true);
//                                     ↑ true 表示忽略大小写、类型转换
```

---

## 四、Stream vs 其他 MQ 对比

| 维度 | Redis Stream | RabbitMQ | Kafka |
| ---- | ---- | ---- | ---- |
| 部署成本 | 极低（已有 Redis） | 高 | 高 |
| 吞吐量 | 中（几万 QPS） | 中（几万 QPS） | 极高（百万 QPS） |
| 消息可靠性 | ✅ ACK + Pending | ✅ 强 | ✅ 极强 |
| 适用规模 | 小型项目、秒杀 | 中型项目 | 大数据、流式计算 |

**秒杀场景**：日订单 10 万以下 Redis Stream 够用，再大上 RabbitMQ/Kafka。

---

## 五、完整秒杀架构图

```
用户点击"抢购"
    ↓
Lua 脚本：判断资格 + 扣库存 + 记用户
    ├─ 失败 → 返回错误码
    └─ 成功 → 生成订单 ID（RedisIdWorker）
              ↓
         XADD 发订单到 Stream → 立刻返回"抢券成功"
              ↓
         异步消费者 XREADGROUP
              ↓
         BeanUtil.toBean 转 VoucherOrder
              ↓
         写 MySQL + ACK
```

---

## 六、一句话总结

| 关键点 | 一句话 |
| ---- | ---- |
| **Lua 脚本** | Redis 单线程 + Lua 脚本 = "判断+扣库存"原子执行 |
| **超卖 / 一人单** | 用 `decr` + `sadd` 在脚本里一并解决 |
| **Stream 消息队列** | Redis 5.0+ 轻量级 MQ，支持 ACK 不丢消息 |
| **异步下单** | 立刻返回 + 后台异步写 DB，秒杀体验不卡顿 |
| **BeanUtil.fillBeanWithMap** | Map → Bean 的标准工具 |

**核心**：秒杀 = Lua 脚本守门 + Stream 异步处理 + RedisIdWorker 拼 ID，三件套组合实现高并发抢券。