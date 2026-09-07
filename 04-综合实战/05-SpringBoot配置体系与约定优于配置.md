# Spring Boot 配置体系：约定优于配置的"分层加载"

> 这篇博客把 Spring Boot 配置 + MyBatis-Plus 配置 + Redis 配置三个层面的配置优先级整合到一起，讲清楚"为什么我改了 YAML 不生效"这种经典问题。

## 一句话定位

**Spring Boot 的配置加载是分层级的：JVM 参数 → 环境变量 → application-{profile}.yml → application.yml → @Configuration 类 → 框架默认。** 用户优先级高于框架，局部优先级高于全局。

---

## 一、Spring Boot 配置体系全景

```
┌─────────────────────────────────────────────┐
│ 优先级最高                                  │
│   @PropertySource 注解指定的额外配置文件    │
│   JVM 参数（-Dxxx=yyy）                     │
│   环境变量（SPRING_DATASOURCE_URL）         │
│   命令行参数（--xxx=yyy）                    │
│   application-{profile}.yml（如 dev/prod）  │
│   application.yml                           │
│   @Configuration 类里的 @Bean               │
│   框架默认值                                │
│ 优先级最低                                  │
└─────────────────────────────────────────────┘
```

---

## 二、典型 application.yml 结构

```yaml
# application.yml
spring:
  profiles:
    active: dev                # 激活 dev 环境
  datasource:                  # 公共部分
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis-plus:                  # 全局公共
  global-config:
    db-config:
      logic-delete-field: deleted

redis:                         # 公共 Redis
  host: localhost
  port: 6379
```

```yaml
# application-dev.yml（开发环境）
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp_dev
    username: root
    password: 123456

redis:
  password: ""                 # 开发环境无密码
```

```yaml
# application-prod.yml（生产环境）
spring:
  datasource:
    url: jdbc:mysql://prod-mysql:3306/myapp_prod
    username: appuser
    password: ${MYSQL_PASSWORD}   # 从环境变量读

redis:
  password: ${REDIS_PASSWORD}
```

**核心规则**：

| 规则 | 说明 |
| ---- | ---- |
| **激活** | `spring.profiles.active=dev` 决定用哪个 profile |
| **合并** | `application.yml` + `application-dev.yml` 会合并，profile 覆盖公共 |
| **覆盖** | 同名配置，profile 优先级高于公共 |

---

## 三、配置类的优先级

### 3.1 @Configuration + @Bean

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // 设置 JSON 序列化器
        Jackson2JsonRedisSerializer<Object> json = new Jackson2JsonRedisSerializer<>(Object.class);
        template.setDefaultSerializer(json);
        return template;
    }
}
```

### 3.2 @ConfigurationProperties

```java
@Data
@Component
@ConfigurationProperties(prefix = "mybatis-plus")
public class MybatisPlusProperties {
    private GlobalConfig globalConfig = new GlobalConfig();
    private Configuration configuration = new Configuration();
}
```

**好处**：把 YAML 结构映射成 Java 对象，IDE 还能给提示。

### 3.3 三种配置方式对比

| 方式 | 适用场景 | 例子 |
| ---- | ---- | ---- |
| **application.yml** | 通用配置 | 数据源、Redis 地址 |
| **@Value("...")** | 单值注入 | `@Value("${server.port}")` |
| **@ConfigurationProperties** | 批量绑定 | MyBatis-Plus 大量配置 |

---

## 四、MyBatis-Plus 配置优先级

```
1. 编程式配置（new Configuration）
   ↓
2. 自定义 @Configuration 类的 @Bean
   ↓
3. application-{profile}.yml 里的 mybatis-plus.*
   ↓
4. application.yml 里的 mybatis-plus.*
   ↓
5. 框架默认值
```

**实战配置**：

```yaml
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true       # user_name → userName 自动转
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl   # SQL 日志
  global-config:
    db-config:
      id-type: ASSIGN_ID
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
      table-prefix: tb_
