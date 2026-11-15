# Java 语法学习手册（面向 Go 开发者）

## 第 2 部分（上）：真正的区别

> 本文件包含第 9–13 章：类与对象、引用语义与 null、封装与 record、继承、接口。
> 第 14–18 章（枚举与模式匹配、lambda、包装类型、equals/hashCode、泛型）在下一个文件。

**使用约定（同第 0 部分）**：每个代码块第一行注释是文件名，保存为同名文件后执行 `java 文件名.java` 即可（JDK 21）。带 `main` 的类写在文件最前面，辅助类写在后面。

---

## 9. 类与对象

**和 Go 的差异**

- Go 里 struct 和方法是分开定义的；Java 的 `class` 把字段、构造器、方法写在一起。
- Go 没有构造器，惯例是 `NewXxx()` 函数或直接 `Xxx{...}` 字面量。Java 有**构造器**：与类同名、没有返回类型，用 `new` 调用。
- Java 没有 struct 字面量，不能写 `Person{Name: "x"}`。
- `this` ≈ Go 的方法接收者（`p *Person`），指向当前对象。字段和参数同名时需要写 `this.` 区分，其余情况可以省略。
- 没写任何构造器时，编译器会提供一个无参构造器；**一旦写了任何构造器，这个默认的就消失了**。
- `this(...)` 可以调用本类的另一个构造器，必须写在第一行。
- 字段有默认值（数字 0、`boolean` false、引用类型 `null`），和 Go 的零值类似。
- `static` 成员属于类而不是对象，≈ Go 的包级变量和函数。静态方法里不能使用 `this`，也不能直接访问实例字段。
- `toString()` ≈ Go 的 `String()`（`fmt.Stringer`），`println` 打印对象时会自动调用。
- `private` 是**类级别**的：同一个类的方法可以访问另一个对象的私有字段。Go 的可见性是包级别的。

**访问修饰符**

| 修饰符 | 同一个类 | 同一个包 | 子类 | 其他包 |
|---|---|---|---|---|
| `private` | ✓ | | | |
| （不写，包私有） | ✓ | ✓ | | |
| `protected` | ✓ | ✓ | ✓ | |
| `public` | ✓ | ✓ | ✓ | ✓ |

```java
// ClassDemo.java
public class ClassDemo {
    public static void main(String[] args) {
        Person p1 = new Person("Alice", 30);   // new + 构造器 (Go: NewPerson(...) 或 &Person{...})
        Person p2 = new Person("Bob");         // 调用另一个构造器
        Person p3 = new Person();              // 无参构造器
        p1.birthday();
        System.out.println(p1.describe());
        System.out.println(p2.describe());
        System.out.println(p3.describe());
        System.out.println(p1);                // println 自动调用 toString()
        System.out.println(p1.sameAge(p2));
        // System.out.println(p1.name);        // 编译错误: name 在 Person 中是 private

        // 静态成员属于类, 通过类名访问
        System.out.println(Person.getCount() + " " + Person.SPECIES);

        // 没写任何构造器的类: 编译器提供无参构造器, 字段是默认值
        Box box = new Box();
        System.out.println(box.w + " " + box.label + " " + box.open);
    }
}

class Person {
    static final String SPECIES = "human";   // 类常量
    private static int count = 0;            // 静态字段: 所有对象共享一份

    private String name;
    private int age;

    Person() { this("unknown", 0); }         // this(...) 调用本类另一个构造器, 必须写在第一行
    Person(String name) { this(name, 18); }
    Person(String name, int age) {
        this.name = name;                    // this 区分字段和同名参数, ≈ Go 的接收者 p
        this.age = age;
        count++;
    }

    void birthday() { age++; }
    String describe() { return name + " (" + age + ")"; }

    // private 是"类级别"的: 可以访问同类另一个对象的私有字段
    boolean sameAge(Person other) { return this.age == other.age; }

    static int getCount() { return count; }  // 静态方法里不能用 this

    @Override
    public String toString() {               // ≈ Go 的 String() 方法
        return "Person{name=" + name + ", age=" + age + "}";
    }
}

class Box {
    int w;
    String label;
    boolean open;
}
```

**运行结果：**

