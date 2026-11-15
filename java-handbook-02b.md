# Java 语法学习手册（面向 Go 开发者）

## 第 2 部分（下）：真正的区别

> 本文件包含第 14–18 章：枚举与模式匹配、内部类与 lambda、包装类型、equals/hashCode/Comparable、泛型。
> 第 18 章和第 17 章会少量用到 `List`、`Set`，它们的完整讲解在第 3 部分，这里只当"能用的东西"看待。

**使用约定（同前）**：每个代码块第一行注释是文件名，保存为同名文件后执行 `java 文件名.java` 即可（JDK 21）。带 `main` 的类写在文件最前面，辅助类写在后面。

---

## 14. 枚举 enum、密封类 sealed、模式匹配

**和 Go 的差异**

- Go 没有枚举，用 `const` + `iota` 模拟，本质上只是整数。Java 的 `enum` 是**真正的类型**：类型安全，可以有字段、构造器、方法，还能实现接口。
- 每个枚举常量都是该类型的唯一实例，所以直接用 `==` 比较即可。
- 内置方法：`values()` 返回所有常量，`valueOf("X")` 按名字查找（找不到抛异常），`name()`，`ordinal()`（声明顺序的下标）。枚举天然可比较，按声明顺序。
- `switch` 表达式遍历枚举时，**覆盖全部常量就不需要 `default`；漏掉一个会编译报错**。Go 的 `switch` 不会检查。
- 每个枚举常量可以带自己的方法体（见下面的 `Op`）。
- **`sealed`（Java 17+）**：限定一个类或接口只能被指定的几个类型继承或实现。Go 里没有对应概念，它相当于其他语言里的"代数数据类型 / sum type"。
- 子类必须声明为 `final`、`sealed` 或 `non-sealed`，`record` 隐式是 `final`。
- **模式匹配（Java 21 正式）**：`switch` 可以直接按类型分支，≈ Go 的 `switch v := x.(type)`，但更强：
  - 可以用 `when` 加守卫条件。
  - 可以用**记录模式**直接解构 `record`：`case Rectangle(double w, double h)`。
  - 对 `sealed` 类型的 `switch` 做**穷尽性检查**，覆盖全部子类型就不用写 `default`，漏掉一个编译报错。Go 的 type switch 做不到。
  - 可以匹配 `null`：`case null`。

```java
// EnumDemo.java
import java.util.Arrays;

public class EnumDemo {
    public static void main(String[] args) {
        Color c = Color.GREEN;
        System.out.println(c + " " + c.name() + " " + c.ordinal());
        System.out.println(Arrays.toString(Color.values()));
        System.out.println(Color.valueOf("BLUE"));
        System.out.println(c == Color.GREEN);                 // 枚举直接用 == 比较
        try {
            Color.valueOf("PINK");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }

        // 带字段 / 方法的枚举
        for (Level lv : Level.values()) {
            System.out.println(lv + " weight=" + lv.weight() + " atLeastMid=" + lv.atLeast(Level.MID));
        }

        // switch 表达式 + 枚举: 覆盖全部常量就不需要 default, 漏掉一个会编译错误
        String hex = switch (c) {
            case RED -> "#f00";
            case GREEN -> "#0f0";
            case BLUE -> "#00f";
        };
        System.out.println(hex);

        // 每个常量可以有自己的方法实现
        for (Op op : Op.values()) {
            System.out.println("3 " + op.symbol() + " 4 = " + op.apply(3, 4));
        }
    }
}

enum Color { RED, GREEN, BLUE }

enum Level {
    LOW(1), MID(5), HIGH(10);                  // 常量后面的括号调用构造器
    private final int weight;

    Level(int weight) { this.weight = weight; }   // 枚举的构造器隐式 private
    int weight() { return weight; }
    boolean atLeast(Level other) { return compareTo(other) >= 0; }   // 按声明顺序比较
}

enum Op {
    PLUS("+") {
        int apply(int a, int b) { return a + b; }
    },
    TIMES("*") {
        int apply(int a, int b) { return a * b; }
    };

    private final String symbol;
    Op(String symbol) { this.symbol = symbol; }
    String symbol() { return symbol; }
    abstract int apply(int a, int b);          // 每个常量必须实现
}
```

**运行结果：**

```text
GREEN GREEN 1
[RED, GREEN, BLUE]
BLUE
true
No enum constant Color.PINK
LOW weight=1 atLeastMid=false
MID weight=5 atLeastMid=true
HIGH weight=10 atLeastMid=true
#0f0
3 + 4 = 7
3 * 4 = 12
```


