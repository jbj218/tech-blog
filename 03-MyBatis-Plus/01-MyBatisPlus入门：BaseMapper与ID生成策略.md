# MyBatis-Plus 入门：BaseMapper 与 ID 生成策略

> 这篇博客对应我学过的《TyporaFile/MybatisPlus/MybatisPlus笔记.md》开头部分。

## 一句话定位

**MyBatis-Plus（MP）是 MyBatis 的"增强版"——只做增强不做改变。** 引入它之后，单表 CRUD 不用写一行 SQL，ID 生成、条件构造、分页、逻辑删除全都开箱即用。

---

## 一、为什么需要 MyBatis-Plus

### 1.1 原生 MyBatis 的痛点

```xml
<!-- UserMapper.xml -->
<select id="selectById" resultType="User">
    SELECT * FROM user WHERE id = #{id}
</select>
<insert id="insert" parameterType="User">
    INSERT INTO user (name, age) VALUES (#{name}, #{age})
</insert>
<update id="updateById">
    UPDATE user SET name=#{name}, age=#{age} WHERE id=#{id}
</update>
<delete id="deleteById">
    DELETE FROM user WHERE id=#{id}
</delete>
```

**每张表都要写一遍这 4 个方法**，20 张表 = 80 个 SQL 片段，纯粹复制粘贴。

### 1.2 MP 的解决方案

```java
// 继承 BaseMapper 后，CRUD 全有了
public interface UserMapper extends BaseMapper<User> {
}

// 直接用
userMapper.selectById(1L);
userMapper.insert(user);
userMapper.updateById(user);
userMapper.deleteById(1L);
```

**零 SQL、零 XML、单表 CRUD 全到位**。

---

## 二、BaseMapper 的核心方法

| 方法 | 作用 |
| ---- | ---- |
| `insert(T entity)` | 插入一条记录 |
| `deleteById(Serializable id)` | 按 ID 删除 |
| `delete(Wrapper<T> wrapper)` | 按条件删除 |
| `updateById(T entity)` | 按 ID 更新 |
| `update(T entity, Wrapper<T> wrapper)` | 按条件更新 |
| `selectById(Serializable id)` | 按 ID 查询 |
| `selectList(Wrapper<T> wrapper)` | 按条件查列表 |
| `selectCount(Wrapper<T> wrapper)` | 按条件查总数 |
| `selectPage(IPage<T> page, Wrapper<T> wrapper)` | 分页查询 |

---

## 三、常用注解

### 3.1 `@TableName`

```java
@TableName("tb_user")   // 默认映射到 user 表，加上就是 tb_user
public class User { ... }
```

### 3.2 `@TableId`：ID 生成策略

```java
@TableId(type = IdType.ASSIGN_ID)   // 默认雪花算法 Long ID
private Long id;

@TableId(type = IdType.AUTO)        // 数据库自增
private Long id;

@TableId(type = IdType.INPUT)       // 手动赋值
private Long id;
```

| 策略 | 说明 |
| ---- | ---- |
| `AUTO` | 数据库自增（依赖 DB 的 `AUTO_INCREMENT`） |
| `ASSIGN_ID` | MP 默认，雪花算法生成 Long ID |
| `INPUT` | 手动赋值（自己生成） |
| `ASSIGN_UUID` | UUID 字符串（不推荐用于索引） |
| `NONE` | 不设策略，等于 INPUT |

### 3.3 `@TableField`

```java
@TableField("user_name")             // 数据库字段名跟属性名不一致
private String userName;

@TableField(exist = false)           // 表示这个字段不映射到数据库
private String temp;

@TableField(select = false)          // 查询时不查这个字段（密码字段常用）
private String password;

@TableField(fill = FieldFill.INSERT) // 插入时自动填充
private LocalDateTime createTime;
```

---

## 四、ID 生成问题的深度拆解

> **数据库自增和 MyBatis-Plus 同时存在，会冲突吗？**

**结论：不会报错，但会"打架"，最终谁生效取决于 `@TableId` 的配置。**

### 4.1 两个角色

| 角色 | 机制 | 生效时机 |
| ---- | ---- | ---- |
| **数据库 `AUTO_INCREMENT`** | 不传 ID 时由 DB 生成 | SQL 执行时 |
| **MyBatis-Plus `@TableId`** | 拼 SQL 前决定要不要塞 ID | 拦截器/填充阶段 |

**关键点**：MyBatis-Plus 的策略是**前置**的——它在拼 SQL 之前就已经决定要不要给对象塞 ID。

### 4.2 ID 生成优先级链路

```
实体对象是否有 ID？
    ├─ 有（用户手动 set）→ 直接用
    └─ 无 / null
         ↓
   @TableId 是否配置了 ID 策略？
         ├─ ASSIGN_ID → 雪花算法生成 Long
         ├─ AUTO → 不设 ID，留给 DB
         └─ INPUT → 用户自己处理
              ↓
   到数据库时，id 字段是 null 吗？
         ├─ 是 → 走 DB 自增规则
         └─ 否 → 用已有的 ID
```

**一句话总结**：数据库自增是"你不给我就给"，MyBatis-Plus 是"我在发 SQL 前就先决定给不给"。如果框架决定给了，数据库就没有出手的机会。

---

## 五、自动填充：createTime / updateTime

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```

实体类：

```java
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;
```

---

## 六、逻辑删除

### 6.1 配置

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted   # 全局逻辑删除字段
      logic-delete-value: 1         # 已删除值
      logic-not-delete-value: 0     # 未删除值
```

### 6.2 实体

```java
@TableLogic
private Integer deleted;
```

### 6.3 效果

`userMapper.deleteById(1L)` 不会真的删，而是 `UPDATE user SET deleted=1 WHERE id=1`。

`selectList(...)` 自动加 `WHERE deleted=0` 过滤。

---

## 七、一句话总结

| 概念 | 一句话 |
| ---- | ---- |
| **BaseMapper** | 继承即得单表 CRUD，零 SQL |
| **@TableId** | ID 生成策略（雪花/自增/手动） |
| **@TableField** | 字段映射规则（改名、不查、填充） |
| **ID 生成优先级** | MP 前置决策 > 数据库自增 |
| **自动填充** | `MetaObjectHandler` 统一处理 createTime/updateTime |
| **逻辑删除** | `@TableLogic` + 全局配置，查询自动加 WHERE deleted=0 |

**核心**：MP 的设计哲学是**"约定优于配置"**——按它的命名约定（如 `id` / `create_time`）几乎零配置可用。