```text
Alice (31)
Bob (18)
unknown (0)
Person{name=Alice, age=31}
false
3 human
0 null false
```


## 10. 引用语义与 null

**和 Go 的差异**

- Go 里 struct 变量是**值**，赋值会整体复制，想共享要用指针 `*T`。Java 里对象变量本身就是**引用**，赋值只复制引用，两个变量指向同一个对象。
- Java 没有 `&`、`*`，没有指针运算。可以把每个对象变量理解成 Go 里的 `*T`。
- 除了 8 个基本类型，Java 没有"值类型"。想要独立副本，必须自己复制（拷贝构造器），或者使用不可变对象 / `record`（下一章）。
- `==` 比较两个引用是否指向同一个对象，不是比较内容。没有重写 `equals` 的类，`equals` 默认也是比引用（第 17 章）。
- `null` ≈ Go 的 `nil`，对 `null` 调用方法或访问字段会抛 `NullPointerException`（NPE）。
- 区别在于：Java 里**任何引用类型都可能是 null**，包括 `String` 和数组；Go 的 `string`、struct 永远不会是 `nil`。
- `final` 修饰引用变量，只表示这个变量不能再指向别的对象，对象内部的内容仍然可以改。
- 有垃圾回收，没有手动释放。
- 常用的 null 防御：把常量放前面 `"abc".equals(s)`，两边都可能为 null 时用 `Objects.equals`，入口处用 `Objects.requireNonNull`，或者用 `Optional`（第 25 章）。

```java
// RefDemo.java
import java.util.Objects;

public class RefDemo {
    static String title;   // 引用类型字段默认 null

    public static void main(String[] args) {
        // 赋值只复制引用 (Go 里 struct 赋值是复制整个值)
        Point a = new Point(1, 2);
        Point b = a;
        b.x = 100;
        System.out.println(a.x + " " + (a == b));

        // 想要独立副本, 必须自己复制
        Point c = new Point(a.x, a.y);
        c.x = -1;
        System.out.println(a.x + " " + c.x + " " + (a == c));

        // == 比引用; 没有重写 equals 的类, equals 默认也比引用
        Point d = new Point(1, 2);
        Point e = new Point(1, 2);
        System.out.println((d == e) + " " + d.equals(e));

        // final 引用: 引用不能改, 对象内容可以改
        final Point f = new Point(0, 0);
        f.x = 5;
        // f = new Point(1, 1);     // 编译错误
        System.out.println(f.x);

        // null 与 NPE
        try {
            System.out.println(title.length());
        } catch (NullPointerException ex) {
            System.out.println("NPE: " + ex.getMessage());
        }

        // null 防御
        String s = null;
        System.out.println("abc".equals(s));          // false: 常量放前面不会 NPE
        System.out.println(Objects.equals(s, null));  // true: 两边都可能为 null 时使用
        System.out.println("value: " + s);            // 字符串拼接遇到 null 会变成 "null"
        try {
            Objects.requireNonNull(s, "s must not be null");
        } catch (NullPointerException ex) {
            System.out.println(ex.getMessage());
        }
    }
}

class Point {
    int x, y;    // 为了简洁这里用了公开字段, 下一章讲封装
    Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

**运行结果：**

```text
100 true
100 -1 false
false false
5
NPE: Cannot invoke "String.length()" because "RefDemo.title" is null
false
true
value: null
s must not be null
```


## 11. 封装与 getter/setter，以及 record

**和 Go 的差异**

- Go 靠首字母大小写控制导出，字段经常直接导出。Java 的惯例是**字段 `private`，通过方法访问**。
- 命名惯例：`getName()` / `setName(...)`，布尔值用 `isActive()`。Go 的惯例是 `Name()` 不带 `Get`。
- 为什么这么麻烦：可以在 setter 里校验、可以改内部实现而不影响调用方，大量框架也依赖这个命名约定。
- 不可变对象：字段 `final`、不提供 setter，构造器里一次性赋值。要"修改"就返回新对象。
- **`record`（Java 16+）**是 Java 版的轻量 struct：一行声明就自动生成构造器、访问器、`equals`、`hashCode`、`toString`。
- record 的访问器是 `x()`，不是 `getX()`。
- record 的字段隐式 `private final`，不可变；`equals` 按字段值比较，类似 Go struct 的 `==`。
- record 可以加方法、静态成员，可以实现接口，不能继承别的类。
- record 的不可变是**浅层**的：如果字段是一个 `List`，List 本身仍然可以被修改。
- 紧凑构造器（没有参数列表的 `Record { ... }`）用来做校验。

```java
// Encapsulation.java
public class Encapsulation {
    public static void main(String[] args) {
        Account acc = new Account("A-001", 100);
        acc.deposit(50);
        System.out.println(acc.getId() + " " + acc.getBalance());
        try {
            acc.withdraw(1000);
        } catch (IllegalArgumentException e) {
            System.out.println("error: " + e.getMessage());
        }
        acc.setNickname("main");
        System.out.println(acc.getNickname() + " " + acc.isActive());

        // record: 自动生成构造器 / 访问器 / equals / hashCode / toString
        Point p1 = new Point(1, 2);
        Point p2 = new Point(1, 2);
        System.out.println(p1);
        System.out.println(p1.x() + " " + p1.y());     // 访问器是 x(), 不是 getX()
        System.out.println(p1.equals(p2) + " " + (p1 == p2) + " " + (p1.hashCode() == p2.hashCode()));
        System.out.println(p1.plus(new Point(10, 10))); // 不可变: 返回新对象
        // p1.x = 5;                                    // 编译错误: record 字段不可修改
        System.out.println(Point.ORIGIN);

        // 紧凑构造器做校验
        System.out.println(new Range(1, 5).length());
        try {
            new Range(5, 1);
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }
}

class Account {
    private final String id;          // final: 构造后不能再改
    private long balance;
    private String nickname = "";
    private boolean active = true;