下面是 `sealed` 和模式匹配：

```java
// SealedDemo.java
public class SealedDemo {
    public static void main(String[] args) {
        Shape[] shapes = { new Circle(1), new Square(2), new Rectangle(2, 3) };
        for (Shape s : shapes) {
            System.out.printf("%s -> %.2f%n", s, area(s));
        }

        // instanceof 里也可以直接解构 record
        if (shapes[2] instanceof Rectangle(double w, double h)) {
            System.out.println("w=" + w + ", h=" + h);
        }

        // 对任意类型做模式匹配 ≈ Go 的 switch v := x.(type)
        Object[] samples = { 42, 7, "hello", new int[3], 3.5, null };
        for (Object o : samples) {
            System.out.println(describe(o));
        }
    }

    // sealed + record + switch: 覆盖了所有子类型, 不需要 default (漏掉一个会编译错误)
    static double area(Shape s) {
        return switch (s) {
            case Circle c -> Math.PI * c.r() * c.r();
            case Square sq -> sq.side() * sq.side();
            case Rectangle(double w, double h) -> w * h;     // 记录模式: 直接取出字段
        };
    }

    static String describe(Object o) {
        return switch (o) {
            case null -> "null!";                            // 可以匹配 null
            case Integer i when i > 40 -> "big int " + i;    // when: 守卫条件
            case Integer i -> "int " + i;
            case String s -> "string of length " + s.length();
            case int[] arr -> "int array of " + arr.length;
            default -> "other " + o;
        };
    }
}

sealed interface Shape permits Circle, Square, Rectangle {}   // 只允许这三个类型实现
record Circle(double r) implements Shape {}
record Square(double side) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
```

**运行结果：**

```text
Circle[r=1.0] -> 3.14
Square[side=2.0] -> 4.00
Rectangle[w=2.0, h=3.0] -> 6.00
w=2.0, h=3.0
big int 42
int 7
string of length 5
int array of 3
other 3.5
null!
```


## 15. 内部类、匿名类、lambda 与方法引用

**和 Go 的差异**

- Go 的函数是一等公民，闭包天然可用。Java 的**方法不是值**，要靠 **lambda** 表达"函数值"。lambda 的本质是**函数式接口的实例**（第 13 章），所以调用时要写接口的方法名：`f.apply(x)`，不能写 `f(x)`。
- lambda 写法：`x -> x * 2`、`(a, b) -> a + b`、多行用 `(a, b) -> { ...; return ...; }`。
- **闭包只能捕获"有效 final"的局部变量**：捕获之后变量不能再被修改。Go 的闭包可以随意修改外部变量。需要可变计数器时，用长度为 1 的数组或 `AtomicInteger`（第 31 章）。
- 方法引用是 lambda 的简写，四种形式：`String::length`（类名::实例方法）、`System.out::println`（对象::实例方法）、`Integer::parseInt`（类名::静态方法）、`ArrayList::new`（构造器引用）。
- 内部类有四种，常用的是前两种：
  - **static 嵌套类**：只是把类放进另一个类的命名空间里，不依赖外部类对象。
  - **成员内部类**：每个实例**隐式持有外部类对象的引用**，可以访问外部类的私有成员。在 `static` 方法里不能直接 `new Inner()`，需要 `outer.new Inner()`。
  - 局部类：定义在方法里。
  - **匿名类**：当场实现一个接口或继承一个类，`new Interface() { ... }`。
- lambda 里的 `this` 指向外围对象；匿名类里的 `this` 指向匿名类自己。

**常用函数式接口 vs Go 的函数类型**

| Go | Java |
|---|---|
| `func()` | `Runnable` |
| `func() T` | `Supplier<T>` |
| `func(T)` | `Consumer<T>` |
| `func(T) R` | `Function<T, R>` |
| `func(T) bool` | `Predicate<T>` |
| `func(T, U) R` | `BiFunction<T, U, R>` |
| `func(T) T` | `UnaryOperator<T>` |
| `func(T, T) T` | `BinaryOperator<T>` |

标准库里还有避免装箱的基本类型版本，如 `IntFunction`、`IntPredicate`、`ToIntFunction`。

