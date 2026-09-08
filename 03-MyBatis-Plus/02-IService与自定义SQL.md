# MyBatis-Plus IService 与自定义 SQL：消灭样板代码

> 这篇博客对应我学过的《TyporaFile/MybatisPlus/MybatisPlus笔记.md》核心功能部分。

## 一句话定位

**IService = 在 BaseMapper 的"单表 CRUD SQL"之上，把"Service 层的样板代码"也只写一遍。** 自定义 SQL = 当 MP 结构化 API 表达不了 SET 表达式时，把骨架放 Mapper、动态条件仍由 Wrapper 管的折中方案。

---

## 一、为什么还要 IService

### 1.1 疑问

> mp 中的 service 层提供了 IService 来实现 CRUD，我很奇怪明明 CRUD 在 basemapper 中就已经写好了，我直接调用 mapper 中的方法不就行了？service 调 mapper 不是已经很顺畅的逻辑吗？为啥还要再提供一次呢？

### 1.2 三个理由

**理由 1：消灭"每张表抄一遍"的样板代码（核心原因）**

没有 IService，20 张表就要手写 20 个 Service，每个都长一样：

```java
public User getById(Long id) { return userMapper.selectById(id); }
public void save(User u) { userMapper.insert(u); }
// × 20 张表，全是这种转发
```

泛型基类 `ServiceImpl<M, T>` 把这些只写一次，你只需：

```java
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements IUserService {}
// 两行，通用 CRUD 全部到手
```

**理由 2：有些方法 Mapper 层给不了，手写成本很高**

| 方法 | 实现成本 |
| ---- | ---- |
| `saveBatch` / `updateBatchById` | 要管理 `ExecutorType.BATCH` 的 SqlSession、分批 flush、包事务——几十行机制代码 |
| `saveOrUpdate` | 先查后判断再插/改的两步组合 + 事务 |
| `lambdaQuery().eq(...).one()` | 链式查询入口，免手写 Wrapper 样板 |

这些是"**流程/编排**"，不是"一条 SQL"，所以结构上只能放在 Service 层——IService 就是提前替你写好的那一版。

**理由 3：统一门面**

调用方统一依赖 `IService<T>` 接口：换实现、mock 测试、跨模块复用都有同一张脸。

### 1.3 与相关概念的边界

| 问题 | 答案 | 与 IService 的关系 |
| ---- | ---- | ---- |
| 为什么要有 Service 层？ | 事务、业务编排（@Transactional 只能标在 Spring Bean 上） | 自己手写 Service 就满足，**不需要 IService** |
| 为什么要有 Mapper？ | 只有 Mapper 这套代理能执行 SQL | IService 内部就是调它 |
| 为什么要有 IService？ | 省掉通用 Service 样板的重复劳动 | **可选便利件**，不 extends 自己写也完全正确 |

---

## 二、IService 的常用方法

```java
public interface UserService extends IService<User> {
    // IService 提供（部分）：
    boolean save(User entity);                       // 插入
    boolean saveBatch(Collection<User> entities);    // 批量插入
    boolean updateById(User entity);
    boolean removeById(Serializable id);
    User getById(Serializable id);
    List<User> listByIds(Collection<? extends Serializable> idList);
    long count();
    // ... 几十个
}
```

### 2.1 批量操作

```java
boolean saveBatch(Collection<T> entityList);          // 默认 1000 条一批
boolean saveBatch(Collection<T> entityList, int batchSize);
```

底层其实是循环 `Mapper.insert(...)`，但它帮你**自动处理批处理 + 事务**。

### 2.2 saveOrUpdate

```java
boolean saveOrUpdate(T entity);
// 内部逻辑：先 selectById，判断 id 是否存在 → 存在 update / 不存在 insert
```

### 2.3 链式查询

```java
List<User> list = userService.lambdaQuery()
    .eq(User::getName, "张三")
    .ge(User::getAge, 18)
    .like(User::getEmail, "163.com")
    .orderByDesc(User::getCreateTime)
    .list();
```

比起 `QueryWrapper<User>().eq(...).ge(...).list()`，lambda 版用方法引用代替字符串，更安全（编译期检查）。

---

## 三、LambdaQueryWrapper vs QueryWrapper

