# Java 进阶融合：泛型 + Lambda + Stream + 内部类的"四位一体"

> 这篇博客把我学过的 Java 进阶四个核心概念串到一起——它们不是孤立的语法，而是同一套"函数式编程"理念在不同维度的落地。

## 一句话定位

**Java 8 的函数式编程是一次系统升级：泛型提供类型约束，Lambda 提供行为传递，内部类是 Lambda 的前身，Stream 是 Lambda 的流水线化。** 四个概念组合起来用，代码会从"啰嗦的过程式"变成"简洁的声明式"。

---

## 一、四者的关系网

```
泛型（Generic）
   ↓ 为"通用代码"提供类型安全
   ↓
集合 / 函数式接口（Function<T, R> / Consumer<T>）
   ↓
Lambda 表达式（行为参数化）
   ↓
方法引用（Lambda 的语法糖）
   ↓
Stream 流（Lambda 的流水线化）
   ↓
内部类（Lambda 出现前，传行为的唯一方式）
```

---

## 二、场景：List 过滤 "张" 开头且长度为 3 的元素

### 2.1 远古写法（Java 7 之前）

```java
List<String> list = Arrays.asList("张三丰","张无忌","张翠山","王二麻子","张良","谢广坤");
List<String> result = new ArrayList<>();
for (String s : list) {
    if (s.startsWith("张") && s.length() == 3) {
        result.add(s);
    }
}
```

**问题**：声明了三个变量（list / result / s）、三个语义步骤（遍历、判断、收集），混在一起。

### 2.2 Lambda + 函数式接口写法

```java
Predicate<String> startsWithZhang = s -> s.startsWith("张");
Predicate<String> lengthIs3 = s -> s.length() == 3;
list.stream()
    .filter(startsWithZhang.and(lengthIs3))
    .collect(Collectors.toList());
```

**核心概念**：

| 概念 | 在这一句中的作用 |
| ---- | ---- |
| **泛型** | `Predicate<T>`、`<T>` 让一个接口能处理任何类型 |
| **Lambda** | `s -> s.startsWith("张")` 是 Predicate 接口 `test` 方法的实现 |
| **方法引用** | `Collectors::toList` 代替 `Collectors.toList()` |
| **Stream** | `.filter().collect()` 链式调用 |

---

## 三、泛型：让通用代码不丢失类型

### 3.1 没有泛型时

```java
// 远古 API：只能返回 Object
List list = new ArrayList();
list.add("msc");
String s = (String) list.get(0);   // 强转，运行时才检查
```

### 3.2 加了泛型后

```java
List<String> list = new ArrayList<>();
list.add(123);                     // 编译报错
String s = list.get(0);            // 直接是 String
```

### 3.3 泛型方法 + Lambda 的组合

```java
public static <T> List<T> filter(List<T> list, Predicate<T> predicate) {
    List<T> result = new ArrayList<>();
    for (T item : list) {
        if (predicate.test(item)) {
            result.add(item);
        }
    }
    return result;
}

// 调用
List<String> filtered = filter(names, s -> s.startsWith("张"));
```

**`<T>` 在方法签名上的泛型声明 + `Predicate<T>` 在参数上** —— 这就是 Java 函数式 API 的基本骨架。

---

## 四、内部类：Lambda 的前身

### 4.1 写一个自定义函数式接口

```java
@FunctionalInterface
public interface MyFilter<T> {
    boolean test(T t);
}
```

### 4.2 匿名内部类实现（Lambda 之前）

```java
List<String> result = filter(names, new MyFilter<String>() {
    @Override
    public boolean test(String s) {
        return s.startsWith("张");
    }
});
```

### 4.3 Lambda 写法（一行）

```java
List<String> result = filter(names, s -> s.startsWith("张"));
```

**演进路径**：

```
6 行（匿名内部类）→ 2 行（方法引用）→ 1 行（Lambda）
```

---

## 五、Lambda 表达式：行为参数化

### 5.1 函数式接口三条件

```java
@FunctionalInterface   // 1. 必须是接口
public interface MyFunction<T, R> {   // 2. 只能一个抽象方法
    R apply(T t);
    default R applyDefault(T t) { return apply(t); }   // default 不算
}
```

### 5.2 Lambda 语法形态

| 形态 | 示例 |
| ---- | ---- |
| 无参 | `() -> System.out.println("hi")` |
| 单参 | `s -> s.length() > 3` |
| 多参 | `(a, b) -> a + b` |
| 多行 | `(a, b) -> { ... return result; }` |
| 类型推断 | `(String s, Integer n) -> ...`（一般不用写） |