```java
// LambdaDemo.java
import java.util.ArrayList;
import java.util.function.*;

public class LambdaDemo {
    interface Op { int apply(int a, int b); }       // 自定义函数式接口 (嵌套在类里)

    static int applyTwice(Function<Integer, Integer> f, int x) {   // 函数作为参数
        return f.apply(f.apply(x));
    }

    static Function<Integer, Integer> makeAdder(int n) {           // 函数作为返回值
        return x -> x + n;
    }

    static Supplier<Integer> makeCounter() {        // ≈ Go 的计数器闭包
        int[] count = {0};                          // 局部变量不能被修改, 用数组绕开
        return () -> ++count[0];
    }

    public static void main(String[] args) {
        // lambda 是函数式接口的实例 ≈ Go 的 func 值
        Function<Integer, Integer> dbl = x -> x * 2;
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
        Supplier<String> sup = () -> "supplied";
        Consumer<String> print = s -> System.out.println("consumed " + s);
        Predicate<String> empty = String::isEmpty;           // 方法引用
        Runnable r = () -> System.out.println("run!");
        Op mul = (a, b) -> { return a * b; };                // 多行写法

        // 调用要通过接口的方法名: apply / get / test / accept / run, 不能写 dbl(5)
        System.out.println(dbl.apply(5) + " " + add.apply(1, 2) + " " + sup.get() + " "
                + empty.test("") + " " + mul.apply(3, 4));
        print.accept("x");
        r.run();

        System.out.println(applyTwice(dbl, 3));
        System.out.println(makeAdder(10).apply(5));
        Supplier<Integer> counter = makeCounter();
        counter.get();
        counter.get();
        System.out.println(counter.get());

        // 闭包只能捕获"有效 final"的局部变量
        int base = 100;
        Supplier<Integer> s1 = () -> base + 1;
        // base++;          // 取消注释会导致上一行的 lambda 编译错误
        System.out.println(s1.get());

        // 四种方法引用
        Function<String, Integer> len = String::length;           // 类名::实例方法
        Consumer<Object> out = System.out::println;               // 对象::实例方法
        Function<String, Integer> parse = Integer::parseInt;      // 类名::静态方法
        Supplier<ArrayList<String>> mk = ArrayList::new;          // 构造器引用
        out.accept(len.apply("hello") + " " + parse.apply("42") + " " + mk.get());

        // 嵌套类 / 内部类
        System.out.println(new Outer.Nested().hi());              // static 嵌套类: 不需要外部类对象
        Outer outer = new Outer("outer");
        Outer.Inner inner = outer.new Inner();                    // 成员内部类: 必须依附一个外部类对象
        System.out.println(inner.show());

        // 匿名类: 当场实现接口
        Op sub = new Op() {
            @Override
            public int apply(int a, int b) { return a - b; }
        };
        System.out.println(sub.apply(10, 3));

        // lambda 与匿名类里 this 的含义不同
        new ThisDemo().run();
    }
}

class Outer {
    private final String name;
    Outer(String name) { this.name = name; }

    static class Nested {
        String hi() { return "nested"; }
    }

    class Inner {                                  // 隐式持有 Outer.this
        String show() { return "inner sees " + name; }   // 可以访问外部类的私有字段
    }
}

class ThisDemo {
    String name = "ThisDemo";

    void run() {
        Runnable lam = () -> System.out.println("lambda this.name = " + this.name);
        Runnable anon = new Runnable() {
            String name = "anonymous";
            @Override
            public void run() { System.out.println("anonymous this.name = " + this.name); }
        };
        lam.run();
        anon.run();
    }
}
```

**运行结果：**

```text
10 3 supplied true 12
consumed x
run!
12
15
3
101
5 42 []
nested
inner sees outer
7
lambda this.name = ThisDemo
anonymous this.name = anonymous
```


## 16. 包装类型与自动装箱

**和 Go 的差异**

- Java 每个基本类型都有对应的**包装类**：`int`↔`Integer`，`long`↔`Long`，`double`↔`Double`，`boolean`↔`Boolean`，`char`↔`Character`。
- 为什么需要：泛型和集合里只能放对象，不能写 `List<int>`；包装类型还可以是 `null`，用来表示"没有值"（Go 里常用 `*int` 或 `sql.NullInt64`）。
- **自动装箱 / 拆箱**：`int` 和 `Integer` 之间自动转换，写起来几乎无感，但有两个坑。
- **坑 1：`==` 比较包装类型比的是引用**。`Integer` 缓存了 -128~127，范围内 `==` 碰巧为 true，超出范围就是 false。比较包装类型一律用 `equals`。
- **坑 2：拆箱 `null` 会抛 `NullPointerException`**。
- 包装类的 `equals` 会同时比较类型：`Long.valueOf(1).equals(1)` 是 false，因为右边是 `Integer`。
- 装箱会创建对象，在热点循环里（如 `Long sum = 0L; sum += i`）性能明显更差，用基本类型。
- 整数除以 0 抛 `ArithmeticException`；浮点数除以 0 得到 `Infinity` 或 `NaN`，不抛异常。
- 包装类里有大量实用静态方法：`parseInt`、`compare`、`toBinaryString`、`isDigit` 等。