| 维度 | QueryWrapper | LambdaQueryWrapper |
| ---- | ---- | ---- |
| 字段名 | 字符串 `"name"`（拼错编译不报错） | 方法引用 `User::getName`（重命名编译报错） |
| 可读性 | 一般 | ✅ 更好 |
| 推荐场景 | 简单条件、动态 SQL 拼接 | **日常开发首选** |

```java
// ❌ 字符串版（拼错字段名也不报错）
new QueryWrapper<User>().eq("namae", "张三");

// ✅ Lambda 版（改名就编译报错）
new LambdaQueryWrapper<User>().eq(User::getName, "张三");
```

---

## 四、自定义 SQL：什么时候需要

### 4.1 触发场景

```sql
UPDATE user SET balance = balance - ? WHERE id IN (?, ?, ?)
```

**普通 Wrapper 的能力边界**：MP 结构化 API 能表达的 SET，**只有"列 = 常量"这一种形态**：

```java
luw.set(User::getBalance, 2001);   // SET balance = ?   ← 值是参数，OK
luw.set(User::getStatus, 1);       // SET status = ?    ← 同上
```

而 `SET balance = balance - ?` 的值那一侧出现了列名和减法，**API 表达不了**。

### 4.2 两条出路

**出路 A：`setSql` —— SQL 文本落在业务层**

```java
// Service 层
userMapper.update(null,
    new UpdateWrapper<User>()
        .setSql("balance = balance - " + money)   // ← 裸 SQL 字符串
        .in("id", ids));
```

**问题**：一条 UPDATE 语句的骨架被拆散在 Java 业务代码里。DBA 想排查"这笔扣减业务到底怎么改库存的"，得去翻 Service 源码——正是"SQL 维护在持久层"规范要防的事。

**出路 B：自定义 SQL —— SQL 骨架回持久层，动态条件仍由 Wrapper 管**

```java
// Mapper 接口
@Update("UPDATE user SET balance = balance - #{money} ${ew.customSqlSegment}")
void deductBalanceByIds(@Param("money") int money, @Param("ew") QueryWrapper<User> wrapper);

// Service 层（业务层）——只有条件，没有 SQL
QueryWrapper<User> wrapper = new QueryWrapper<User>().in("id", ids);
userMapper.deductBalanceByIds(200, wrapper);
```

**分工**：

| 部分 | 放哪 | 为什么 |
| ---- | ---- | ---- |
| `UPDATE user SET balance = balance - #{money}` | Mapper（持久层） | SQL 骨架 |
| `WHERE id IN (1,2,4)` | Wrapper（业务层） | 动态条件运行时才知道 |
| 桥接 | `${ew.customSqlSegment}` | Wrapper 渲染的 SQL 片段 |

### 4.3 `${ew.customSqlSegment}` 注入风险？

**没有**。注入的前提是"拼接的内容来自用户输入"，而 `customSqlSegment` 的内容是**框架根据 Wrapper 链生成的**，条件值走的是 `?` 预编译。`${}` 只是拼接"框架自己写的 SQL 片段"，和你手写死文本一个性质。

### 4.4 新版本第三条路（MP 3.5.7+）

```java
new UpdateWrapper<User>().setDecrBy(User::getBalance, 200).in("id", ids);
// SET balance = balance - ?  ← API 原生表达，不用任何 SQL 文本
```

> 这恰恰印证了整个逻辑：**凡是能被 API 结构化表达的，就轮不到裸 SQL 文本出场**。

---

## 五、自定义 SQL 的本质

**一句话**：凡是能被 API 结构化表达的，就用 API；表达不了的，自定义 SQL + Wrapper 是符合分层的容身之所。

---

## 六、一句话总结

| 概念 | 一句话 |
| ---- | ---- |
| **IService** | 在 BaseMapper 之上，省掉通用 Service 样板代码 |
| **ServiceImpl<M, T>** | 泛型基类，两行接入通用 CRUD |
| **LambdaQueryWrapper** | 用方法引用代替字符串，编译期检查 |
| **自定义 SQL** | API 表达不了的 SQL 骨架放 Mapper，条件仍由 Wrapper 管 |
| **${ew.customSqlSegment}** | 把 Wrapper 渲染成 SQL 文本拼进 Mapper（无注入风险） |

**核心思想**：MP 全套设计都围绕"**让程序员少写重复代码**"——IService 是消灭 Service 层重复，自定义 SQL 是给"API 表达不了的 SQL"指定规范出口。