### 5.3 this 指向（与匿名内部类的核心区别）

```java
public class Outer {
    void test() {
        Runnable r1 = new Runnable() {
            @Override public void run() {
                System.out.println(this);    // Outer$1@xxx（匿名类自身）
            }
        };

        Runnable r2 = () -> {
            System.out.println(this);         // Outer@xxx（外部类！）
        };
    }
}
```

> Lambda **不是**内部类的语法糖——它底层用的是 `invokedynamic` 指令，没有 `.class` 文件。

---

## 六、方法引用：Lambda 的语法糖

### 6.1 三种引用形式

```java
// 1. 静态方法引用
list.stream().map(Integer::parseInt);

// 2. 实例方法引用
list.stream().forEach(System.out::println);

// 3. 构造方法引用
list.stream().map(Student::new);
```

### 6.2 与 Lambda 的对应

```java
s -> Integer.parseInt(s)        等价于    Integer::parseInt
s -> System.out.println(s)      等价于    System.out::println
() -> new ArrayList()           等价于    ArrayList::new
```

---

## 七、Stream 流：Lambda 的流水线

### 7.1 三段式

```
获取流（1 次）→ 中间方法（多次，懒加载）→ 终结方法（1 次，触发执行）
```

### 7.2 常用操作

| 类型 | 方法 | 说明 |
| ---- | ---- | ---- |
| 中间 | `filter(Predicate)` | 过滤 |
| 中间 | `map(Function)` | 转换 |
| 中间 | `sorted(Comparator)` | 排序 |
| 中间 | `distinct()` | 去重 |
| 中间 | `limit(n)` | 截取 |
| 中间 | `skip(n)` | 跳过 |
| 中间 | `concat(s1, s2)` | 合并 |
| 终结 | `forEach(Consumer)` | 遍历 |
| 终结 | `collect(Collector)` | 收集 |
| 终结 | `count()` | 计数 |
| 终结 | `reduce(BinaryOperator)` | 归约 |

### 7.3 实战

```java
List<String> result = list.stream()
    .filter(s -> s.startsWith("张"))          // Predicate<String>
    .filter(s -> s.length() == 3)            // Predicate<String>
    .map(s -> s + "_processed")              // Function<String, String>
    .sorted(Comparator.comparingInt(String::length).reversed())  // 方法引用
    .limit(10)
    .collect(Collectors.toList());           // 终结
```

---

## 八、不可变集合 + Stream：防御性编程

```java
List<String> names = List.of("张三", "李四", "王五");   // 不可变
// names.add("aaa");   // ❌ 编译报错
// names.set(0, "赵六"); // ❌ 编译报错

names.stream().filter(s -> s.length() == 2).forEach(System.out::println);
```

**为什么用 `List.of()` 而不是 `new ArrayList<>()`**：

| 场景 | 推荐 |
| ---- | ---- |
| 数据只读 | `List.of(...)` |
| 数据要修改 | `new ArrayList<>(...)` |
| 数据要传给不可信库 | `List.copyOf()` / `Map.copyOf()` |

---

## 九、四大概念在项目里的实际出现

| 概念 | 出现位置 |
| ---- | ---- |
| **泛型** | `BaseMapper<User>`、`ServiceImpl<UserMapper, User>`、`RedisData<T>` |
| **内部类** | 拦截器是 Spring MVC 的 `HandlerInterceptor` 匿名内部类实现 |
| **Lambda** | `userMapper.selectList(wrapper)` 的参数、`Runnable`、`Consumer` |
| **方法引用** | `BeanUtil.fillBeanWithMap`、`Collectors::toList`、`User::getName` |
| **Stream** | 业务代码里大量 `list.stream().filter().map().collect()` |
| **不可变集合** | `Map.of("code", 200, "msg", "ok")` |

---

## 十、一句话总结

| 概念 | 解决的问题 | 关键 API |
| ---- | ---- | ---- |
| **泛型** | 编译期类型检查 + 避免强转 | `List<T>`、`<T extends X>` |
| **内部类** | 一段行为只在一个地方用 | 匿名内部类 |
| **Lambda** | 一段行为多次用，简洁 | `s -> s.startsWith("张")` |
| **方法引用** | Lambda 已有现成方法时更简洁 | `String::length` |
| **Stream** | 集合操作流水线化 | `filter().map().collect()` |
| **不可变集合** | 数据安全 | `List.of()`、`Map.copyOf()` |

**核心**：这套"四位一体"是 Java 8 之后所有现代代码的基石——学完了之后看任何框架源码（Spring / MyBatis-Plus / Stream）都不应该害怕，因为它们都在用这一套。