```java
// BoxingDemo.java
import java.util.ArrayList;
import java.util.List;

public class BoxingDemo {
    public static void main(String[] args) {
        // 自动装箱 / 拆箱
        Integer boxed = 5;                         // int -> Integer
        int prim = boxed;                          // Integer -> int
        List<Integer> list = new ArrayList<>();    // 泛型只能用包装类型, 不能写 List<int>
        list.add(7);                               // 自动装箱
        int first = list.get(0);                   // 自动拆箱
        System.out.println(boxed + prim + first);

        // == 比较包装类型比的是引用! 缓存范围 -128 ~ 127
        Integer a = 127, b = 127;
        Integer c = 128, d = 128;
        System.out.println((a == b) + " " + (c == d));
        System.out.println(c.equals(d) + " " + (c.intValue() == d));   // 用 equals, 或者拆箱后再比

        // equals 会同时比较类型
        Long l = 1L;
        System.out.println(l.equals(1) + " " + l.equals(1L));

        // 拆箱 null -> NPE
        Integer missing = null;
        try {
            int x = missing;
        } catch (NullPointerException e) {
            System.out.println("unboxing null -> NPE");
        }

        // 常用工具方法
        System.out.println(Integer.MAX_VALUE + " " + Integer.toBinaryString(10) + " " + Integer.toHexString(255));
        System.out.println(Integer.compare(3, 5) + " " + Integer.valueOf("42") + " " + Character.isDigit('7'));
        System.out.println(Integer.parseInt("ff", 16));

        // 浮点数除以 0 不抛异常
        System.out.println(Double.compare(1.0, 2.0) + " " + Double.isNaN(0.0 / 0.0) + " " + 1.0 / 0);
    }
}
```

**运行结果：**

```text
17
true false
true true
false true
unboxing null -> NPE
2147483647 1010 ff
-1 42 true
255
-1 true Infinity
```


## 17. `equals` / `hashCode` / `toString`，`Comparable` 与 `Comparator`

**和 Go 的差异**

- Go 里 struct 的所有字段都可比较时，可以直接用 `==` 比较，也能直接当 map 的 key。Java 的类默认 `equals` 比**引用**，想要"内容相等"必须自己重写。
- **重写 `equals` 就必须同时重写 `hashCode`**。契约：两个对象 `equals` 为 true，`hashCode` 必须相同。否则放进 `HashSet` / `HashMap` 会出现"内容相同却当成两个"的问题。
- `toString()` ≈ Go 的 `String()`。
- 写法：用 `Objects.equals` / `Objects.hash` 辅助，或让 IDE 生成。**最简单的办法是直接用 `record`**，三个方法自动生成。
- 排序有两套机制：
  - `Comparable<T>`：类自己定义"自然顺序"，实现 `compareTo`。
  - `Comparator<T>`：外部传入的比较器，可以有多个，≈ Go 的 `sort.Slice` 里的比较函数。
- 比较方法返回的是**负数 / 0 / 正数**，不是 Go 里的 `bool`（`less`）。
- `Comparator` 可以链式组合：`Comparator.comparingInt(...).thenComparing(...).reversed()`。
- 不要用 `a - b` 做 `int` 比较，可能溢出，用 `Integer.compare(a, b)`。
- `List.sort` 是稳定排序。

