# Java 语法学习手册（面向 Go 开发者）

> 读者：熟悉 Go，想快速上手 Java。
> 原则：相似的地方一个例子带过，不同的地方重点讲。
> 所有代码都在 JDK 21 下实际运行过，"运行结果"是真实输出。

## 使用约定

1. 每个 Java 代码块的第一行注释是文件名，例如 `// Hello.java`。把代码块保存为同名文件，然后执行 `java Hello.java` 即可。不需要先 `javac`，也不需要任何构建工具。
2. 需要 JDK 21 或更高版本，用 `java -version` 检查。用到较新语法的地方会标注最低版本。
3. 每章先看「和 Go 的差异」，再看代码。代码注释里会标出 Go 的对应写法。
4. 本手册会分多个文件逐步生成，最后可合并为一份。本文件包含第 0、1 部分。
5. Windows 终端如果中文输出乱码，运行时加参数：`java -Dstdout.encoding=UTF-8 Hello.java`，或先执行 `chcp 65001`。

## 全手册路线

| 部分 | 内容 | 状态 |
|---|---|---|
| 第 0 部分 | 环境与运行 | 本文件 |
| 第 1 部分 | 相似部分速览（类型、控制流、方法、数组、字符串） | 本文件 |
| 第 2 部分 | 真正的区别：类、引用、接口、泛型等 | 待生成 |
| 第 3 部分 | 集合与数据处理 | 待生成 |
| 第 4 部分 | 错误处理与资源管理 | 待生成 |
| 第 5 部分 | 并发 | 待生成 |
| 第 6 部分 | 常用标准库 | 待生成 |
| 附录 | 对照表、常见坑、综合项目 | 待生成 |

---

# 第 0 部分：环境与运行

## 1. 第一个程序

**和 Go 的差异**

| | Go | Java |
|---|---|---|
| 入口 | `package main` + `func main()` | 类里的 `public static void main(String[] args)` |
| 代码放哪 | 包级函数、包级变量 | 一切都必须在类里 |
| 语句结尾 | 换行自动补分号 | 必须写分号 `;` |
| 打印 | `fmt.Println` | `System.out.println` |
| 运行 | `go run x.go` | `java X.java`（源文件模式） |
| 产物 | 原生二进制 | `.class` 字节码，由 JVM 执行 |

- `main` 的签名是固定的，写错一个修饰符就找不到入口。
- `args` 不包含程序名，和 Go 的 `os.Args[0]` 不同。
- `java X.java` 是 Java 11 起提供的源文件模式：在内存中编译并运行，不产生 `.class` 文件。正式项目仍然是 `javac` 编译加 `java` 运行（见附录）。

```java
// Hello.java
public class Hello {
    // 入口方法签名固定: public static void main(String[] args)
    public static void main(String[] args) {
        System.out.println("Hello, Java!");                          // ≈ fmt.Println
        System.out.print("no newline, ");                           // ≈ fmt.Print
        System.out.printf("%s has %d args%n", "main", args.length); // ≈ fmt.Printf, %n 表示换行
        for (String a : args) {
            System.out.println("arg: " + a);
        }
    }
}
```

**运行结果：**

```text
Hello, Java!
no newline, main has 0 args
```


带参数运行：

```bash
java Hello.java a b c
```

```text
Hello, Java!
no newline, main has 3 args
arg: a
arg: b
arg: c
```

### 同一个文件里放多个类

源文件模式运行的是文件里**第一个**顶层类的 `main`。所以本手册里的约定是：带 `main` 的类写在最前面，辅助类写在它后面。

```java
// TwoClasses.java
public class TwoClasses {
    public static void main(String[] args) {
        Helper h = new Helper();
        System.out.println(h.hi());
    }
}

// 辅助类必须放在 main 所在类的后面
class Helper {
    String hi() {
        return "hi from Helper";
    }
}
```

**运行结果：**

```text
hi from Helper
```


## 2. 包（package）与目录结构

**和 Go 的差异**