    Account(String id, long balance) {
        this.id = id;
        this.balance = balance;
    }

    String getId() { return id; }
    long getBalance() { return balance; }
    String getNickname() { return nickname; }
    void setNickname(String nickname) { this.nickname = nickname; }
    boolean isActive() { return active; }       // boolean 的 getter 惯例是 isXxx

    void deposit(long amount) {
        if (amount <= 0) throw new IllegalArgumentException("amount must be positive");
        balance += amount;
    }

    void withdraw(long amount) {
        if (amount > balance) throw new IllegalArgumentException("insufficient balance: " + balance);
        balance -= amount;
    }
}

record Point(int x, int y) {
    static final Point ORIGIN = new Point(0, 0);   // record 里可以有静态成员
    Point plus(Point o) { return new Point(x + o.x, y + o.y); }
}

record Range(int lo, int hi) {
    Range {   // 紧凑构造器: 没有参数列表, 在字段赋值之前执行
        if (lo > hi) throw new IllegalArgumentException("lo > hi: " + lo + " > " + hi);
    }
    int length() { return hi - lo; }
}
```

**运行结果：**

```text
A-001 150
error: insufficient balance: 150
main true
Point[x=1, y=2]
1 2
true false true
Point[x=11, y=12]
Point[x=0, y=0]
4
lo > hi: 5 > 1
```


## 12. 继承

**和 Go 的差异**

- Go 没有继承，用**组合 + 嵌入 + 接口**。Java 用 `extends` 实现**单继承**：子类拥有父类的字段和方法。
- 一个类只能 `extends` 一个类，但可以 `implements` 多个接口（下一章）。所有类隐式继承 `Object`。
- `super(...)` 调用父类构造器，必须写在子类构造器第一行；`super.method()` 调用父类的实现。
- `@Override` 让编译器检查"我确实重写了父类方法"，拼错方法名时会报错。**一律加上**。
- **多态是最大的区别**：父类类型的变量可以指向子类对象，调用方法时按**运行时的实际类型**分发。Go 的嵌入没有这种能力：被嵌入的结构体方法里调用 `e.Foo()`，永远是它自己的 `Foo`，不会分发到外层结构体的"重写"。
- 因此 Java 里可以写"模板方法"：父类定义流程，调用抽象方法，由子类填充细节。
- `abstract` 类不能直接实例化，可以包含抽象方法（没有方法体，子类必须实现）。
- `final` 修饰的方法不能被重写，`final` 修饰的类不能被继承。
- 只有**方法**有多态，字段没有。不要在子类里声明和父类同名的字段。
- `instanceof` 判断类型；Java 16+ 支持模式匹配 `obj instanceof Dog d`，判断成功就直接得到转好类型的变量。≈ Go 的类型断言 `v, ok := x.(Dog)`。
- Java 里同样推荐"组合优于继承"，但没有嵌入的方法提升，要手动转发。

```java
// Inheritance.java
public class Inheritance {
    public static void main(String[] args) {
        // 构造器链: 先父类, 后子类
        Animal[] animals = { new Dog("Rex"), new Cat("Tom") };

        // 多态: 变量类型是 Animal, 调用哪个 speak() 由运行时的实际对象决定
        for (Animal a : animals) {
            System.out.println(a.getName() + " (" + a.kind() + "): " + a.speak());
        }

        // super.speak() 调用父类实现
        System.out.println(new Puppy("Bit").speak());

        // instanceof + 模式匹配 (Java 16+)
        Animal first = animals[0];
        if (first instanceof Dog d) {     // 成功时 d 已经是 Dog 类型
            d.fetch();
        }
        // Animal x = new Animal("x");    // 编译错误: abstract 类不能实例化

        // 模板方法: 父类方法里调用的抽象方法, 会分发到子类的实现 (Go 的嵌入做不到)
        Report r = new SalesReport(42);
        System.out.println(r.render());

        // 组合: Java 没有嵌入的方法提升, 需要手动转发
        new Car().start();
    }
}

abstract class Animal {
    private final String name;