```java
// EqualsDemo.java
import java.util.*;

public class EqualsDemo {
    public static void main(String[] args) {
        // 1. 没有重写 equals / hashCode: 比引用
        BadPoint b1 = new BadPoint(1, 2), b2 = new BadPoint(1, 2);
        System.out.println(b1.equals(b2));
        Set<BadPoint> badSet = new HashSet<>();
        badSet.add(b1);
        badSet.add(b2);
        System.out.println("badSet size = " + badSet.size());    // 内容相同却被当成两个

        // 2. 只重写 equals、没重写 hashCode: 违反契约, HashSet 通常仍然去不了重
        OnlyEquals o1 = new OnlyEquals(1, 2), o2 = new OnlyEquals(1, 2);
        Set<OnlyEquals> onlySet = new HashSet<>();
        onlySet.add(o1);
        onlySet.add(o2);
        System.out.println(o1.equals(o2) + " onlySet size = " + onlySet.size() + " (通常是 2)");

        // 3. 两个都重写
        GoodPoint g1 = new GoodPoint(1, 2), g2 = new GoodPoint(1, 2);
        Set<GoodPoint> goodSet = new HashSet<>();
        goodSet.add(g1);
        goodSet.add(g2);
        System.out.println(g1.equals(g2) + " " + (g1.hashCode() == g2.hashCode()) + " goodSet size = " + goodSet.size());
        System.out.println(g1);

        // 4. 最简单: record 自动生成
        System.out.println(new RecPoint(1, 2).equals(new RecPoint(1, 2)));

        // 排序: Comparable 自然顺序
        List<Person> people = new ArrayList<>(List.of(
                new Person("Carol", 25), new Person("Alice", 30), new Person("Bob", 25)));
        Collections.sort(people);                                  // 使用 compareTo (按 name)
        System.out.println(people);

        // Comparator 链式组合
        people.sort(Comparator.comparingInt(Person::age).thenComparing(Person::name));
        System.out.println(people);
        people.sort(Comparator.comparing(Person::name).reversed());
        System.out.println(people);

        // 手写比较器 ≈ Go 的 sort.Slice(people, func(i, j int) bool {...})
        people.sort((x, y) -> Integer.compare(y.age(), x.age()));  // 按年龄降序
        System.out.println(people);

        // 不要用减法比较 int: 可能溢出
        int big = Integer.MAX_VALUE, small = -1;
        System.out.println((big - small) + " vs " + Integer.compare(big, small));
    }
}

class BadPoint {                                // 没有重写任何方法
    final int x, y;
    BadPoint(int x, int y) { this.x = x; this.y = y; }
}

class OnlyEquals {                              // 只重写了 equals
    final int x, y;
    OnlyEquals(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        return o instanceof OnlyEquals p && x == p.x && y == p.y;
    }
}

class GoodPoint {                               // equals 和 hashCode 一起重写
    private final int x, y;
    GoodPoint(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof GoodPoint p)) return false;   // instanceof 模式匹配
        return x == p.x && y == p.y;
    }

    @Override
    public int hashCode() { return Objects.hash(x, y); }

    @Override
    public String toString() { return "GoodPoint(" + x + "," + y + ")"; }
}

record RecPoint(int x, int y) {}                // 一行搞定

record Person(String name, int age) implements Comparable<Person> {
    @Override
    public int compareTo(Person o) { return name.compareTo(o.name); }   // 自然顺序: 按名字
}
```

**运行结果：**

```text
false
badSet size = 2
true onlySet size = 2 (通常是 2)
true true goodSet size = 1
GoodPoint(1,2)
true
[Person[name=Alice, age=30], Person[name=Bob, age=25], Person[name=Carol, age=25]]
[Person[name=Bob, age=25], Person[name=Carol, age=25], Person[name=Alice, age=30]]
[Person[name=Carol, age=25], Person[name=Bob, age=25], Person[name=Alice, age=30]]
[Person[name=Alice, age=30], Person[name=Carol, age=25], Person[name=Bob, age=25]]
-2147483648 vs 1
```


## 18. 泛型

**和 Go 的差异**

- Go 1.18 才有泛型，Java 从 5 开始就有。语法对照：
  - Go：`func First[T any](s []T) T`
  - Java：`static <T> T first(List<T> list)`，类型参数 `<T>` 写在**返回类型前面**。
- 泛型类：`class Box<T>`，泛型 record：`record Pair<A, B>(A key, B value)`。创建时可以用菱形 `<>` 省略类型实参：`new ArrayList<>()`。
- 约束：Go 的 `[T constraints.Ordered]` ≈ Java 的 `<T extends Comparable<T>>`。接口约束也用 `extends`，多个约束用 `&`：`T extends A & B`。Java 的约束只能是"某个类型的子类型"，**不能约束运算符**，所以没法对 `T` 直接用 `+`。
- **类型擦除**：编译后泛型参数信息被擦掉，运行时 `List<String>` 和 `List<Integer>` 是同一个类。后果：
  - 不能 `new T()`、不能 `new T[n]`。
  - 不能 `obj instanceof List<String>`。
  - 不能用基本类型作类型实参：`List<int>` 不行，要 `List<Integer>`。