| | Go | Java |
|---|---|---|
| 包名 | 通常取目录名，简短 | 文件顶部 `package` 声明，惯例是反向域名：`com.example.util` |
| 目录与包 | 一个目录一个包 | 目录结构必须和包名一致：`com/example/util/` |
| 一个文件里 | 任意多个类型、函数 | 最多一个 `public` 类，且文件名必须等于类名 |
| 导入 | `import "pkg"`，用 `pkg.Name` | `import com.example.util.MathUtil;`，直接用 `MathUtil` |
| 可见性 | 首字母大小写 | `public` / `protected` / 默认（包私有）/ `private` |
| 自动导入 | 无 | `java.lang.*` 自动导入（`String`、`System`、`Math` 等） |

- Java 的 `import` 导入的是单个类（或用 `*` 导入全部），不是整个包的命名空间。
- 不写任何访问修饰符时，是**包私有**：同一个包内可见，别的包看不到。这个默认值 Go 里没有对应概念。
- 源文件模式（上面的 `java X.java`）适合单文件。多个包、多个文件要用 `javac`。

目录结构：

```text
demo/
└── com/
    └── example/
        ├── App.java
        └── util/
            └── MathUtil.java
```

```java
// com/example/util/MathUtil.java
package com.example.util;

public class MathUtil {
    public static int square(int x) {
        return x * x;
    }

    static int hidden() {      // 没写修饰符: 包私有, 其他包看不到
        return 1;
    }
}
```

```java
// com/example/App.java
package com.example;

import com.example.util.MathUtil;

public class App {
    public static void main(String[] args) {
        System.out.println(MathUtil.square(7));
        // MathUtil.hidden();   // 编译错误: hidden() 不是 public
    }
}
```

编译并运行（在 `demo/` 目录下执行）：

```bash
javac -d out com/example/App.java com/example/util/MathUtil.java
java -cp out com.example.App
```

```text
49
```

`-d out` 指定 `.class` 文件的输出目录，`-cp out` 指定运行时去哪里找类。JDK 22 起支持多文件源文件模式，可以直接 `java com/example/App.java`，但 JDK 21 不行。

## 3. 构建工具

真实项目通常用 Maven 或 Gradle 管理依赖、编译和打包，作用类似 `go.mod` + `go build`。本手册所有例子都不需要它们，附录 D 会简单介绍。

---

# 第 1 部分：相似部分速览

## 4. 基本类型与变量

**和 Go 的差异**

| Go | Java | 备注 |
|---|---|---|
| `int8/16/32/64` | `byte/short/int/long` | Java 的 `int` 固定 32 位；Go 的 `int` 在 64 位平台是 64 位 |
| `uint*` | 无 | Java 没有无符号类型 |
| `byte`（= `uint8`） | `byte` | Java 的 `byte` **有符号**，范围 -128~127 |
| `float32/64` | `float/double` | |
| `bool` | `boolean` | |
| `rune` | `int`（码点）/ `char`（UTF-16 单元） | `char` 不是完整的 Unicode 字符 |
| `string` | `String` | `String` 是引用类型，不是基本类型 |

- 基本类型只有 8 个：`byte short int long float double boolean char`。其余全是引用类型，包括 `String` 和数组。
- **局部变量没有零值**：必须先赋值才能用，否则编译报错。只有字段（类里的变量）才有默认值。
- 引用类型字段的默认值是 `null`，不是 Go 里 `string` 的 `""`。
- `var x = 1;` 是局部变量类型推断，≈ Go 的 `:=`。只能用于方法内部的局部变量。
- `final` 表示变量不能再被赋值，≈ Go 的 `const`，但不限于编译期常量。类级别常量写成 `static final`。
- 窄化转换必须写强转：`(int) 3.99`，对应 Go 的 `int(3.99)`。宽化转换自动发生，Go 里 `int + int64` 是编译错误，Java 里不是。
- 内层作用域不能重新声明和外层同名的局部变量（Go 允许遮蔽）。
- 没有 `a, b = b, a` 这种交换写法，要用临时变量。
- 整数溢出不报错，静默回绕。
- 数字字面量可以用下划线分隔：`1_000_000`。`long` 字面量加 `L`，`float` 字面量加 `f`。

