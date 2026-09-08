# 内部类到 Lambda：Java 的精简之路

> 这篇博客对应我学过的《TyporaFile/从内部类到Lanmbda.md》。

## 一句话定位

**匿名内部类是"小而全的类"，lambda 是"一段行为"。** 同样是实现接口的某个方法，匿名内部类需要 6 行代码，lambda 只要 1 行；背后不只是语法简化，而是底层完全不同的实现机制。

---

## 一、四大内部类对比

| 类型 | 位置 | 关键特点 |
| ---- | ---- | ---- |
| **成员内部类** | 类内、方法外 | 寄生在外部类里，能访问外部类**所有成员**（包括 private） |
| **静态嵌套类** | 类内、方法外 | **不持有**外部类引用，只能访问外部类静态成员 |
| **局部内部类** | 方法内 | 作用域只在当前方法 |
| **匿名内部类** | 方法内 | 即定义即实例化的局部内部类，**Lambda 的前身** |

### 1.1 成员内部类：寄生

```java
public class Outer {
    private String name = "Outer";

    class Inner {
        void print() {
            System.out.println(Outer.this.name);  // 隐式持有外部类引用
        }
    }

    void create() {
        Inner i = new Inner();    // 外部类内部，直接 new
    }
}

// 外部创建：必须先有外部实例
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

**编译后的秘密**：编译器偷偷给 `Inner` 的构造器塞一个 `Outer` 参数：`Inner(Outer this$0)`。这就是为什么内部类总能访问外部类成员——它手里一直攥着外部类的引用。

**潜在问题**：如果外部类已经不需要了，但内部类还被引用着，外部类就没办法被 GC，造成**内存泄漏**。

### 1.2 静态嵌套类：只是命名空间

```java
class Outer {
    static String staticName = "static";
    String instanceName = "instance";

    static class Nested {
        void test() {
            System.out.println(staticName);    // ✅ OK
            // System.out.println(instanceName); // ❌ 编译错误
        }
    }
}

// 创建：不需要外部实例
Outer.Nested n = new Outer.Nested();
```

**易错点**：`Outer.Nested` 里的 `Outer` 只是**寻址作用**（告诉编译器类在哪），**就像包名一样**；而成员内部类的 `outer.new Inner()` 语法里的 `outer` 是真实对象。

**实践建议**：如果嵌套类不需要访问外部实例成员，**一律声明为 `static`**，避免内存泄漏。`HashMap.Node`、`LinkedList.Node` 都是静态嵌套类的经典例子。

### 1.3 局部内部类：方法里的类

作用域限定在当前方法内，基本被匿名内部类替代，不展开。

### 1.4 匿名内部类：Lambda 的前身

**本质上是"即定义即实例化"的局部内部类**，主要用途是实现**只有一个方法的接口**（函数式接口）。

```java
// 6 行的匿名内部类
Runnable r1 = new Runnable() {
    @Override public void run() {
        System.out.println("hello");
    }
};

// 1 行的 Lambda
Runnable r2 = () -> System.out.println("hello");
```

---

## 二、Lambda 表达式

### 2.1 使用前提：函数式接口三条件

| 条件 | 说明 |
| ---- | ---- |
| 必须是**接口** | 抽象类、普通类都没戏 |
| **只能有一个抽象方法** | 两个就报"不是函数式接口" |
| default / static / Object 方法 | 随便有多少，都不算数 |

### 2.2 this 指向（核心区别）

```java
public class Outer {
    void test() {
        // 匿名内部类：this 指向匿名类自身
        Runnable r1 = new Runnable() {
            @Override public void run() {
                System.out.println(this);   // Outer$1@xxx
            }
        };

        // Lambda：this 直接指向外部类！
        Runnable r2 = () -> {
            System.out.println(this);       // Outer@xxx
        };
    }
}
```

### 2.3 底层机制（最硬核的区分）

> Lambda **不是**内部类的语法糖。

| | 匿名内部类 | Lambda |
| ---- | ---- | ---- |
| 本质 | 真的 new 了一个子类对象（有构造、有状态、有 `.class` 文件） | 只生成一段行为（无构造、无状态、无 `.class` 文件） |
| 字节码 | 编译时生成 `Outer$1.class` | 用 `invokedynamic` 指令，运行时动态生成 |

---

## 四、匿名内部类 vs Lambda（最硬的分界线）

### Lambda 对抽象类无能为力

```java
abstract class Task {
    abstract void run();
}

// ✅ 匿名内部类：可以继承抽象类
Task t1 = new Task() {
    @Override void run() {
        System.out.println("跑起来了");
    }
};

// ❌ Lambda：编译报错
Task t2 = () -> System.out.println("跑不了");
// 报错：Target type of a lambda conversion must be an interface
```

**哪怕抽象类只有一个抽象方法、长得和接口一模一样，lambda 也不认——它只认 `interface`。**

### 完整对比表

| 能当目标吗 | 匿名内部类 | Lambda |
| ---- | ---- | ---- |
| 函数式接口（1 个抽象方法） | ✅ | ✅ |
| 普通接口（多个抽象方法） | ✅ 全都能实现 | ❌ |
| **抽象类** | ✅ 能继承 | ❌ 永远不行 |
| **普通类** | ✅ 能继承并改写方法 | ❌ |
| 一次重写**多个**方法 | ✅ | ❌ |
| 自带**字段/状态** | ✅ | ❌ |

### 体现差距的例子：要重写两个方法

```java
abstract class Task {
    abstract void run();
    abstract void stop();
    int count = 0;  // 抽象类还能带状态
}

Task t = new Task() {
    private int retry = 3;   // ✅ 匿名类可以有状态
    @Override void run()  { count++; }
    @Override void stop() { }
};
```

这种活 lambda 完全接不了：它**只能填一个方法的空，也存不了任何状态**。

---

## 五、一句话总结

| | 匿名内部类 | Lambda |
| ---- | ---- | ---- |
| 本质 | 真的 new 了一个子类对象 | 只生成一段行为 |
| 适用范围 | 接口 + 抽象类 + 普通类 | 只有函数式接口 |
| 多方法 / 状态 | 都行 | 都不行 |

**记法**：要继承抽象类、要状态、要重写多个方法——只能匿名内部类；只是给接口的一个方法填行为——lambda 一行搞定。