# MyBatis-Plus 配置优先级与"约定优于配置"

> 这篇博客对应我学过的《TyporaFile/MybatisPlus/MybatisPlus笔记.md》配置相关。

## 一句话定位

**MyBatis-Plus 的"约定优于配置"思想不是魔法——是它按"字段名 → 默认值 → 你的配置"的顺序找最终值。** 理解这套优先级，你就能解释为什么数据库叫 `user` 就不用配 `@TableName`、为什么加个 YAML 配置就改了默认 ID 策略。

---

## 一、配置优先级链

> **属性优先级（从高到低）**

```
编程式配置（代码里 new Configuration()）
        ↓
自定义配置类（@Configuration 的 @Bean）
        ↓
application-{profile}.yml（指定环境的配置）
        ↓
application.yml（全局配置）
        ↓
MP 默认值（写在代码里的硬编码默认）
```

**一句话**：**用户的局部配置 > 全局配置 > 框架默认值**。

---

## 二、典型场景分析

### 2.1 ID 字段叫 `id` 就自动映射

**约定**：

```java
@Data
public class User {
    private Long id;        // ← 字段就叫 id，MP 默认就把它当主键
    private String name;
}
```

**等价于**：

```java
@Data
public class User {
    @TableId(type = IdType.ASSIGN_ID)   // ← MP 自动加的
    private Long id;
    private String name;
}
```

**根因**：MP 启动时会扫描实体类的字段名，遇到 `id` 就默认当成 `@TableId(type = ASSIGN_ID)` 处理。

### 2.2 表名 `tb_user` 也可以自动识别

```yaml
mybatis-plus:
  global-config:
    db-config:
      table-prefix: tb_       # 全局表前缀
```

```java
public class User {}            // → 自动映射到 tb_user 表
```

> **全局规则** + **类名** = 真实表名。

### 2.3 ID 策略全局改

```yaml
mybatis-plus:
  global-config:
    db-config:
      id-type: auto            # 全局改成 AUTO（数据库自增）
```

```java
public class User {
    private Long id;            // 不写 @TableId 也能用 AUTO 策略
}
```

**单个实体想覆盖全局**：

```java
@TableId(type = IdType.ASSIGN_ID)   // 局部配置优先级高于全局
private Long id;
```

---

## 三、Spring Boot 配置加载顺序

```
启动 Spring Boot
    ↓
读 application.yml / application.properties
    ↓
读 application-{profile}.yml（如 application-dev.yml）
    ↓
创建自定义 @Configuration 类（@Bean）
    ↓
初始化 MyBatis-Plus Configuration
    ↓
执行自定义的 MetaObjectHandler / 拦截器等
```

**重要规则**：

| 优先级 | 来源 |
| ---- | ---- |
| 最高 | `@PropertySource` 注解指定的额外配置文件 |
| ↑ | JVM 参数 `-Dxxx=yyy` |
| ↑ | 环境变量 `SPRING_DATASOURCE_URL` |
| ↑ | `application-{profile}.yml` |
| ↑ | `application.yml` |
| 最低 | 代码硬编码 |

---

## 四、典型配置实战

### 4.1 数据库连接

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp?serverTimezone=Asia/Shanghai
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
```

### 4.2 MyBatis-Plus 配置

```yaml
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true       # user_name → userName 自动转
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl   # SQL 日志
    cache-enabled: false
  global-config:
    db-config:
      id-type: ASSIGN_ID                     # ID 策略
      logic-delete-field: deleted            # 逻辑删除字段
      logic-delete-value: 1
      logic-not-delete-value: 0
      table-prefix: tb_                      # 表前缀
```

### 4.3 自定义拦截器（分页插件）

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

---

## 五、实战常见疑问

### 5.1 我改了 YAML 但没生效

**排查清单**：

1. ✅ YAML 缩进对了吗？（子属性是 2 空格缩进）
2. ✅ 是不是 `application-dev.yml` 没被激活？（检查 `spring.profiles.active`）
3. ✅ 是不是本地配置类用 `@ConfigurationProperties` 覆盖了？
4. ✅ 是不是 IDE 缓存了旧的 YAML？重新 `mvn clean` 试试

### 5.2 数据库自增和 `@TableId(AUTO)` 哪个生效

看生成链路优先级（前面 RedisIdWorker 篇讲过）：

```
实体有 ID？ → 用实体的
无 ID？ → @TableId 决定
         ├─ AUTO → 不塞，让 DB 自增
         └─ ASSIGN_ID → 雪花算法塞 Long
```

**YAML 全局 `id-type: AUTO` + 实体无 `@TableId` 注解** → DB 自增。
**YAML 全局 `id-type: AUTO` + 实体有 `@TableId(ASSIGN_ID)`** → 雪花算法（局部 > 全局）。

### 5.3 日志显示 SQL 但变量没替换

```yaml
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

```
==> Preparing: SELECT * FROM user WHERE id = ?
==> Parameters: 1(Integer)
```

`?` 占位符是预编译的——这是好事（防 SQL 注入），不是配置错。

---

## 六、"约定优于配置"的边界

| 场景 | MP 怎么处理 |
| ---- | ---- |
| 表名 `user` + 类名 `User` | ✅ 自动映射 |
| 字段名 `user_name` + 属性 `userName` | ✅ 自动转（默认开） |
| 字段名 `id` | ✅ 自动当主键 |
| 字段名 `create_time` + 属性 `createTime` | ✅ 自动转 |
| 字段名 `password` | ⚠️ 默认会被查出来（要手动 `select=false`） |
| 表名带前缀 `tb_user` + 类名 `User` | 配置 `table-prefix: tb_` |
| 逻辑删除字段叫 `is_deleted` | 配置 `logic-delete-field` |

> **约定是"合理默认值"，不是死规则**——遇到约定之外的，就在 YAML 或注解里覆盖。

---

## 七、一句话总结

| 概念 | 一句话 |
| ---- | ---- |
| **优先级链** | 局部 > 全局 > 框架默认 |
| **约定** | id 字段、user_name → userName、表名 = 类名 |
| **配置** | YAML 自定义规则 |
| **覆盖** | 注解优先级 > YAML > 框架默认 |
| **防注入** | `?` 占位符是预编译的，不是 bug |

**核心**：理解 MP 的"约定 + 配置 + 覆盖"三层结构，才能在项目中**灵活但不出错**。