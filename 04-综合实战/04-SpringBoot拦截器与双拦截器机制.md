# Spring Boot 拦截器与 MvcConfig：拦截器在实际项目里的完整链路

> 这篇博客把 Spring MVC 拦截器、MvcConfig 配置、双拦截器机制、登录态管理串成一条完整的链路。

## 一句话定位

**Spring MVC 拦截器 = "所有请求的统一入口"——Tomcat 把请求转给 Spring MVC 之后，DispatcherServlet 会按顺序过一遍注册的拦截器，再到 Controller。** 在实际项目里，用双拦截器（刷新拦截器 + 校验拦截器）实现了"自动续命 + 强制登录"的组合。

---

## 一、Tomcat → Spring MVC → 拦截器 → Controller

```
浏览器 → Nginx → Tomcat
    ↓ Tomcat 把请求交给 DispatcherServlet（Spring MVC 的前端控制器）
    ↓
DispatcherServlet 收到请求
    ↓
按 order 升序执行所有 HandlerInterceptor.preHandle()
    ├─ return true → 继续
    └─ return false → 后续都不执行
    ↓
HandlerMapping 找到对应的 Controller 方法
    ↓
Controller 处理业务（Service / Mapper / Redis）
    ↓
请求执行完后，按 order 倒序执行 afterCompletion()
    ↓
返回响应给浏览器
```

**拦截器是 Spring MVC 提供的"所有请求的统一入口"**——不管请求打到哪个 Controller，都先过拦截器。

---

## 二、HandlerInterceptor 三个方法

```java
public interface HandlerInterceptor {
    default boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
        return true;   // 1. Controller 之前执行，return false 直接拦截
    }

    default void postHandle(HttpServletRequest req, HttpServletResponse resp, Object handler, ModelAndView mav) {
        // 2. Controller 之后、视图渲染之前（现在前后端分离用得少）
    }

    default void afterCompletion(HttpServletRequest req, HttpServletResponse resp, Object handler, Exception ex) {
        // 3. 整个请求完毕后（用于清理资源，比如 ThreadLocal）
    }
}
```

**执行顺序**：

```
preHandle1 → preHandle2 → Controller
    ↓                  ↓
afterCompletion2  ← afterCompletion1   ← 倒序
```

---

## 三、双拦截器架构

### 3.1 为什么需要两个

| 拦截器 | 拦截路径 | 职责 |
| ---- | ---- | ---- |
| **RefreshTokenInterceptor** | `/**`（所有路径） | 刷新 Token 有效期（不强制登录） |
| **LoginInterceptor** | `/order/**` 等敏感路径 | 强制校验登录态 |

### 3.2 配置代码

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 拦截器 1：刷新拦截器（拦截一切路径，order=0）
        registry.addInterceptor(new RefreshTokenInterceptor(stringRedisTemplate))
                .addPathPatterns("/**")
                .order(0);

        // 拦截器 2：登录校验拦截器（只拦截敏感路径，order=1）
        registry.addInterceptor(new LoginInterceptor())
                .excludePathPatterns(
                    "/shop/**",
                    "/voucher/**",
                    "/shop-type/**",
                    "/upload/**",
                    "/blog/hot",
                    "/user/code",
                    "/user/login"
                )
                .order(1);
    }
}
```

### 3.3 拦截器 1：RefreshTokenInterceptor

```java
public class RefreshTokenInterceptor implements HandlerInterceptor {

    private StringRedisTemplate stringRedisTemplate;

    public RefreshTokenInterceptor(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 1. 拿 token
        String token = request.getHeader("authorization");
        if (token == null || token.isEmpty()) {
            return true;   // 没 token 直接放行（让拦截器 2 处理）
        }

        // 2. 查 Redis
        String key = LOGIN_USER_KEY + token;
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);
        if (userMap.isEmpty()) {
            return true;   // Redis 查不到也放行（让拦截器 2 拦截）
        }

        // 3. 把 userMap 转 UserDTO + 存 ThreadLocal
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
        UserHolder.saveUser(userDTO);

