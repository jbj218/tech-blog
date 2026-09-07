# 我做的一个本地生活类项目：Redis + MyBatis-Plus + Spring Boot 的整合

> 这篇博客把我做项目时踩过的真实业务问题串起来——从登录到缓存到秒杀，看一个完整的实战项目是怎么把 Redis、MyBatis-Plus、Spring Boot、Hutool 整合起来的。

## 一句话定位

**这个项目 = 一个本地生活类 App 的后端**，核心技术栈是 **Spring Boot + MyBatis-Plus + Redis + Hutool**，把"短信登录、店铺查询、优惠券秒杀"这些经典业务场景串成了一条完整的实战链路。

---

## 一、项目整体架构

```
浏览器/手机 App
    ↓ HTTP 请求
Nginx（负载均衡）
    ↓
Tomcat 集群（Spring Boot 应用）
    ├─ Controller 层（接收请求、返回 Result）
    ├─ Service 层（业务逻辑 + @Transactional）
    ├─ Mapper 层（MyBatis-Plus BaseMapper）
    ├─ Redis（缓存 + 分布式锁 + 消息队列）
    └─ MySQL（最终落库）
    ↓
返回 JSON 响应
```

### 1.1 技术栈

| 层级 | 技术 |
| ---- | ---- |
| Web 框架 | Spring Boot 2.3.12 + Spring MVC |
| 持久层 | MyBatis-Plus 3.x |
| 缓存 / 锁 | Redis（StringRedisTemplate） |
| 工具类 | Hutool（BeanUtil / JSONUtil / IdUtil / RandomUtil） |
| JDK | 19 |
| 构建 | Maven |

### 1.2 项目模块

| 模块 | 关键业务 | 用到的技术 |
| ---- | ---- | ---- |
| 短信登录 | 手机号登录、Token、双拦截器 | Redis + ThreadLocal + 拦截器 |
| 商户查询缓存 | 店铺详情查询、缓存三大问题 | Redis + Hutool BeanUtil + Lua |
| 优惠券秒杀 | 抢券、超卖、一人单 | Lua 脚本 + RedisIdWorker + Stream MQ |
| 附近商户 | GEO 查询 | Redis GEO 数据结构 |
| UV 统计 | 海量去重 | Redis HyperLogLog |
| 用户签到 | 签到统计 | Redis BitMap |
| 好友关注 | 关注、取关、共同关注 | Redis Set |
| 达人探店 | 点赞、排行榜 | Redis List + SortedSet |

---

## 二、短信登录：基础链路

### 2.1 整体流程

```
用户输入手机号 → 后端生成验证码 → 存 Redis（5 分钟）→ 模拟发送
    ↓
用户输入验证码 → 校验 → 查/建用户 → 生成 token → 存 Redis（30 分钟）
    ↓
返回 token → 前端存 localStorage → 后续请求带 authorization 头
    ↓
双拦截器：刷新拦截器（/ *）+ 校验拦截器（/order/**）
    ↓
业务代码里 UserHolder.getUser() 取当前用户
```

### 2.2 关键技术

