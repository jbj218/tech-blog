# Redis 分布式锁 SimpleRedisLock：从 SETNX 到 Redisson

> 这篇博客对应我学过的 SimpleRedisLock 实现 + Redis 实战篇"分布式锁"内容。

## 一句话定位

**分布式锁 = 用 Redis 这种"公共存储"实现的、跨 JVM 都能互斥的锁。** 它的精髓是"用一个 UUID 标识归属、用 Lua 脚本保证原子性"，以及"先比对再删除"防误删。

---

## 一、为什么需要分布式锁

### 1.1 JVM 锁的局限

`synchronized`、`ReentrantLock` 都是**单 JVM 内**的锁。集群部署时，每台 JVM 各有自己的锁——线程 A 在机器 1 拿到锁，线程 B 在机器 2 同时也能拿到 → 数据错乱。

### 1.2 分布式锁要解决的三个问题

1. **互斥**：任何时刻只能有一个客户端持有锁
2. **不死锁**：即使持锁客户端崩溃，锁也要能被释放
3. **不错乱**：不能误删别人的锁

---

## 二、Redis 分布式锁的演进

### 2.1 第一版：SETNX + 单独 EX（❌ 非原子）

```java
// ❌ 错误！如果 SETNX 之后客户端崩溃，永远不会 EX
redis.setnx("lock", "1");
redis.expire("lock", 10);
```

**根因**：两条命令不是原子的，可能只执行了第一条。

### 2.2 第二版：SETNX + EX 一句话（✅ 原子）

```java
// ✅ Redis 2.6.12+ 原生支持 SETNX + EX 原子
Boolean flag = redis.opsForValue().setIfAbsent("lock", "1", 10, TimeUnit.SECONDS);
```

SETNX + EX 合并成一条 SET 命令，原子性由 Redis 保证。

### 2.3 第三版：UUID 标识 + Lua 释放（✅ 防误删）

只解决原子性还不够，还要解决"误删别人的锁"。

**场景**：

```
线程 A 拿到锁（value=A），设 10 秒过期
doBusiness() 耗时 15 秒
→ 10 秒后锁过期 → 线程 B 抢到锁（value=B）
→ 线程 A 跑完，执行 redis.delete("lock") → 删的是 B 的锁！❌
```

**解法**：用 UUID 标识"这把锁是我的"，释放时先比对再删除，且**比对+删除必须是原子的**（Lua 脚本）。

```java
// 加锁：value 用 UUID
String uuid = UUID.randomUUID().toString();
Boolean flag = redis.opsForValue().setIfAbsent("lock", uuid, 10, TimeUnit.SECONDS);

// 释放锁：Lua 脚本保证"比对 + 删除"原子
String script =
    "if redis.call('get', KEYS[1]) == ARGV[1] then " +
    "    return redis.call('del', KEYS[1]) " +
    "else " +
    "    return 0 " +
    "end";

redis.execute(new DefaultRedisScript<>(script, Long.class),
              Collections.singletonList("lock"), uuid);
```

### 2.4 SimpleRedisLock 完整代码

```java
public class SimpleRedisLock implements Lock {

    private StringRedisTemplate stringRedisTemplate;
    private String name;   // 业务前缀

    private static final String ID_PREFIX = UUID.randomUUID().toString() + "-";

    public SimpleRedisLock(StringRedisTemplate stringRedisTemplate, String name) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.name = name;
    }

    // 当前线程的 UUID（static 保证线程间隔离）
    private static final ThreadLocal<String> THREAD_ID =
        ThreadLocal.withInitial(() -> ID_PREFIX + Thread.currentThread().getId());

    @Override
    public boolean tryLock(long timeoutSec) {
        // SETNX + EX 一句话
        Boolean flag = stringRedisTemplate.opsForValue()
                .setIfAbsent(KEY_PREFIX + name, THREAD_ID.get(), timeoutSec, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);
    }

    @Override
    public void unlock() {
        // Lua 脚本：先比对线程标识，再 DEL
        String script =
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        stringRedisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(KEY_PREFIX + name),
            THREAD_ID.get()
        );
    }
}
```

---

## 三、UUID 设计的两个细节

### 3.1 为什么用 `UUID + ThreadId` 而不只是 UUID

```java
private static final String ID_PREFIX = UUID.randomUUID().toString() + "-";

private static final ThreadLocal<String> THREAD_ID =
    ThreadLocal.withInitial(() -> ID_PREFIX + Thread.currentThread().getId());
```