    Animal(String name) {
        this.name = name;
        System.out.println("Animal constructor: " + name);
    }

    String getName() { return name; }
    abstract String speak();             // 抽象方法: 子类必须实现
    String kind() { return "animal"; }   // 普通方法: 可以被重写
}

class Dog extends Animal {
    Dog(String name) { super(name); }    // 必须先调用父类构造器
    @Override String speak() { return "Woof"; }
    void fetch() { System.out.println(getName() + " fetches"); }
}

class Cat extends Animal {
    Cat(String name) { super(name); }
    @Override String speak() { return "Meow"; }
    @Override String kind() { return "feline"; }
}

class Puppy extends Dog {
    Puppy(String name) {
        super(name);
        System.out.println("Puppy constructor");
    }
    @Override String speak() { return super.speak() + " (squeaky)"; }   // 调用父类实现
}

abstract class Report {
    final String render() { return header() + " | " + body(); }   // final: 子类不能重写
    String header() { return "Report"; }
    abstract String body();
}

class SalesReport extends Report {
    private final int total;
    SalesReport(int total) { this.total = total; }
    @Override String header() { return "Sales Report"; }
    @Override String body() { return "total=" + total; }
}

class Engine {
    void start() { System.out.println("engine started"); }
}

class Car {
    private final Engine engine = new Engine();   // 组合: 持有, 而不是继承
    void start() {
        engine.start();                           // 手动转发 (Go 嵌入会自动提升 Start)
        System.out.println("car started");
    }
}
```

**运行结果：**

```text
Animal constructor: Rex
Animal constructor: Tom
Rex (animal): Woof
Tom (feline): Meow
Animal constructor: Bit
Puppy constructor
Woof (squeaky)
Rex fetches
Sales Report | total=42
engine started
car started
```


## 13. 接口

**和 Go 的差异**

- **Go 是隐式实现，Java 是显式实现**：Go 里只要方法集合满足就自动实现接口；Java 必须写 `implements`。后果是你无法让第三方已有的类型事后"满足"某个接口，只能写一个适配器类。
- 接口里的方法默认是 `public abstract`。**实现类里必须写 `public`**，否则编译报错（不能降低访问权限）。
- Java 8+ 接口可以有 `default` 方法（带实现）和 `static` 方法。Java 9+ 还可以有 `private` 方法。
- 一个类可以 `implements` 多个接口，接口也可以 `extends` 多个接口。
- 两个接口有同名 `default` 方法时，实现类必须重写，并用 `接口名.super.方法()` 明确选择。
- 接口里不能有实例字段，只能有常量（`public static final`）。
- **函数式接口**：只有一个抽象方法的接口，可以用 lambda 实现（第 15 章）。标准库提供了现成的：`Runnable`、`Comparator`、`Function`、`Predicate`、`Supplier`、`Consumer` 等。
- Go 的 `any`（空接口）≈ Java 的 `Object`。

**抽象类 vs 接口**

| | 抽象类 | 接口 |
|---|---|---|
| 继承数量 | 只能继承一个 | 可以实现多个 |
| 实例字段、构造器 | 有 | 没有 |
| 适合 | 多个子类共享状态和流程 | 定义"能做什么"的契约 |

```java
// Interfaces.java
import java.util.function.Function;

public class Interfaces {
    public static void main(String[] args) {
        // Go 里只要有 Area/Perimeter/Name 方法就自动满足 Shape; Java 必须 implements
        Shape[] shapes = { new Circle(1), new Rect(2, 3) };
        for (Shape s : shapes) {
            System.out.printf("%s area=%.2f perimeter=%.2f%n", s.name(), s.area(), s.perimeter());
        }
        System.out.println(shapes[0].describe());      // default 方法
        System.out.println(Shape.unit().name());       // 接口的静态方法

        // 一个类实现多个接口
        Duck duck = new Duck();
        duck.fly();
        duck.swim();
        Flyer f = duck;                                // 用接口类型的变量持有
        f.fly();
        System.out.println((f instanceof Swimmer) + " " + (f instanceof Duck));

        // default 方法冲突: 必须重写并明确选择
        System.out.println(new C().hi());

        // 函数式接口: 用 lambda 实现 (详见第 15 章)
        Greeter g = n -> "Hello, " + n;
        System.out.println(g.greet("lambda"));
        Function<Integer, Integer> dbl = x -> x * 2;   // 标准库里现成的函数式接口
        System.out.println(dbl.apply(21));
    }
}

interface Shape {
    double area();                       // 默认就是 public abstract
    double perimeter();
    String name();