- **泛型不变性**：`List<Integer>` **不是** `List<Number>` 的子类型，不能互相赋值。
- 通配符（Go 没有）：
  - `List<? extends Number>`：可以读，不能写（生产者）。
  - `List<? super Integer>`：可以写，读出来只能当 `Object`（消费者）。
  - 口诀 PECS：Producer-Extends，Consumer-Super。
- 数组却是**协变**的：`Object[] objs = new String[1]` 能通过编译，往里放 `Integer` 到运行时才抛 `ArrayStoreException`。这是历史遗留的坑。
- 不要使用没有类型参数的**原始类型**（如 `List`），会丢掉类型检查。

```java
// GenericsDemo.java
import java.util.*;

public class GenericsDemo {
    // 泛型方法: <T> 写在返回类型前, ≈ Go 的 func First[T any](s []T) T
    static <T> T first(List<T> list) {
        return list.get(0);
    }

    // 类型约束: ≈ Go 的 [T constraints.Ordered]
    static <T extends Comparable<T>> T max(List<T> list) {
        T best = list.get(0);
        for (T t : list) {
            if (t.compareTo(best) > 0) best = t;
        }
        return best;
    }

    // ? extends: 只读 (生产者)
    static double sum(List<? extends Number> nums) {
        double total = 0;
        for (Number n : nums) total += n.doubleValue();
        return total;
    }

    // ? super: 只写 (消费者)
    static void fillInts(List<? super Integer> sink) {
        for (int i = 0; i < 3; i++) sink.add(i);
    }

    public static void main(String[] args) {
        System.out.println(first(List.of("a", "b")));
        System.out.println(max(List.of(3, 9, 4)) + " " + max(List.of("pear", "apple")));
        System.out.println(sum(List.of(1, 2, 3)) + " " + sum(List.of(1.5, 2.5)));
        List<Number> sink = new ArrayList<>();
        fillInts(sink);
        System.out.println(sink);

        // 泛型类 / 泛型 record
        Pair<String, Integer> p = new Pair<>("age", 30);          // <> 自动推断类型实参
        System.out.println(p.key() + "=" + p.value() + " " + p.swap());
        MyStack<String> st = new MyStack<>();
        st.push("x");
        st.push("y");
        System.out.println(st.pop() + st.pop() + " " + st.isEmpty());

        // 泛型不变: List<Integer> 不能赋给 List<Number>
        List<Integer> ints = List.of(1, 2);
        // List<Number> nums = ints;          // 编译错误
        List<? extends Number> nums = ints;   // 用通配符

        // 数组是协变的 (坑): 编译通过, 运行时才报错
        Object[] objs = new String[1];
        try {
            objs[0] = 1;
        } catch (ArrayStoreException e) {
            System.out.println("ArrayStoreException: " + e.getMessage());
        }

        // 类型擦除: 运行时看不到类型实参
        List<String> ls = new ArrayList<>();
        List<Integer> li = new ArrayList<>();
        System.out.println(ls.getClass() == li.getClass());
    }
}

record Pair<A, B>(A key, B value) {
    Pair<B, A> swap() {
        return new Pair<>(value, key);
    }
}

class MyStack<T> {
    private final ArrayList<T> items = new ArrayList<>();
    void push(T t) { items.add(t); }
    T pop() { return items.remove(items.size() - 1); }
    boolean isEmpty() { return items.isEmpty(); }
}
```

**运行结果：**

```text
a
9 pear
6.0 4.0
[0, 1, 2]
age=30 Pair[key=30, value=age]
yx true
ArrayStoreException: java.lang.Integer
true
```


---

## 第 2 部分（下）小结

1. `enum` 是真正的类型，可以带字段和方法；`sealed` + `record` + `switch` 模式匹配可以让编译器检查你有没有漏掉分支。
2. lambda 是函数式接口的实例，调用要写方法名；闭包只能捕获有效 final 的局部变量。
3. 包装类型比较一律用 `equals`；拆箱 `null` 会 NPE。
4. 重写 `equals` 必须同时重写 `hashCode`；懒得写就用 `record`。
5. 泛型有类型擦除，不能用基本类型，`List<Integer>` 不是 `List<Number>`；通配符遵循 PECS。

至此第 2 部分结束。下一个文件进入第 3 部分：`List`、`Map`、`Set`、`Deque`、不可变集合、迭代与并发修改异常、Stream API、`Optional`。
