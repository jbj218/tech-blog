# 逻辑过期方案：让热点 key "永不过期"的艺术

> 这篇博客对应我学过的《Redis实战篇》逻辑过期部分，对应 RedisData + Hutool BeanUtil + 异步重建。

## 一句话定位

**逻辑过期 = 物理上不设过期时间 + 在 value 里塞一个 expireTime 字段 + 后台线程异步重建。** 比起互斥锁，它**不阻塞任何线程**，但牺牲了一点数据一致性——适合"宁可读到旧数据、也不能让请求被卡住"的场景。

---

## 一、两种击穿方案对比

| 维度 | 互斥锁 | 逻辑过期 |
| ---- | ---- | ---- |
| 线程是否阻塞 | ✅ 会阻塞（抢不到锁的等） | ❌ 不阻塞 |
| 数据一致性 | ✅ 强（拿到锁的查到啥就返回啥） | ⚠️ 弱（过期瞬间可能返回旧数据） |
| 实现复杂度 | 中（要写锁、Lua） | 高（要建缓存对象、异步线程） |
| 适用场景 | 数据必须准（库存、金额） | 宁可旧也不能卡（商品详情、文章） |

---

## 二、逻辑过期的数据结构

不直接存 `Shop`，而是**包一层 RedisData，把过期时间当字段塞进去**：

```java
@Data
public class RedisData {
    private LocalDateTime expireTime;   // 逻辑过期时间
    private Object data;                // 真正的业务数据
}
```

存进 Redis 的样子：

```json
{
  "expireTime": "2026-07-31T10:00:00",
  "data": {
    "id": 1,
    "name": "百味鲜面",
    "address": "..."
  }
}
```

---

## 三、为什么不能用强转

```java
RedisData redisData = JSONUtil.toBean(json, RedisData.class);

// ❌ 报错！
Shop shop = (Shop) redisData.getData();
// ClassCastException: JSONObject cannot be cast to Shop
```

### 根因

**`StringRedisTemplate` 反序列化出来的数据不会记录存入数据原本的数据类型。**

`RedisData` 里 `data` 声明为 `Object`，序列化整个对象为 JSON 存入 Redis 后，反序列化时 JSON 里没有"这个 Object 原本是 Shop"的类型信息，所以 `JSONUtil` 只能把它还原成通用的 **`JSONObject`（本质是个 Map）**。

`JSONObject` 和 `Shop` 是两个完全不同的类，没有继承关系，强转直接 `ClassCastException`。

### 必须用 `JSONUtil.toBean`

```java
Shop shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
```

`JSONUtil.toBean(...)` 的原理是**字段名反射拷贝**——把 Map 里的字段按字段名反射拷贝到 Shop 对象里，和强转有本质区别。

### 为啥还要强转 `(JSONObject)`

```java
JSONUtil.toBean(Object source, Class<T> clazz)     // 传 Object → 内部再判断
JSONUtil.toBean(JSONObject source, Class<T> clazz)  // 传 JSONObject → 直接映射
```

> **声明类型**和**运行时类型**是两回事。`redisData.getData()` 在编译器眼里就是 `Object`，会走到第一个重载（多一层反射判断）；强转后精确匹配第二个重载，方法调用更精准、代码意图更明确。

---

## 四、完整的逻辑过期方案

### 4.1 缓存读取流程

```
① 从 Redis 读 RedisData（没命中 → 缓存空标记 → 查 DB 重建）
② 判断 expireTime 是否过期
    ├─ 未过期 → 直接返回 data（Shop）
    └─ 已过期 → 异步开启线程重建缓存 + 返回旧数据（不阻塞）
```

### 4.2 异步重建线程（关键设计）

```java
// 拿到锁才能重建（避免 N 个线程同时重建）
if (tryLock("lock:shop:" + id)) {
    // 异步：开启独立线程查 DB + 写回 Redis
    CACHE_REBUILD_EXECUTOR.submit(() -> {
        try {
            // 1. 双重检查（防止在拿到锁前已经被其他线程重建过）
            RedisData redisData = stringRedisTemplate.opsForValue().get(key);
            if (redisData != null && redisData.getExpireTime().isAfter(now)) {
                return;
            }
            // 2. 查 DB
            Shop shop = shopMapper.selectById(id);
            // 3. 写回 Redis（设新的过期时间）
            RedisData newData = new RedisData();
            newData.setData(shop);
            newData.setExpireTime(LocalDateTime.now().plusSeconds(EXPIRE_SECONDS));
            stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(newData));
        } finally {
            unlock("lock:shop:" + id);
        }
    });
}

// 不管线程池里异步重建咋样，主线程直接返回旧数据
return shop;
```

### 4.3 为什么一定要加锁

**目的**：防止雪崩——N 个请求都发现"过期了"，如果同时各起一个线程重建，反而对数据库造成第二次雪崩。

---

## 五、互斥锁 vs 逻辑过期 的选型口诀

```
问自己两个问题：
1. 这个接口宁可卡死也不能返回旧数据吗？
   - 是 → 互斥锁（库存、扣款）
   - 否 → 逻辑过期（商品详情、博客正文）

2. 这个接口 QPS 高到不能容忍任何一个请求等待吗？
   - 是 → 逻辑过期（首页、热门商品）
   - 否 → 互斥锁（管理后台、低频接口）
```

---

## 六、Hutool BeanUtil 在缓存里的妙用

除了 `JSONUtil.toBean()` 做 Map → Bean，Hutool 还有一整套工具：

| 工具 | 用途 | 例子 |
| ---- | ---- | ---- |
| `JSONUtil.toBean(json, Class)` | JSON → Bean | 缓存反序列化 |
| `JSONUtil.toJsonStr(obj)` | Bean → JSON | 缓存序列化 |
| `BeanUtil.copyProperties(src, dest)` | 属性拷贝 | DTO ↔ Entity |
| `BeanUtil.isTrue(Boolean)` | null 安全布尔判断 | setIfAbsent 返回值处理 |
| `IdUtil.randomUUID()` | 生成 UUID | 分布式锁标识 |
| `RandomUtil.randomNumbers(n)` | 生成随机数字串 | 验证码生成 |
| `RegexUtils.isPhoneInvalid()` | 手机号校验 | 登录校验 |

> **核心**：Hutool 是 Java 开发的瑞士军刀，缓存、登录、工具类场景几乎都能用上。

---

## 七、总结

| 方案 | 核心 |
| ---- | ---- |
| **逻辑过期** | 物理不设 TTL + value 里塞 expireTime + 后台异步重建 |
| **JSONUtil.toBean** | Map/JSON 转 Bean 的标准方式（强转会 ClassCastException） |
| **异步重建线程** | 必须加锁 + 双重检查，避免雪崩 |
| **选型** | 一致性敏感选互斥锁，可用性敏感选逻辑过期 |

**一句话**：逻辑过期方案的精髓是"**返回旧数据 + 静默重建**"，是缓存击穿的高阶解法。