```

**想覆盖全局 ID 策略**：在实体类上写 `@TableId(type = AUTO)`——局部 > 全局。

---

## 五、Redis 配置实战

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password:                    # 没密码留空
    database: 0                  # 默认 0 号库
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
        max-wait: 100ms
```

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // String 序列化
        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        // JSON 序列化（值用 JSON）
        Jackson2JsonRedisSerializer<Object> jsonSerializer = new Jackson2JsonRedisSerializer<>(Object.class);

        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

**StringRedisTemplate vs RedisTemplate**：

| 类型 | 序列化方式 | 适用 |
| ---- | ---- | ---- |
| **StringRedisTemplate** | String（value 是 JSON 字符串） | ✅ 大多数缓存场景 |
| **RedisTemplate** | JDK 序列化（默认）/ JSON | 复杂对象序列化 |

---

## 六、Spring Boot 自动装配原理

> **约定**：引入依赖 → 自动装配 → 跑起来就用

### 6.1 启动流程

```
SpringApplication.run()
    ↓
读取 spring.factories（Spring Boot 2.x）
或 META-INF/spring/...AutoConfiguration.imports（3.x）
    ↓
加载所有 *AutoConfiguration 类
    ↓
按 @Conditional 注解判断是否生效
    ↓
创建对应 Bean（DataSource、RedisTemplate、SqlSessionFactory...）
```

### 6.2 经典 Conditional

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)   // 有 Redis 类才生效
@ConditionalOnMissingBean(RedisTemplate.class) // 用户没自定义才生效
public class RedisAutoConfiguration {
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(...) { ... }
}
```

---

## 七、经典问题排查

### 7.1 改了 YAML 但没生效

**排查清单**：

```
1. 检查 application.yml 缩进（子属性 2 空格）
2. 检查 spring.profiles.active（不激活 profile 就不生效）
3. 检查是否有 @Configuration 类用 @Bean 覆盖了
4. 检查 IDE 是否缓存（mvn clean 试试）
5. 检查 application-{profile}.yml 是否被正确加载
```

### 7.2 改了 yml 但 ID 仍然是数据库自增

```yaml
mybatis-plus:
  global-config:
    db-config:
      id-type: ASSIGN_ID
```

但代码里：

```java
@TableId(type = IdType.AUTO)   // ← 这个优先级 > yml
private Long id;
```

**局部覆盖了全局**——这就是"约定优于配置"的双刃剑。

### 7.3 Redis 连不上

```yaml
spring:
  redis:
    host: 127.0.0.1     # 别忘了写
    port: 6379          # 默认端口
```

如果是远程 Redis：

```yaml
spring:
  redis:
    host: 192.168.1.100
    port: 6379
    password: your_password
    timeout: 5000ms     # 网络不好加大超时
```

---

## 八、约定优于配置的边界

| 场景 | Spring Boot 怎么处理 |
| ---- | ---- |
| 引入 `spring-boot-starter-data-redis` | 自动配置 RedisTemplate |
| 引入 `mybatis-plus-boot-starter` | 自动配置 SqlSessionFactory + MapperScanner |
| 引入 `spring-boot-starter-web` | 自动配置 DispatcherServlet、嵌入式 Tomcat |
| application.yml 没写 server.port | 默认 8080 |
| application.yml 没写 spring.datasource | 默认连 H2 嵌入式数据库 |

> **约定的魔力**：依赖加进来 + 配置写好 → 项目就跑起来，零 Java 配置。

---

## 九、一句话总结

| 概念 | 一句话 |
| ---- | ---- |
| **配置加载顺序** | 命令行 > 环境变量 > profile yml > application.yml > @Configuration > 默认 |
| **profile 切换** | `spring.profiles.active=dev/prod` 切换环境 |
| **@ConfigurationProperties** | 把 YAML 结构映射成 Java 对象 |
| **MyBatis-Plus 优先级** | 局部注解 > 全局 yml > 框架默认 |
| **Redis 配置** | `spring.redis.*` + 自定义 RedisTemplate Bean |
| **自动装配** | 依赖 + @Conditional = 零配置跑起来 |

**核心**：理解这套"分层 + 优先级"的配置体系，才能在 Spring Boot 项目里**改一个配置心里有数**，而不是"改完没生效就乱试"。