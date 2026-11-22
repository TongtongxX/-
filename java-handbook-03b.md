# Java 语法学习手册（面向 Go 开发者）

## 第 3 部分（下）：Stream 与 Optional

> 本文件包含第 24、25 章：Stream API、Optional。
> 前置知识：lambda 与方法引用（第 15 章）、`List` / `Map`（第 19、20 章）。

**使用约定（同前）**：每个代码块第一行注释是文件名，保存为同名文件后执行 `java 文件名.java` 即可（JDK 21）。带 `main` 的类写在文件最前面，辅助类写在后面。

---

## 24. Stream API

**和 Go 的差异**

- Go 里处理集合靠 `for` 循环（加上 1.21 起的 `slices` / `maps` 包），没有链式的 map / filter / reduce。Java 的 Stream 是**声明式管道**：**数据源 → 中间操作 → 终止操作**。
- Stream **不是集合**，不存储数据，只是对数据源的一次处理流程。
- **中间操作是惰性的**：`filter`、`map`、`sorted` 等只是在描述流程，**没有终止操作就什么都不会执行**。
- 执行时是**每个元素依次走完整条管道**，不是一个阶段处理完所有元素再进下一阶段；遇到 `findFirst`、`limit`、`anyMatch` 这类短路操作会提前结束。
- **Stream 只能消费一次**，再次使用抛 `IllegalStateException`。
- Stream 不会修改数据源，每次得到的是新结果。
- lambda 里**不要修改外部变量**（本来也只能捕获有效 final 的变量），要累加请用 `reduce` / `collect`。
- lambda 里**不能直接抛 checked 异常**（第 26 章），需要在 lambda 内部 `try-catch`。

**创建流**

| 来源 | 写法 |
|---|---|
| 集合 | `list.stream()` |
| 数组 | `Arrays.stream(arr)` |
| 若干值 | `Stream.of("a", "b")` |
| 整数区间 | `IntStream.range(0, n)`（左闭右开）/ `IntStream.rangeClosed(1, n)` |
| 无限流 | `Stream.iterate(1, x -> x * 2)`、`Stream.generate(supplier)`，配合 `limit` |
| 字符串 | `"abc".chars()` |
| Map | `map.entrySet().stream()` |

**中间操作**：`filter`、`map`、`flatMap`（展开嵌套）、`distinct`、`sorted`、`limit`、`skip`、`peek`（调试用）、`mapToInt`、`boxed`。

**终止操作**：`forEach`、`collect`、`toList()`（Java 16+，返回**不可变**列表）、`count`、`reduce`、`min` / `max`、`anyMatch` / `allMatch` / `noneMatch`、`findFirst` / `findAny`、`toArray`。

- `min` / `max` / `findFirst` / `reduce`（无初始值版本）返回 `Optional`，见第 25 章。
- **基本类型流**：`IntStream` / `LongStream` / `DoubleStream` 避免装箱，才有 `sum()`、`average()`、`summaryStatistics()`。`Stream<Integer>` 上没有 `sum()`，要先 `mapToInt(Integer::intValue)`。
- `Collectors` 提供常用的收集器：`toList`、`toSet`、`joining`、`counting`、`groupingBy`、`partitioningBy`、`toMap`、`mapping`、`summingInt`、`averagingInt`。
- `Collectors.toMap` 遇到重复 key 会抛 `IllegalStateException`，需要传第三个参数（合并函数）。
- `groupingBy` 默认返回 `HashMap`，**遍历顺序不保证**。需要稳定顺序时传 `TreeMap::new` 或 `LinkedHashMap::new` 作为第二个参数。
- `stream.toList()` 返回的列表不能修改；`Collectors.toList()` 返回的是可变列表（实现上是 `ArrayList`，但规范不保证）。
- `parallelStream()` 可以并行处理，但对小数据量通常更慢，也容易引入线程安全问题，默认不要用。
- 什么时候用：转换、过滤、分组、聚合这类"数据流水线"很清晰；控制流复杂（多重 `break` / `continue`、需要改外部状态）时，还是普通 `for` 循环更好读。