**目的**：防止"同一 JVM 多线程用同一个 UUID 实例"造成的串号。UUID 前缀保证**跨进程不重复**，ThreadId 后缀保证**同一进程多线程不重复**——双重保险。

### 3.2 为什么用 ThreadLocal 而不是直接 Thread.currentThread().getId()

避免每次解锁都重新算 UUID 字符串，提升性能。同时也避免 ThreadLocalMap 复用线程时的"内存泄漏"风险。

---

## 四、Lua 脚本为什么必须

> **Lua 脚本在 Redis 里是单线程串行执行的，是 Redis 提供"原子操作"的官方方式。**

不用 Lua 的话，常规写法：

```java
String value = redis.get("lock");
if (value.equals(myUuid)) {   // ① 读
    redis.del("lock");        // ② 删
}
// ❌ ①② 之间可能被其他线程改了！读到的是别人的值，删的也是别人的锁！
```

用 Lua 把 ①② 包成一个整体 → **原子** → 这就是 Redis 提供事务/原子性的唯一靠谱方式（普通 `MULTI/EXEC` 太弱）。

---

## 五、可重入锁（进阶）

### 5.1 什么是可重入

**同一个线程可以多次获取同一把锁**而不死锁自己：

```java
lock.lock();        // 第一次获取，count=1
methodA();
lock.lock();        // 第二次获取，count=2
methodB();
lock.unlock();      // count=1
lock.unlock();      // count=0，真正释放
```

**好处**：递归调用 / 嵌套调用不会死锁。

### 5.2 数据结构选择

需要记录"谁持锁 + 持了几次"，自然选 **Hash**：

```
KEY: lock:order
HASH:
  field1: "uuid-123",  value1: "2"   ← 这把锁被 uuid-123 持有，count=2
```

### 5.3 加锁 Lua（先查 count 再决定 SET 还是 INCR）

```lua
-- KEYS[1] = 锁 key
-- ARGV[1] = 线程标识
-- ARGV[2] = 过期时间

if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hset', KEYS[1], ARGV[1], '1');
    redis.call('expire', KEYS[1], ARGV[2]);
    return 1;   -- 加锁成功
end;

if (redis.call('hexists', KEYS[1], ARGV[1]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[1], '1');
    redis.call('expire', KEYS[1], ARGV[2]);
    return 1;   -- 重入成功
end;

return 0;       -- 加锁失败
```

### 5.4 释放锁 Lua（count -1，归零才 DEL）

```lua
if (redis.call('hexists', KEYS[1], ARGV[1]) == 0) then
    return nil;   -- 不是你的锁
end;

local count = redis.call('hincrby', KEYS[1], ARGV[1], '-1');
if (count == 0) then
    redis.call('del', KEYS[1]);
    return 1;     -- 真正释放
end;
return 0;         -- 还不能释放
```

---

## 六、生产级方案：Redisson

自己写的 SimpleRedisLock 只是教学用，**生产环境必须用 Redisson**：

```java
RLock lock = redissonClient.getLock("lock:order");

lock.lock(10, TimeUnit.SECONDS);    // 10 秒业务时间
try {
    doBusiness();
} finally {
    lock.unlock();
}
```

Redisson 的核心特性：

| 特性 | 作用 |
| ---- | ---- |
| **看门狗自动续期** | 默认 30 秒过期，每 10 秒检查一次自动延长 |
| **可重入** | 基于 Hash 计数 |
| **公平/非公平锁** | 多种获取策略 |
| **联锁（MultiLock）** | 同时锁多个资源 |
| **读写锁** | 读写互斥、写写互斥、读读共享 |
| **Semaphore** | 限流 |
| **CountDownLatch** | 同步等待 |

---

## 七、一句话总结

| 演进版本 | 解决了什么 | 还差什么 |
| ---- | ---- | ---- |
| SETNX + 单独 EX | 互斥 | 不原子 → 死锁 |
| SETNX + EX 一句话 | 原子 | 还是会误删别人的锁 |
| UUID + Lua 释放 | 防误删 | 不支持重入 |
| Hash + Lua 计数 | 支持重入 | 看门狗续期 |
| **Redisson** | **企业级全套** | ✅ |

**核心思想**：Redis 分布式锁的精髓是"**用 UUID 标识归属 + 用 Lua 保证原子**"——自己写代码也是这个套路，生产用 Redisson。