```java
// Types.java
public class Types {
    static final double PI = 3.14159;  // ≈ Go 的包级 const
    static int count;                  // 字段有默认值: 0
    static String text;                // 引用类型字段默认 null (Go 的 string 零值是 "")
    static boolean flag;               // false

    public static void main(String[] args) {
        int i = 42;
        long big = 10_000_000_000L;    // long 字面量要加 L
        double d = 3.14;
        float f = 1.5f;                // float 字面量要加 f
        boolean ok = true;
        char c = 'A';                  // 单引号, UTF-16 单元, 不是 rune
        System.out.println(i + " " + big + " " + d + " " + f + " " + ok + " " + c);

        var name = "Java";             // 类型推断 ≈ Go 的 :=
        final int max = 100;           // 不能再赋值
        // max = 200;                  // 编译错误
        System.out.println(name + " " + max + " " + PI);

        // 字段有默认值
        System.out.println(count + " " + text + " " + flag);

        // 局部变量没有零值, 必须先赋值
        int x;
        // System.out.println(x);      // 编译错误: 变量可能尚未初始化
        x = 1;
        System.out.println(x);

        // 溢出: 静默回绕, 不报错
        int maxInt = Integer.MAX_VALUE;
        System.out.println(maxInt + " " + (maxInt + 1));
        System.out.println(Long.MAX_VALUE);

        // 类型转换
        long l = i;                    // 宽化: 自动
        int fromD = (int) 3.99;        // 窄化: 必须强转, 截断小数
        long sum = i + l;              // int + long 自动提升为 long (Go 里是编译错误)
        System.out.println(l + " " + fromD + " " + sum);
        System.out.println(7 / 2 + " " + 7 / 2.0 + " " + 7 % 3 + " " + -7 % 3);

        // byte 有符号 (Go 的 byte 无符号)
        byte wrapped = (byte) 200;
        System.out.println(wrapped + " " + (wrapped & 0xFF));

        // char 参与运算会变成 int
        char next = (char) (c + 1);
        System.out.println(next + " " + (int) c + " " + (c + 1));

        // 交换: 没有 a, b = b, a
        int a = 1, b = 2;
        int tmp = a;
        a = b;
        b = tmp;
        System.out.println(a + " " + b);
    }
}
```

**运行结果：**

```text
42 10000000000 3.14 1.5 true A
Java 100 3.14159
0 null false
1
2147483647 -2147483648
9223372036854775807
42 3 84
3 3.5 1 -1
-56 200
B 65 66
2 1
```


## 5. 运算符与控制流

**和 Go 的差异**

- `if` / `while` 的条件**必须带括号**。没有 `if x := f(); x > 0` 这种带初始化语句的写法。
- 有三元运算符 `cond ? a : b`，Go 没有。
- 循环有四种：经典 `for(;;)`、`while`、`do-while`、增强 for（`for (T v : coll)`）。Go 的 `for cond {}` 在 Java 里写成 `while`。
- 增强 for ≈ Go 的 `for _, v := range`，但**拿不到下标**。需要下标时用经典 for。
- `i++` 是表达式，可以写 `arr[i++]`（Go 里 `++` 只是语句）。
- **传统 `switch` 默认 fallthrough**，必须自己写 `break`，和 Go 正好相反。
- 新式 `switch`（Java 14+）用 `->`，没有 fallthrough，还能作为表达式返回值，推荐优先使用。
- `&&` `||` 短路，`&` `|` 在布尔值上是不短路的版本。
- 带标签的 `break` / `continue` 和 Go 一致。

```java
// ControlFlow.java
public class ControlFlow {
    public static void main(String[] args) {
        // if: 条件必须带括号
        int score = 85;
        if (score >= 90) {
            System.out.println("A");
        } else if (score >= 80) {
            System.out.println("B");
        } else {
            System.out.println("C");
        }

        // 三元运算符 (Go 没有)
        String result = score >= 60 ? "pass" : "fail";
        System.out.println(result);

        // 经典 for
        for (int i = 0; i < 3; i++) {
            System.out.println("i=" + i);
        }

        // while / do-while (Go 里写作 for cond {})
        int n = 3;
        while (n > 0) {
            n--;
        }
        do {
            n++;
        } while (n < 3);
        System.out.println("n=" + n);

        // 增强 for ≈ for _, v := range, 但没有下标
        int[] nums = {10, 20, 30};
        for (int v : nums) {
            System.out.println("v=" + v);
        }
        for (int i = 0; i < nums.length; i++) {   // 需要下标时用经典 for
            System.out.println(i + ":" + nums[i]);
        }

        // i++ 是表达式 (Go 里只是语句)
        int k = 0;
        int[] buf = new int[3];
        buf[k++] = 7;
        System.out.println(k + " " + buf[0]);

        // 带标签的 break
        outer:
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                if (i * j == 2) break outer;
                System.out.println(i + "," + j);
            }
        }

        // 传统 switch: 不写 break 就会 fallthrough (Go 正好相反)
        int day = 2;
        switch (day) {
            case 1:
                System.out.println("Mon");
                break;
            case 2:
                System.out.println("Tue");
                // 这里没有 break, 会继续执行下一个 case
            case 3:
                System.out.println("Wed (fallthrough)");
                break;
            default:
                System.out.println("other");
        }

        // 新式 switch (Java 14+): 用 ->, 无 fallthrough, 可以当表达式
        for (int d : new int[]{1, 2, 5}) {
            String label = switch (d) {
                case 1, 7 -> "weekend-ish";      // 多个值用逗号
                case 2 -> "Tue";
                default -> {
                    String tmp = "day" + d;      // 多行分支用 {}, 用 yield 返回值
                    yield tmp;
                }
            };
            System.out.println(d + " -> " + label);
        }

        // switch 也支持 String 和 enum
        String cmd = "stop";
        switch (cmd) {
            case "start" -> System.out.println("starting");
            case "stop" -> System.out.println("stopping");
            default -> System.out.println("unknown");
        }
    }
}
```