```java
// StreamDemo.java
import java.util.*;
import java.util.stream.*;

public class StreamDemo {
    public static void main(String[] args) {
        basics();
        collectors();
    }

    static void basics() {
        List<Integer> nums = List.of(5, 3, 8, 1, 9, 2);

        // filter -> map -> toList
        // Go: for _, n := range nums { if n%2 == 0 { out = append(out, n*n) } }
        List<Integer> evenSquares = nums.stream()
                .filter(n -> n % 2 == 0)
                .map(n -> n * n)
                .toList();
        System.out.println(evenSquares);
        try {
            evenSquares.add(1);
        } catch (UnsupportedOperationException ex) {
            System.out.println("Stream.toList() returns an unmodifiable list");
        }

        // 惰性 + 短路: 每个元素依次走完整条管道, findFirst 找到第一个就停
        nums.stream()
                .filter(n -> { System.out.println("filter " + n); return n > 4; })
                .map(n -> { System.out.println("map " + n); return n * 2; })
                .findFirst();

        // 没有终止操作, 什么都不会执行 (下面这行不会输出任何内容)
        nums.stream().peek(n -> System.out.println("never printed"));

        // Stream 只能消费一次
        Stream<Integer> once = nums.stream();
        once.count();
        try {
            once.count();
        } catch (IllegalStateException ex) {
            System.out.println(ex.getMessage());
        }

        // 基本类型流: 避免装箱, 才有 sum / average
        int sum = nums.stream().mapToInt(Integer::intValue).sum();
        double avg = IntStream.rangeClosed(1, 5).average().orElse(0);
        IntSummaryStatistics stats = nums.stream().mapToInt(n -> n).summaryStatistics();
        System.out.println(sum + " " + avg + " " + stats.getMin() + " " + stats.getMax());
        System.out.println(IntStream.range(0, 5).boxed().toList());

        // max / reduce / sorted / limit
        int max = nums.stream().max(Integer::compare).orElse(-1);     // 返回 Optional, 见第 25 章
        int total = nums.stream().reduce(0, Integer::sum);
        System.out.println(max + " " + total + " " + nums.stream().sorted().limit(3).toList());

        // anyMatch / allMatch / noneMatch
        System.out.println(nums.stream().anyMatch(n -> n > 8) + " "
                + nums.stream().allMatch(n -> n > 0) + " "
                + nums.stream().noneMatch(n -> n > 9));

        // 创建流的几种方式
        System.out.println(Stream.of("a", "b", "c").collect(Collectors.joining(",", "[", "]")));
        System.out.println(Stream.iterate(1, x -> x * 2).limit(6).toList());
        System.out.println(Arrays.stream(new int[]{3, 1, 2}).sorted().boxed().toList());
        System.out.println("hello".chars().filter(c -> c == 'l').count());

        // flatMap 展开嵌套; distinct / sorted / skip
        List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3), List.of(4, 5));
        System.out.println(nested.stream().flatMap(List::stream).toList());
        System.out.println(Stream.of(3, 1, 3, 2, 1).distinct().sorted(Comparator.reverseOrder()).skip(1).toList());
    }

    record Employee(String name, String dept, int salary, int age) {}

    static void collectors() {
        List<Employee> staff = List.of(
                new Employee("Alice", "Eng", 120, 30),
                new Employee("Bob", "Eng", 100, 45),
                new Employee("Carol", "Ops", 80, 28),
                new Employee("Dave", "Ops", 90, 39),
                new Employee("Eve", "HR", 70, 51));

        // map + toList
        System.out.println(staff.stream().map(Employee::name).toList());

        // filter + sorted + map + joining
        String top = staff.stream()
                .filter(e -> e.salary() >= 80)
                .sorted(Comparator.comparingInt(Employee::salary).reversed())
                .map(Employee::name)
                .collect(Collectors.joining(", "));
        System.out.println(top);

        // groupingBy ≈ Go 里手写的 map[string][]Employee
        // 第二个参数 TreeMap::new 让结果按 key 排序 (默认是 HashMap, 顺序不保证)
        Map<String, List<String>> namesByDept = staff.stream().collect(
                Collectors.groupingBy(Employee::dept, TreeMap::new,
                        Collectors.mapping(Employee::name, Collectors.toList())));
        System.out.println(namesByDept);

        // groupingBy 配合不同的"下游收集器"
        Map<String, Long> countByDept = staff.stream().collect(
                Collectors.groupingBy(Employee::dept, TreeMap::new, Collectors.counting()));
        Map<String, Double> avgByDept = staff.stream().collect(
                Collectors.groupingBy(Employee::dept, TreeMap::new, Collectors.averagingInt(Employee::salary)));
        Map<String, Integer> sumByDept = staff.stream().collect(
                Collectors.groupingBy(Employee::dept, TreeMap::new, Collectors.summingInt(Employee::salary)));
        System.out.println(countByDept + " " + avgByDept + " " + sumByDept);

        // partitioningBy: 按条件分成 true / false 两组
        Map<Boolean, List<String>> parts = staff.stream().collect(
                Collectors.partitioningBy(e -> e.salary() >= 90,
                        Collectors.mapping(Employee::name, Collectors.toList())));
        System.out.println(parts);

        // toMap: 第三个参数是 key 重复时的合并函数, 第四个参数指定 Map 实现
        Map<String, Integer> salaryByName = staff.stream().collect(
                Collectors.toMap(Employee::name, Employee::salary, (a, b) -> a, LinkedHashMap::new));
        System.out.println(salaryByName);
        try {
            staff.stream().collect(Collectors.toMap(Employee::dept, Employee::name));   // dept 有重复
        } catch (IllegalStateException ex) {
            System.out.println(ex.getMessage());
        }

        // 统计信息 / 取最大
        System.out.println(staff.stream().mapToInt(Employee::salary).summaryStatistics());
        String richest = staff.stream()
                .max(Comparator.comparingInt(Employee::salary))
                .map(Employee::name)
                .orElse("none");
        System.out.println(richest);
    }
}
```

