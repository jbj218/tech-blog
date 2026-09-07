# Redis 短信登录：Session → Token → 双拦截器

> 这篇博客对应我学过的《Redis 实战笔记》短信登录部分。

## 一句话定位

**从 Session 到 Redis 共享 Session，再到 Token + 双拦截器，本质都是为了解决"HTTP 无状态 + 集群下身份共享"的问题。** 这套方案是后续所有缓存、秒杀、分布式锁的"登录态基础"。

---

## 一、问题起源：HTTP 无状态

> **HTTP 协议天生就是无状态的。** 服务器眼里，每个请求都是一封自包含的信：
>
> ```
> POST /orders   请求头: authorization: abc123
> ```

服务器处理完就忘，下一个请求来了又从零开始。所以"我登录过"这个状态，服务器从不保存——靠的是**每个请求都带着 token、每个请求都被拦截器独立验证一次**。

> 登录 = 办会员卡（一次）。验证 = 每次进店刷卡（每次，但刷卡只要一秒）。

---

## 二、方案演进

### 2.1 Session 方案

```
用户登录 → Tomcat 自动生成 JSESSIONID Cookie → 后端 Session 存用户
    ↓
下次请求自动带上 JSESSIONID → Tomcat 用它从 Session 取用户
```

**问题**：集群下每台 Tomcat 的 Session 是独立的，用户请求落到不同机器就找不到身份。

### 2.2 Redis 共享 Session

把 Session 数据从 Tomcat 内存搬到 Redis，所有 Tomcat 共享。

### 2.3 Token 方案（最终方案）

完全不用 Session，自己生成一个随机字符串（token）当"凭证"，存 Redis 里 → **返回给前端** → 前端每次请求手动带上（一般放在 `authorization` 请求头）。

---

## 三、为什么 Redis 方案要手动返回 token

```java
// Session 方案
public Result login(...) {
    session.setAttribute("user", user);
    return Result.ok();   // ✅ 不需要返回啥，JSESSIONID Tomcat 自动管
}

// Redis 方案
public Result login(...) {
    String token = UUID.randomUUID().toString();
    stringRedisTemplate.opsForValue().set("login:token:" + token, user, 30, TimeUnit.MINUTES);
    return Result.ok(token);   // ← 必须返回给前端！
}
```

| 方案 | 凭证传递 | 后端返回值 |
| ---- | ---- | ---- |
| **Session** | Tomcat 自动创建 JSESSIONID Cookie，前端无感知 | `Result.ok()` 即可 |
| **Redis Token** | 手动生成的字符串，没有任何自动传递机制 | **必须 `Result.ok(token)`**，前端手动存 + 请求头带 |

---

## 四、双拦截器机制

> **目标**：看商品是自由的（不强制登录），但想买就得先交"通行证"。

### 4.1 单拦截器的"30 分钟死局"

```
0 分钟：登录成功，Token 有效期 30 分钟（存 Redis）
0-30 分钟：用户只看商品页面（/shop/**）
         → 拦截器没被触发（因为只拦截 /order/**）
         → Token 在 Redis 里默默走字，没人续命
30 分钟：Redis 里 Token 准时过期
30 分 01 秒：用户点"立即购买"（/order/**）
         → 拦截器查 Redis → 查不到！
         → 强制踢出登录 ❌
```

**死结**：你虽然一直"在线"看商品，但因为没触发拦截器，系统就认为你"离线"了。

### 4.2 双拦截器如何破局

```
拦截器 1（刷新拦截器，拦截一切路径 /*，order=0）
    → 不管访问啥页面，都执行"有 Token 就续命"
    → 用户每点一次，Token 重置为 30 分钟
    → 用户一直滑动 → Token 永远死不了

拦截器 2（校验拦截器，只拦截 /order/**，order=1）
    → 查 ThreadLocal 是否有用户
    → 有就放行，没就返回 401
```

**未登录用户**：

```
1. 阶段一：自由浏览
   → 访问 /shop/** → 拦截器 1 没 Token，直接放行
   → 拦截器 2 不拦此路径，也放行
   → ThreadLocal 是空的
2. 阶段二：点"立即购买"（/order/add）
   → 拦截器 1 没 Token，直接放行
   → 拦截器 2 检查 ThreadLocal 为空 → 返回 401
   → 前端收到 401 → 自动跳转登录页
```

---

## 五、为什么用 ThreadLocal

**核心矛盾**：登录只发生一次，而"获取当前用户"是每个请求都要做的事。

> HTTP 无状态意味着：登录那一刻确实可以把 user 存进 ThreadLocal，但登录请求结束后这个存储就随着请求结束了。之后的请求是全新的请求，Tomcat 用线程池复用线程——分到的线程很可能不是登录那个线程，而 ThreadLocal 是线程私有的，别的线程拿不到。

```
登录请求 → 线程A ThreadLocal 存 user → 请求结束
下单请求 → 线程B ThreadLocal 是空的 → 拿不到 ❌
```

**正确流程**：每个请求都重新确认"我是谁"。唯一能证明身份的就是请求头里的 token，于是：

```
每个请求 → 拿 token → 去 Redis 查用户 → 存进**当前线程**的 ThreadLocal
   ↓
业务代码里随时 UserHolder.getUser() 取用
```

**为什么放拦截器而不是别的地方**：

| 方案 | 问题 |
| ---- | ---- |
| 登录时存 | 只对登录那一次请求有效 |
| 静态变量共享 | 所有线程共用一个用户，并发下用户串号 |
| 每个 Controller 自己查 | 每个接口都抄一遍，重复代码泛滥 |
| **拦截器统一做** | ✅ 所有请求自动经过，写一次全生效 |

---

## 六、三个角色的分工

| 角色 | 职责 | 生命周期 |
| ---- | ---- | ---- |
| **token** | 证明"我是谁" | 跨请求，长期（存 Redis，30 分钟滑动） |
| **Redis 里的用户信息** | token 对应的真实数据 | 跨请求，长期 |
| **ThreadLocal 里的 UserDTO** | 一次请求内的便利缓存 | 一次请求，用完就删（afterCompletion 清） |

> 没有 ThreadLocal 也能活——每个 Service 里都自己拿 token 查 Redis 就行，只是要写几百遍重复代码。ThreadLocal 纯粹是为了方便。

---

## 七、拦截器里的"间接注入"

### 7.1 为什么拦截器不能直接 `@Autowired`

拦截器是 `new` 出来的，**不在 Spring 容器里**，所以 `@Autowired` 不生效（运行时为 null）。

### 7.2 标准做法（课程中用的）

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;   // MvcConfig 是 @Configuration，是 Spring Bean

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 通过构造器把 Redis 传给 new 出来的拦截器
        registry.addInterceptor(new LoginInterceptor(stringRedisTemplate))
                .addPathPatterns("/**").order(1);
    }
}
```

**规律**：所有 `new` 出来的非 Spring 管理组件，要拿依赖都靠"宿主 Bean 注入 → 构造器传参"这条链。

---

## 八、@Component 方案行不行？

可以，但**前提是 MvcConfig 里不再 `new`，而是从 Spring 注入**：

```java
@Component
public class RefreshTokenInterceptor implements HandlerInterceptor {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
}

@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Autowired
    private RefreshTokenInterceptor refreshTokenInterceptor;   // 注入，不再 new

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(refreshTokenInterceptor).addPathPatterns("/**").order(0);
    }
}
```

**两种方案不能同时出现**（要么全 `new` + 构造器传，要么全 `@Component` + 注入）。

**标准做法还是构造器方案**：改动少、拦截器保持纯净、不耦合 Spring。