**运行结果：**

```text
B
pass
i=0
i=1
i=2
n=3
v=10
v=20
v=30
0:10
1:20
2:30
1 7
0,0
0,1
0,2
1,0
1,1
Tue
Wed (fallthrough)
1 -> weekend-ish
2 -> Tue
5 -> day5
stopping
```


## 6. 方法

**和 Go 的差异**

- 方法必须定义在类里。`static` 方法 ≈ Go 的包级函数，调用时不需要对象。
- 返回类型写在方法名前面：`int add(int a, int b)`。
- **没有多返回值**。需要返回多个值时，用 `record`（第 11 章）、数组或自定义类。Go 的 `(T, error)` 惯用法在 Java 里用异常表达（第 26 章）。
- **支持重载**：同名方法，参数不同即可。Go 不支持。
- **没有默认参数**，用重载模拟。
- 可变参数写作 `int... nums`，方法内是 `int[]`。
- 参数永远是**值传递**。传对象时，传的是引用的副本：能修改对象内容，不能让调用方的变量指向别的对象。
- 方法不是一等公民，不能直接当值传递。需要时用 lambda 或方法引用（第 15 章）。
- 没有命名返回值，没有 `defer`（对应的是 `try-with-resources`，第 28 章）。

```java
// Methods.java
public class Methods {
    // 重载: 同名, 参数不同
    static int add(int a, int b) { return a + b; }
    static double add(double a, double b) { return a + b; }
    static int add(int a, int b, int c) { return a + b + c; }

    // 没有默认参数, 用重载模拟
    static String greet(String name) { return greet(name, "Hello"); }
    static String greet(String name, String prefix) { return prefix + ", " + name; }

    // 可变参数: 方法内部 nums 就是 int[]
    static int sum(int... nums) {
        int total = 0;
        for (int n : nums) total += n;
        return total;
    }

    // 没有多返回值: 用 record 打包 (record 详见第 11 章)
    record DivResult(int quotient, int remainder) {}

    static DivResult divmod(int a, int b) {
        return new DivResult(a / b, a % b);
    }

    // 参数是值传递
    static void inc(int x) { x++; }                          // 只改了副本
    static void fill(int[] arr) { arr[0] = 99; }             // 副本指向同一个数组, 能改内容
    static void reassign(int[] arr) { arr = new int[]{-1}; } // 只让副本指向新数组, 调用方不受影响

    public static void main(String[] args) {
        System.out.println(add(1, 2));
        System.out.println(add(1.5, 2.5));
        System.out.println(add(1, 2, 3));
        System.out.println(greet("Go"));
        System.out.println(greet("Go", "Hi"));
        System.out.println(sum() + " " + sum(1) + " " + sum(1, 2, 3));
        System.out.println(sum(new int[]{4, 5, 6}));         // 数组可以直接传给可变参数

        DivResult r = divmod(17, 5);
        System.out.println(r.quotient() + " " + r.remainder());
        System.out.println(r);

        int x = 1;
        inc(x);
        System.out.println(x);
        int[] arr = {1, 2, 3};
        fill(arr);
        System.out.println(arr[0]);
        reassign(arr);
        System.out.println(arr[0]);
    }
}
```

