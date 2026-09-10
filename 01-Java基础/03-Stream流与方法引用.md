# Stream 流 + 方法引用：函数式编程在 Java 里的落地

> 这篇博客对应我学过的《TyporaFile/Stream/stream流和方法引用.md》。

## 一句话定位

**Stream 流 = 把"集合操作"变成"流水线装配"，方法引用 = 把"已有方法"当成 Lambda 用。** 两者都是 Java 8 引入的函数式编程语法糖，核心是把"做什么"和"怎么做"分开。

---

## 一、为什么要用 Stream

**案例**：从集合里筛出"姓张且长度为 3"的元素并打印。

传统写法：

```java
ArrayList<String> list1 = new ArrayList<>(List.of("张三丰","张无忌","张翠山","王二麻子","张良","谢广坤"));
ArrayList<String> list2 = new ArrayList<>();
for (String s : list1) {
    if(s.startsWith("张")){
        list2.add(s);
    }
}
ArrayList<String> list3 = new ArrayList<>();
for (String s : list2) {
    if(s.length() == 3){
        list3.add(s);
    }
}
for (String s : list3) {
    System.out.println(s);
}
```

Stream 写法（一句话）：

```java
list1.stream()
     .filter(s -> s.startsWith("张"))      // 过滤姓张
     .filter(s -> s.length() == 3)        // 过滤长度为 3
     .forEach(s -> System.out.println(s)); // 打印
```

**Stream 的好处**：直接阅读字面意思即可理解逻辑（获取流 → 过滤姓张 → 过滤长度 3 → 打印），代码简洁，把真正的函数式编程风格引入到 Java 中。

---

## 二、Stream 的三类方法

| 方法类型 | 作用 | 次数 |
| ---- | ---- | ---- |
| **获取 Stream 流** | 创建一条流水线，把数据放上去准备操作 | 1 次 |
| **中间方法** | 流水线上的操作 | 可以多次（懒加载） |
| **终结方法** | 流水线最后一个操作，触发整个流程执行 | 只能 1 次 |

> 一个 Stream 流只能有一个终结方法。

---

## 三、生成 Stream 的 4 种方式

```java
// 1. Collection 体系：使用默认方法 stream()
List<String> list = new ArrayList<>();
Stream<String> listStream = list.stream();

// 2. Map 体系：转成 Set 集合再间接生成流
Map<String,Integer> map = new HashMap<>();
Stream<String> keyStream = map.keySet().stream();
Stream<Integer> valueStream = map.values().stream();
Stream<Map.Entry<String, Integer>> entryStream = map.entrySet().stream();

// 3. 数组：Arrays.stream()
String[] strArray = {"hello","world","java"};
Stream<String> strArrayStream = Arrays.stream(strArray);

// 4. 同种数据类型的多个数据：Stream.of(T... values)
Stream<String> s2 = Stream.of("hello", "world", "java");
Stream<Integer> intStream = Stream.of(10, 20, 30);
```

---

## 四、Stream 中间操作方法

| 方法名 | 说明 |
| ---- | ---- |
| `Stream<T> filter(Predicate)` | 对流中的数据进行过滤 |
| `Stream<T> limit(long maxSize)` | 截取前 maxSize 个 |
| `Stream<T> skip(long n)` | 跳过前 n 个 |
| `static <T> Stream<T> concat(Stream a, Stream b)` | 合并 a 和 b |
| `Stream<T> distinct()` | 去重（按 `equals`） |

> 关键特征：执行完中间方法之后，Stream 流依然可以继续执行其他操作。

---

## 五、方法引用（Lambda 的语法糖）

**本质**：Lambda 体里已经有现成的方法可以代替你写的逻辑，直接用 `::` 引用这个方法，让代码更简洁。

### 5.1 几种引用形式

| 形式 | 写法 | 等价的 Lambda |
| ---- | ---- | ---- |
| 静态方法引用 | `类名::静态方法名` | `(args) -> 类名.静态方法(args)` |
| 实例方法引用 | `对象::实例方法名` | `(args) -> 对象.实例方法(args)` |
| 特定类型方法引用 | `类名::实例方法名` | `(obj, args) -> obj.实例方法(args)` |
| 构造方法引用 | `类名::new` | `(args) -> new 类名(args)` |

### 5.2 示例

```java
// 1. 静态方法引用
list.stream().map(Integer::parseInt).forEach(System.out::println);
// 等价于：list.stream().map(s -> Integer.parseInt(s))

// 2. 实例方法引用（对象已存在）
list.stream().forEach(System.out::println);
// 等价于：list.stream().forEach(s -> System.out.println(s))

// 3. 构造方法引用
list.stream().map(Student::new).collect(Collectors.toList());
// 等价于：list.stream().map(s -> new Student(s))
```

---

## 六、Stream + 不可变集合

Stream 经常和"不可变集合"一起用——先用 `List.of()` / `Set.of()` / `Map.of()` 创建只读集合，再流式处理，避免中途被改动。

```java
// 不可变的 List + Stream
List<String> list = List.of("张三", "李四", "王五", "赵六");
list.stream().filter(s -> s.length() == 2).forEach(System.out::println);
// list.add("aaa");  // ❌ 编译报错
// list.set(0, "aaa"); // ❌ 编译报错（不可变）
```

**不可变集合的细节**：

- `Map.of()` 最多传 20 个参数（10 个键值对）
- 超过 10 个键值对要用 `Map.ofEntries()` 或 `Map.copyOf()`
- `Set.of()` 参数要保证唯一性

---

## 七、总结

| 概念 | 一句话 |
| ---- | ---- |
| **Stream** | 把集合操作串成流水线，filter/map/forEach 链式调用 |
| **方法引用** | `::` 让已有方法当 Lambda 用，是 Lambda 的语法糖 |
| **不可变集合** | `List.of()`/`Map.of()`，防御性拷贝时常用 |
| **三段式流** | 获取流（1次）+ 中间方法（多次）+ 终结方法（1次） |

**适用场景**：批量过滤、转换、统计、拼接字符串、收集到新集合——能写一句话绝不写 for 循环。