        // 4. 刷新 token 有效期（核心动作）
        stringRedisTemplate.expire(key, LOGIN_USER_TTL, TimeUnit.MINUTES);

        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        // 清理 ThreadLocal（防止下一位用户拿到上次的 user）
        UserHolder.removeUser();
    }
}
```

### 3.4 拦截器 2：LoginInterceptor

```java
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (UserHolder.getUser() == null) {
            response.setStatus(401);   // 告诉前端：未授权
            return false;              // 拦截
        }
        return true;
    }
}
```

### 3.5 工具类 UserHolder

```java
public class UserHolder {
    private static final ThreadLocal<UserDTO> tl = new ThreadLocal<>();

    public static void saveUser(UserDTO user) { tl.set(user); }
    public static UserDTO getUser() { return tl.get(); }
    public static void removeUser() { tl.remove(); }
}
```

---

## 四、双拦截器解决"30 分钟死局"

### 4.1 单拦截器的问题

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

### 4.2 双拦截器如何破局

```
0 分钟：登录成功
0-30 分钟：用户看商品
         → 拦截器 1（刷新）拦截 /* → 每隔几秒重置 Token 30 分钟
         → 用户一直在动 → Token 永远死不了
30 分钟：拦截器 1 又续命
30 分 01 秒：用户点"立即购买"
         → 拦截器 1 放行（user 已存 ThreadLocal）
         → 拦截器 2 校验 ThreadLocal → 有 user → 放行
         → 下单成功！✅
```

---

## 五、为什么用 ThreadLocal

> **登录只发生一次，而"获取当前用户"是每个请求都要做的事**——这两个场景根本不是一回事。

```
登录请求 → 线程A ThreadLocal 存 user → 请求结束
下单请求 → 线程B ThreadLocal 是空的 → 拿不到 ❌
```

**唯一能证明身份的就是请求头里的 token**，于是：

```
每个请求 → 拿 token → 去 Redis 查用户 → 存进**当前线程**的 ThreadLocal
   ↓
业务代码 UserHolder.getUser() 取用
```

---

## 六、UserHolder 存什么类型

```java
// 原始 Redis 数据
Map<Object, Object> userMap = { id=1, nickName="msc", icon="..." }

// 转换为 UserDTO（脱敏）
UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
UserHolder.saveUser(userDTO);
```

**为什么不直接存 User 实体**：

| 类型 | 字段 | 用途 |
| ---- | ---- | ---- |
| **User（数据库实体）** | phone, password, ... | 持久化 |
| **UserDTO（传输对象）** | id, nickName, icon | 网络传输、业务使用 |

**DTO 是"脱敏版本"**——不暴露 password 等敏感字段，同时只保留业务需要的字段。

---

## 七、拦截器里 Redis 返回值的两种处理

### 7.1 opsForHash().entries() 返回 Map

```java
Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries("login:token:xxx");
```

`opsForHash` 配合 `BeanUtil.fillBeanWithMap` 一行搞定 Map → Bean 转换。

### 7.2 opsForValue().get() 返回 String

```java
String json = stringRedisTemplate.opsForValue().get("shop:1");
Shop shop = JSONUtil.toBean(json, Shop.class);
```

`opsForValue` 用 `JSONUtil.toBean` 做 JSON → Bean 转换。

---

## 八、一句话总结

| 概念 | 一句话 |
| ---- | ---- |
| **拦截器** | Spring MVC 的统一请求入口，preHandle + afterCompletion 两头把关 |
| **双拦截器** | 刷新拦截器（/*）续命 + 校验拦截器（/order/**）拦截 |
| **MvcConfig** | 实现 WebMvcConfigurer，`addInterceptors()` 注册 |
| **构造器注入** | 拦截器是 `new` 的，依赖靠 MvcConfig（Spring Bean）注入再传 |
| **ThreadLocal** | 单请求内缓存 UserDTO，避免每次都查 Redis |
| **afterCompletion** | 清理 ThreadLocal，防止下一位用户串号 |

**整条链路的精髓**：把"用户身份"从"每次请求"+"每个 Controller"解耦出来，集中到拦截器做，业务代码只管 `UserHolder.getUser()`，干净不漏。