| 技术 | 作用 |
| ---- | ---- |
| **Redis 替代 Session** | 跨 Tomcat 共享登录态 |
| **Token（UUID）** | 手动生成的凭证，前端每次请求带上 |
| **双拦截器** | 刷新拦截器（/*）续命 + 校验拦截器（/order/**）拦截 |
| **ThreadLocal** | 单请求内避免重复查 Redis |
| **构造器注入** | 拦截器是 `new` 的，依赖靠 MvcConfig 注入再传 |

### 2.3 为什么这样设计

> **HTTP 是无状态的**，"我登录过"这个状态服务器从不保存——靠的是每个请求都带 token、每个请求都被拦截器独立验证一次；ThreadLocal 只是把验证结果在单次请求内传递下去，方便业务取用。

---

## 三、商户查询缓存：缓存三大问题

### 3.1 缓存设计

```
请求 → Redis 查缓存
        ├─ 命中 → 返回
        └─ 未命中 → 查 DB → 写回 Redis → 返回
```

### 3.2 击穿 / 穿透 / 雪崩

| 问题 | 项目里的解法 |
| ---- | ---- |
| **击穿（热点 key 过期）** | 互斥锁 + 逻辑过期（店铺详情场景） |
| **穿透（DB 也没有）** | 缓存空值 |
| **雪崩（大量 key 同时过期）** | 实际生产用 TTL 随机扰动 |

### 3.3 互斥锁核心代码

```java
Shop shop = (Shop) stringRedisTemplate.opsForValue().get("shop:" + id);
if (shop != null) return shop;

// 缓存未命中 → 查 DB
shop = shopMapper.selectById(id);
if (shop == null) {
    // 缓存空值（防穿透）
    stringRedisTemplate.opsForValue().set("shop:empty:" + id, "", 2, TimeUnit.MINUTES);
    return null;
}

// 写回缓存（带 TTL）
stringRedisTemplate.opsForValue().set("shop:" + id, JSONUtil.toJsonStr(shop), 30, TimeUnit.MINUTES);
return shop;
```

**进阶版（逻辑过期）**：

```java
RedisData redisData = JSONUtil.toBean(json, RedisData.class);
if (redisData.getExpireTime().isAfter(LocalDateTime.now())) {
    return JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
}
// 已过期 → 异步重建
return handleExpiredShop(id);
```

---

## 四、优惠券秒杀：Redis 的高阶用法

### 4.1 三大挑战

| 挑战 | 解法 |
| ---- | ---- |
| **库存超卖** | Lua 脚本原子判断 + decr |
| **一人一单** | Lua 里 `sadd` 记录已抢用户 |
| **写库慢** | Redis Stream 异步下单 |

### 4.2 Lua 脚本一气呵成

```lua
-- 判断是否已抢过 → 扣库存 → 记录用户
if redis.call('sismember', KEYS[2], ARGV[1]) == 1 then return 2 end
if tonumber(redis.call('get', KEYS[1])) <= 0 then return 1 end
redis.call('decr', KEYS[1])
redis.call('sadd', KEYS[2], ARGV[1])
return 0
```

### 4.3 完整秒杀链路

```
用户请求 → Lua 脚本（判断 + 扣库存 + 记用户）
            ├─ 返回 0（成功） → RedisIdWorker 生成订单 ID
            │               → XADD 发订单到 Stream
            │               → 立刻返回"抢券成功"
            │
            └─ 异步消费者 XREADGROUP
                          → BeanUtil.fillBeanWithMap 转 VoucherOrder
                          → 写 MySQL
                          → XACK 确认
```

### 4.4 完整技术栈融合

```
Spring Boot Controller
   ↓
StringRedisTemplate.execute(Lua 脚本)
   ↓
RedisIdWorker.nextId("order")
   ↓
StringRedisTemplate.opsForStream().add(...)
   ↓
后台消费者 @Component + @PostConstruct 启动线程
   ↓
BeanUtil.fillBeanWithMap(map, new VoucherOrder(), true)
   ↓
MyBatis-Plus voucherOrderMapper.insert(order)
```

---

## 六、整体设计模式总结

### 6.1 "判断 → 操作" 都用 Lua

凡是涉及"先查再改"的 Redis 操作，都塞到 Lua 里——这是 Redis 单线程模型的精髓。

### 6.2 "缓存重建" 都加锁 + 双重检查

无论是互斥锁还是逻辑过期，重建缓存前都要双重检查，防止空跑。

### 6.3 "DTO → Entity" 都用 BeanUtil

Hutool BeanUtil 是项目里反复出现的小工具，省掉大量手写 set。

### 6.4 "返回值" 统一用 Result

```java
public class Result {
    private Boolean success;
    private Integer code;
    private String msg;
    private T data;

    public static <T> Result<T> ok() { return new Result<>(true, 200, "ok", null); }
    public static <T> Result<T> ok(T data) { return new Result<>(true, 200, "ok", data); }
    public static <T> Result<T> fail(String msg) { return new Result<>(false, 500, msg, null); }
}
```

---

## 七、一句话总结

| 模块 | 核心技术 |
| ---- | ---- |
| **登录** | Redis Token + 双拦截器 + ThreadLocal |
| **缓存** | 互斥锁 / 逻辑过期 / 空值缓存 |
| **秒杀** | Lua 脚本 + RedisIdWorker + Stream MQ |
| **持久化** | MyBatis-Plus BaseMapper + LambdaQueryWrapper |
| **工具** | Hutool BeanUtil + JSONUtil + IdUtil |

**整个项目就是一面镜子：把学过的 Redis 三大缓存问题、分布式锁、消息队列、ID 生成器、MyBatis-Plus 整合到 Spring Boot 里，用 Hutool 串起来——这就是真实项目的"骨架"。**

---

## 八、实战常见问题

| 问题 | 一句话答案 |
| ---- | ---- |
| 为什么用 Redis 不用 Session？ | 集群下 Session 不共享，Redis 集中存储跨 Tomcat |
| 缓存击穿怎么办？ | 互斥锁（重）/ 逻辑过期（可用性优先） |
| 秒杀超卖怎么防？ | Lua 脚本原子扣库存 |
| 分布式锁怎么实现？ | SETNX + EX + UUID + Lua（生产用 Redisson） |
| 为什么用 Stream 不用 List？ | Stream 支持消费者组 + 持久化，不丢消息 |
| RedisIdWorker 和雪花算法的区别？ | RedisIdWorker 依赖 Redis，QPS 低但简单；雪花算法无依赖但有时钟回拨问题 |