**运行结果：**

```text
[64, 4]
Stream.toList() returns an unmodifiable list
filter 5
map 5
stream has already been operated upon or closed
28 3.0 1 9
[0, 1, 2, 3, 4]
9 28 [1, 2, 3]
true true true
[a,b,c]
[1, 2, 4, 8, 16, 32]
[1, 2, 3]
2
[1, 2, 3, 4, 5]
[2, 1]
[Alice, Bob, Carol, Dave, Eve]
Alice, Bob, Dave, Carol
{Eng=[Alice, Bob], HR=[Eve], Ops=[Carol, Dave]}
{Eng=2, HR=1, Ops=2} {Eng=110.0, HR=70.0, Ops=85.0} {Eng=220, HR=70, Ops=170}
{false=[Carol, Eve], true=[Alice, Bob, Dave]}
{Alice=120, Bob=100, Carol=80, Dave=90, Eve=70}
Duplicate key Eng (attempted merging values Alice and Bob)
IntSummaryStatistics{count=5, sum=460, min=70, average=92.000000, max=120}
Alice
```


## 25. Optional

**和 Go 的差异**

- Go 用 `(T, bool)`、`(T, error)` 或 `nil` 指针表示"可能没有值"。Java 用 `Optional<T>`：一个**可能有值也可能为空的容器**，主要用作**方法的返回类型**，强制调用方意识到"可能没有"。
- 区分场景：
  - "找不到"是正常情况 → 返回 `Optional`（≈ Go 的 `(T, bool)`）。
  - "不应该发生 / 出错了" → 抛异常（≈ Go 的 `(T, error)`，第 26 章）。
- 创建：`Optional.of(x)`（x 为 `null` 会 NPE）、`Optional.ofNullable(x)`（把可能为 `null` 的值包起来）、`Optional.empty()`。
- 取值：
  - `get()`：为空时抛 `NoSuchElementException`，**不要直接用**。
  - `orElse(默认值)`：参数**总会被计算**，哪怕 Optional 有值。
  - `orElseGet(supplier)`：只有为空时才执行，默认值计算有开销时用它。
  - `orElseThrow(() -> new XxxException(...))`：为空就抛自己的异常。
  - `ifPresent(consumer)` / `ifPresentOrElse(consumer, runnable)`：有值才执行。
- 转换：`map`、`filter`、`flatMap`。**转换函数本身返回 `Optional` 时必须用 `flatMap`**，否则会得到 `Optional<Optional<T>>`。链式调用可以取代层层 `if (x != null)`。
- 基本类型有专门版本：`OptionalInt`、`OptionalDouble`、`OptionalLong`（如 `IntStream.average()` 返回 `OptionalDouble`）。
- `Optional` 常见来源：Stream 的 `findFirst` / `max` / `min` / `reduce`。
- 使用规范：
  - 返回 `Optional` 的方法**绝对不能返回 `null`**。
  - 不要把 `Optional` 用作字段、方法参数或集合元素。
  - 不要写 `if (opt.isPresent()) { opt.get() ... }`，直接用 `map` / `orElse` / `ifPresent`。