    default String describe() {          // default 方法: 带实现, 实现类可以不重写
        return name() + " with area " + String.format("%.1f", area());
    }

    static Shape unit() {                // 接口静态方法
        return new Rect(1, 1);
    }
}

record Circle(double r) implements Shape {
    public double area() { return Math.PI * r * r; }        // 实现接口方法必须写 public
    public double perimeter() { return 2 * Math.PI * r; }
    public String name() { return "Circle"; }
}

record Rect(double w, double h) implements Shape {
    public double area() { return w * h; }
    public double perimeter() { return 2 * (w + h); }
    public String name() { return "Rect"; }
}

interface Flyer { void fly(); }
interface Swimmer { void swim(); }

class Duck implements Flyer, Swimmer {                       // 实现多个接口
    public void fly() { System.out.println("duck flies"); }
    public void swim() { System.out.println("duck swims"); }
}

interface A { default String hi() { return "A"; } }
interface B { default String hi() { return "B"; } }

class C implements A, B {
    public String hi() { return A.super.hi() + B.super.hi(); }   // 不重写会编译错误
}

@FunctionalInterface                                         // 可选注解: 让编译器检查只有一个抽象方法
interface Greeter {
    String greet(String name);
}
```

**运行结果：**

```text
Circle area=3.14 perimeter=6.28
Rect area=6.00 perimeter=10.00
Circle with area 3.1
Rect
duck flies
duck swims
duck flies
true true
AB
Hello, lambda
42
```


---

## 第 2 部分（上）小结

1. `class` 把字段、构造器、方法写在一起；写了任何构造器，默认无参构造器就没了。
2. 对象变量是引用，赋值不复制；任何引用类型都可能是 `null`。
3. 字段 `private` + getter/setter 是惯例；只存数据的场景用 `record`。
4. 单继承 + 运行时多态；**父类方法能分发到子类重写**，这是 Go 嵌入做不到的。
5. 接口要显式 `implements`，可以有 `default` 方法；只有一个抽象方法的接口就是函数式接口。

下一个文件（第 2 部分下）：第 14–18 章，包括枚举、密封类、模式匹配、内部类与 lambda、包装类型的坑、`equals/hashCode/Comparable`、泛型。