**运行结果：**

```text
3
4.0
6
Hello, Go
Hi, Go
0 1 6
15
3 2
DivResult[quotient=3, remainder=2]
1
99
99
```


## 7. 数组

**和 Go 的差异**

- Java 数组是**引用类型**：`int[] b = a` 不会复制，两个变量指向同一个数组。Go 的数组是值类型，赋值会整体复制。
- 数组定长，**没有切片**。需要动态长度时用 `ArrayList`（第 19 章）。
- 数组元素自动初始化为默认值：数字 0、`boolean` false、引用类型 `null`。
- 长度的三种写法容易混：数组是字段 `arr.length`，字符串是方法 `s.length()`，集合是方法 `list.size()`。
- 越界抛 `ArrayIndexOutOfBoundsException`，Go 里是 panic。
- 直接 `System.out.println(arr)` 打印的是类似 `[I@1b6d3586` 的内存标识，要用 `Arrays.toString(arr)`。
- 用 `==` 比较两个数组比的是引用，比内容要用 `Arrays.equals`。
- 类型写在前面，方括号紧跟类型：`int[] a`。

```java
// ArrayDemo.java
import java.util.Arrays;

public class ArrayDemo {
    public static void main(String[] args) {
        int[] a = new int[5];             // 元素自动为 0
        int[] b = {3, 1, 2};              // 字面量初始化
        String[] names = new String[2];   // 引用类型元素默认 null
        System.out.println(Arrays.toString(a));
        System.out.println(Arrays.toString(names));
        System.out.println(b.length);     // 字段, 没有括号

        // 数组是引用类型: 赋值不复制 (Go 的数组赋值会复制)
        int[] c = b;
        c[0] = 100;
        System.out.println(b[0]);
        int[] d = Arrays.copyOf(b, b.length);   // 真正的复制
        d[0] = -1;
        System.out.println(b[0] + " " + d[0]);

        // 常用工具方法
        int[] e = {5, 2, 9, 1};
        Arrays.sort(e);
        System.out.println(Arrays.toString(e));
        System.out.println(Arrays.binarySearch(e, 5));                      // 要求已排序, 返回下标
        System.out.println(Arrays.toString(Arrays.copyOfRange(e, 1, 3)));   // 类似 e[1:3], 但会复制
        Arrays.fill(a, 7);
        System.out.println(Arrays.toString(a));
        System.out.println(Arrays.equals(new int[]{1, 2}, new int[]{1, 2}));   // 比内容
        System.out.println(new int[]{1, 2} == new int[]{1, 2});                // 比引用

        // 二维数组
        int[][] grid = new int[2][3];
        grid[1][2] = 5;
        System.out.println(Arrays.deepToString(grid));
        int[][] jagged = {{1}, {2, 3}, {4, 5, 6}};   // 每行长度可以不同
        for (int[] row : jagged) {
            System.out.println(row.length + " " + Arrays.toString(row));
        }

        // 越界抛异常 (Go 里是 panic)
        try {
            System.out.println(b[10]);
        } catch (ArrayIndexOutOfBoundsException ex) {
            System.out.println("out of bounds: " + ex.getMessage());
        }
    }
}
```

**运行结果：**

```text
[0, 0, 0, 0, 0]
[null, null]
3
100
100 -1
[1, 2, 5, 9]
2
[2, 5]
[7, 7, 7, 7, 7]
true
false
[[0, 0, 0], [0, 0, 5]]
1 [1]
2 [2, 3]
3 [4, 5, 6]
out of bounds: Index 10 out of bounds for length 3
```


## 8. 字符串

**和 Go 的差异**

- `String` 不可变，内部是 UTF-16，不是 UTF-8 字节切片。
- **`==` 比较引用，`equals` 比较内容**。这是 Go 开发者最容易踩的坑，Go 里 `==` 就是比较内容。
- `s.length()` 返回 UTF-16 单元数，不是字符数。emoji 这类字符占 2 个单元。按真实字符遍历用 `codePoints()`。
- 没有 `s[1:3]` 切片语法，用 `substring(1, 3)`。`s.charAt(i)` 返回 `char`，不是字节。
- 拼接用 `+`。循环里反复拼接要用 `StringBuilder`，≈ Go 的 `strings.Builder`。
- 格式化用 `String.format` / `printf` / `"...".formatted(...)`，占位符和 Go 的 `fmt` 很像，换行用 `%n`。
- 文本块 `"""..."""`（Java 15+）≈ Go 的反引号多行字符串，但转义字符仍然生效。
- 字符串和数字互转：`Integer.parseInt` ≈ `strconv.Atoi`，`String.valueOf` ≈ `strconv.Itoa`。解析失败抛 `NumberFormatException`，不是返回 error。