```java
// OptionalDemo.java
import java.util.*;
import java.util.stream.*;

public class OptionalDemo {
    // "可能找不到" -> 返回 Optional ≈ Go 的 (T, bool)
    static Optional<User> findUser(int id) {
        return id == 1 ? Optional.of(new User("alice", "alice@example.com")) : Optional.empty();
    }

    static Optional<String> managerOf(User u) {
        return u.name().equals("alice") ? Optional.of("bob") : Optional.empty();
    }

    static String expensiveDefault() {
        System.out.println("  (computing default)");
        return "default";
    }

    public static void main(String[] args) {
        Optional<User> found = findUser(1);
        Optional<User> missing = findUser(2);
        System.out.println(found);
        System.out.println(missing);
        System.out.println(found.isPresent() + " " + missing.isEmpty());

        // 不要直接 get(): 为空时抛异常
        try {
            missing.get();
        } catch (NoSuchElementException ex) {
            System.out.println("get() on empty -> " + ex.getMessage());
        }

        // 提供默认值
        System.out.println(missing.map(User::name).orElse("anonymous"));

        // orElse 的参数总会被计算, orElseGet 是惰性的
        System.out.println("orElse:");
        found.map(User::name).orElse(expensiveDefault());
        System.out.println("orElseGet:");
        found.map(User::name).orElseGet(OptionalDemo::expensiveDefault);

        // 为空时抛自己的异常
        try {
            missing.orElseThrow(() -> new IllegalStateException("user not found"));
        } catch (IllegalStateException ex) {
            System.out.println(ex.getMessage());
        }

        // 有值才执行
        found.ifPresent(u -> System.out.println("found " + u.name()));
        missing.ifPresentOrElse(u -> System.out.println("found"), () -> System.out.println("not found"));

        // 链式转换, 代替层层 null 检查
        String domain = findUser(1)
                .map(User::email)
                .filter(mail -> mail.contains("@"))
                .map(mail -> mail.substring(mail.indexOf('@') + 1))
                .orElse("unknown");
        System.out.println(domain);

        // 转换函数本身返回 Optional 时用 flatMap
        System.out.println(findUser(1).flatMap(OptionalDemo::managerOf));
        System.out.println(findUser(2).flatMap(OptionalDemo::managerOf));

        // 把可能为 null 的值包成 Optional
        String raw = System.getenv("SURELY_NOT_SET_12345");
        System.out.println(Optional.ofNullable(raw).orElse("env not set"));
        try {
            Optional.of(null);                        // of() 不接受 null
        } catch (NullPointerException ex) {
            System.out.println("Optional.of(null) -> NPE");
        }

        // 实用例子: 读配置, 缺失就用默认值
        Map<String, String> cfg = Map.of("port", "8080");
        int port = Optional.ofNullable(cfg.get("port")).map(Integer::parseInt).orElse(80);
        String host = Optional.ofNullable(cfg.get("host")).orElse("localhost");
        System.out.println(host + ":" + port);

        // Stream 的终止操作经常返回 Optional
        Optional<String> firstLong = Stream.of("go", "java", "kotlin")
                .filter(w -> w.length() > 4)
                .findFirst();
        System.out.println(firstLong.orElse("none"));

        // 基本类型有专门的版本
        System.out.println(IntStream.empty().average() + " " + IntStream.of(1, 2).average());
    }
}

record User(String name, String email) {}
```

**运行结果：**

```text
Optional[User[name=alice, email=alice@example.com]]
Optional.empty
true true
get() on empty -> No value present
anonymous
orElse:
  (computing default)
orElseGet:
user not found
found alice
not found
example.com
Optional[bob]
Optional.empty
env not set
Optional.of(null) -> NPE
localhost:8080
kotlin
OptionalDouble.empty OptionalDouble[1.5]
```


---

## 第 3 部分（下）小结

1. Stream 是"数据源 → 中间操作 → 终止操作"的管道；中间操作惰性，没有终止操作不执行；Stream 只能用一次。
2. 求和、平均要用基本类型流（`mapToInt`）；分组用 `groupingBy`，要稳定输出顺序就传 `TreeMap::new`。
3. `toList()` 不可变；`toMap` 遇到重复 key 会抛异常，要传合并函数。
4. `Optional` 用作**返回值**，表示"可能没有"；别直接 `get()`，用 `map` / `orElse` / `ifPresent`；返回 `Optional` 的方法绝不返回 `null`。
5. 逻辑复杂、需要改外部状态时，普通 `for` 循环往往比 Stream 更清楚。

至此第 3 部分结束。下一个文件进入第 4 部分：异常体系（checked 与 unchecked，对比 Go 的 `error` / `panic`）、`try / catch / finally`、自定义异常、`try-with-resources`（对比 `defer`）。
