# RedisIdWorker：基于 incr 的全局唯一 ID 生成器

> 这篇博客对应我学过的《Redis实战篇》RedisIdWorker 实现 + 秒杀业务部分。

## 一句话定位

**RedisIdWorker = 用 Redis 的 `INCR` 自增特性 + 业务前缀拼接出全局唯一 ID。** 它比 UUID 短、比数据库自增快、是分布式场景下生成订单号的轻量级首选方案。

---

## 一、为什么不用 UUID

| 方案 | 长度 | 趋势递增 | 可读性 | 性能 |
| ---- | ---- | ---- | ---- | ---- |
| **UUID** | 36 字符（带 -） | ❌ 无序 | ❌ 难看 | ✅ 极快 |
| **DB 自增** | 8 字符 | ✅ | ✅ | ⚠️ 写一次慢 |
| **RedisIdWorker** | 十几字符 | ✅ | ✅（带业务前缀） | ✅ 极快 |
| **雪花算法** | 19 位 Long | ✅ | ⚠️ | ✅ 但有机器码问题 |

**秒杀订单 ID 的诉求**：
1. 全局唯一（不能撞）
2. 单调递增（方便排索引、排序、避免页分裂）
3. 短一点（URL、日志要易读）
4. 高并发下生成快（不能拖累主业务）

UUID 满足 1、4，但不满足 2、3。**RedisIdWorker 完美匹配这四条**。

---

## 二、ID 结构设计

```
| 时间戳（31 bit）| 序列号（32 bit） |
```

```java
// 64 bit Long 的典型拆分（也是雪花算法的简化版）
0 | 0000000 00000000 00000000 0000000 0000 | 00000 00000000 00000000 00000000 00000000
  |          31 bit（秒级时间戳，约 68 年）  |               32 bit（自增序列号，1 秒 40 亿）
```

课程里的简化版只用 1 天内的自增序列号：

```java
private static final long BEGIN_TIMESTAMP = 1640995200L;   // 2022-01-01 00:00:00

public long nextId(String keyPrefix) {
    // 1. 生成时间戳
    LocalDateTime now = LocalDateTime.now();
    long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
    long timestamp = nowSecond - BEGIN_TIMESTAMP;

    // 2. 生成序列号（Redis 自增）
    String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
    long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);

    // 3. 拼接
    return timestamp << 32 | count;   // 位运算拼接 64 位 Long
}
```

---

## 三、Redis `INCR` 的关键作用

```java
stringRedisTemplate.opsForValue().increment("icr:order:" + date);
```

**为什么 Redis 适合做计数器**：

| 特性 | Redis | 关系型数据库 |
| ---- | ---- | ---- |
| 单条 INCR 性能 | 几万 QPS | 几千 QPS（写日志） |
| 原子性 | ✅ | ✅ |
| 持久化 | 可选 | ✅ |
| 跨服务 | ✅ 共享 | 集群下要序列号服务器 |

**Redis 的 INCR 是单线程的**，天然原子，不需要额外加锁。

### 3.1 序列号的两个关键设计

```java
"icr:" + keyPrefix + ":" + date   // 按天切分 key
```

| 设计 | 作用 |
| ---- | ---- |
| **按天切分 key** | 每天的 key 独立计数，避免单 key 长期无限增长 |
| **业务前缀** | order / voucher / user 不同业务互不干扰 |

---

## 四、位运算拼接

```java
return timestamp << 32 | count;
```

**为什么用位运算**：

- `timestamp << 32`：时间戳左移 32 位，把低 32 位让出来给序列号
- `| count`：OR 操作填进序列号
- 最终拼成一个 64 位 Long

**好处**：
1. **极快**：CPU 原生指令，纳秒级
2. **紧凑**：一个 Long 表达两个信息
3. **可拆分**：`timestamp >> 32` 又能把时间戳拆出来

> 如果用字符串拼接（`timestamp + "_" + count`），长度翻倍、转字符串开销大、不能直接做 Long 比较。

---

## 五、RedisIdWorker vs 雪花算法

| 维度 | RedisIdWorker | 雪花算法（Snowflake） |
| ---- | ---- | ---- |
| 依赖 | Redis | 无（纯内存） |
| 单机瓶颈 | INCR 几万 QPS | 单机 400 万 QPS |
| 多机协调 | 天然共享 | 需要配置 workerId |
| 部署成本 | 需要 Redis 服务 | 零 |
| 时钟回拨 | 无影响 | ⚠️ 致命（机器时间回退会撞 ID） |
| 位数 | 十几位（业务决定） | 64 位 Long |

**秒杀场景的选型**：

- 项目规模小、QPS < 5 万 → **RedisIdWorker 够用**
- QPS 极高 / 多机房 → **雪花算法**（推荐 Hutool 封装的 `IdUtil.getSnowflake()`）

---

## 六、在秒杀里的应用

### 6.1 下单流程中的 ID 生成点

```java
@Override
public Result seckillVoucher(Long voucherId) {
    // 1. 查券、判断时间、库存（Lua 脚本保证原子）
    // ...

    // 2. 扣库存（Lua 脚本）
    // ...

    // 3. 创建订单（ID 由 RedisIdWorker 生成）
    long orderId = redisIdWorker.nextId("order");

    // 4. 写订单到 DB
    VoucherOrder order = new VoucherOrder();
    order.setId(orderId);
    order.setUserId(UserHolder.getUser().getId());
    order.setVoucherId(voucherId);
    save(order);

    // 5. 返回订单 ID
    return Result.ok(orderId);
}
```

### 6.2 为什么订单 ID 这么重要

- 唯一索引（数据库主键）
- 对账/查询（用户查订单）
- 防重（前端可以幂等提交）
- 美观（URL、日志带业务前缀的时间戳）

---

## 七、一句话总结

| 关键点 | 一句话 |
| ---- | ---- |
| **RedisIdWorker 是什么** | Redis `INCR` + 时间戳 + 位运算拼接的全局唯一 ID |
| **为什么不用 UUID** | UUID 太长、无序、不可读、不利于做索引 |
| **序列号按天切 key** | 避免单 key 无限增长、不同业务互不干扰 |
| **位运算拼接** | 紧凑、极快、可拆分 |
| **生产替代方案** | 雪花算法（Hutool `IdUtil.getSnowflake()`） |

**核心**：RedisIdWorker 是用最简单的方式解决"分布式全局唯一 ID + 趋势递增"的问题，在秒杀场景里够用。