```java
// Strings.java
import java.util.Arrays;

public class Strings {
    public static void main(String[] args) {
        // 常用方法
        String s = "Hello, Java";
        System.out.println(s.length() + " " + s.charAt(0) + " " + s.substring(7) + " " + s.substring(0, 5));
        System.out.println(s.indexOf("Java") + " " + s.contains("lo") + " " + s.startsWith("He"));
        System.out.println(s.toUpperCase() + " " + s.replace("Java", "Go"));
        System.out.println("[" + "  pad  ".trim() + "] [" + " x ".strip() + "]");
        System.out.println("".isEmpty() + " " + "  ".isBlank());
        System.out.println(Arrays.toString("a,b,c".split(",")));
        System.out.println(String.join("-", "a", "b", "c"));
        System.out.println("ab".repeat(3));

        // == 比引用, equals 比内容
        String a = "go";
        String b = new String("go");
        System.out.println((a == b) + " " + a.equals(b));
        System.out.println("apple".compareTo("banana"));      // 负数: 按字典序比较
        System.out.println("A".equalsIgnoreCase("a"));

        // StringBuilder ≈ Go 的 strings.Builder
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 5; i++) {
            sb.append(i).append(',');
        }
        sb.setLength(sb.length() - 1);                       // 去掉末尾的逗号
        System.out.println(sb);
        System.out.println(sb.reverse());

        // 格式化
        System.out.printf("%s is %d years, pi=%.2f%n", "Tom", 30, 3.14159);
        System.out.println(String.format("%5d|%-5d|%05d", 42, 42, 42));
        System.out.println("x=%d".formatted(10));

        // 文本块 (Java 15+)
        String json = """
            {
              "name": "Tom",
              "age": 30
            }
            """;
        System.out.print(json);

        // 字符串与数字互转
        int n = Integer.parseInt("123");                     // ≈ strconv.Atoi
        String t = String.valueOf(456);                      // ≈ strconv.Itoa
        double dd = Double.parseDouble("3.5");
        System.out.println(n + 1);                            // 124, n 是 int
        System.out.println(t + "7");                         // 4567, t 是 String, 这里是拼接
        System.out.println(dd * 2);                          // 7.0
        try {
            Integer.parseInt("abc");
        } catch (NumberFormatException e) {
            System.out.println("parse failed: " + e.getMessage());
        }

        // length() 是 UTF-16 单元数, 不是字符数
        String mixed = "a你😀";
        System.out.println(mixed.length() + " " + mixed.codePointCount(0, mixed.length()));
        mixed.codePoints().forEach(cp -> System.out.println(new String(Character.toChars(cp))));
    }
}
```

**运行结果：**

```text
11 H Java Hello
7 true true
HELLO, JAVA Hello, Go
[pad] [x]
true true
[a, b, c]
a-b-c
ababab
false true
-1
true
0,1,2,3,4
4,3,2,1,0
Tom is 30 years, pi=3.14
   42|42   |00042
x=10
{
  "name": "Tom",
  "age": 30
}
124
4567
7.0
parse failed: For input string: "abc"
4 3
a
你
😀
```


---

## 第 0、1 部分小结

读完这两部分，你应该记住这几件事：

1. 一切都在类里，`main` 签名固定，本手册的例子都是 `java X.java` 直接运行。
2. 局部变量没有零值，`byte` 有符号，没有无符号类型，整数溢出静默回绕。
3. 传统 `switch` 默认 fallthrough；新式 `switch ->` 更安全。
4. 没有多返回值，但有重载和可变参数。
5. 数组是引用类型，定长，没有切片。
6. 字符串比较用 `equals`，不要用 `==`。

下一部分（第 2 部分）进入重点：类与对象、引用语义与 null、record、继承、接口、枚举、lambda、泛型等 Java 与 Go 真正不同的地方。
