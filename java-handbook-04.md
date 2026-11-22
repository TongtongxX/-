# Java 语法学习手册（面向 Go 开发者）

## 第 4 部分：错误处理与资源管理

> 本文件包含第 26–28 章：异常体系、`try / catch / finally` 与自定义异常、`try-with-resources`。
> 这是 Go 开发者思维转换最大的地方之一：Go 的错误是**返回值**，Java 的错误是**自动向上传播的对象**。

**使用约定（同前）**：每个代码块第一行注释是文件名，保存为同名文件后执行 `java 文件名.java` 即可（JDK 21）。带 `main` 的类写在文件最前面，辅助类写在后面。

---

## 26. 异常体系：checked 与 unchecked，对比 error / panic

**和 Go 的差异**

| | Go | Java |
|---|---|---|
| 表示 | `error` 值（接口） | 异常对象（`Throwable` 的子类） |
| 传递方式 | 作为返回值，每一层 `if err != nil` 手动处理或返回 | `throw` 抛出后**自动向上传播**，直到某一层 `catch` |
| 编译器是否强制处理 | 不强制（忽略返回值也能编译） | **checked 异常强制**，unchecked 不强制 |
| 包装 | `fmt.Errorf("...: %w", err)` | `new XxxException("...", cause)` |
| 判断类型 | `errors.Is` / `errors.As` | `catch (Type e)` / `instanceof` / `getCause()` |
| 不可恢复的错误 | `panic` / `recover` | `RuntimeException`（常用）/ `Error`（JVM 级别） |
| 调用栈 | 默认没有 | 自动记录栈追踪 |

**异常的层次**

```text
Throwable
├── Error                        JVM 级严重错误，不要捕获（OutOfMemoryError、StackOverflowError）
└── Exception
    ├── RuntimeException         unchecked：不强制处理
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   │   └── NumberFormatException
    │   ├── IllegalStateException
    │   ├── IndexOutOfBoundsException
    │   │   └── ArrayIndexOutOfBoundsException
    │   ├── ArithmeticException
    │   ├── ClassCastException
    │   ├── UnsupportedOperationException
    │   └── ConcurrentModificationException
    └── 其他 Exception            checked：必须处理或声明
        ├── IOException
        ├── InterruptedException
        └── SQLException ……
```

- **checked 异常**：继承 `Exception` 但不是 `RuntimeException`。编译器强制你要么 `catch`，要么在方法签名里用 `throws` 声明继续往上抛。这一点最接近 Go 的"错误必须处理"，只不过检查发生在编译期。
- **unchecked 异常**：`RuntimeException` 的子类。不需要声明也不需要捕获，没人处理就一路传到线程顶端：主线程会打印栈追踪并以非 0 状态退出。
- **`Error`**：JVM 层面的严重问题，通常不应该捕获。
- 怎么选：
  - 调用方**无法合理恢复**的编程错误（参数不合法、状态不对）→ unchecked，如 `IllegalArgumentException`、`IllegalStateException`。
  - 外部世界的失败（文件、网络、数据库）→ 传统上用 checked。
  - 不少现代项目更倾向于 unchecked，因为 checked 异常会让签名层层带 `throws`，在 lambda / Stream 里也很难用（第 24 章）。
- 异常**不是**普通控制流：抛出和捕获的开销比普通返回大得多，别用它代替 `if`。
- 异常链：`new IllegalStateException("msg", cause)` ≈ Go 的 `%w`；`getCause()` ≈ `errors.Unwrap`。
- 常用方法：`getMessage()`、`getCause()`、`getStackTrace()`、`printStackTrace()`（打印到 stderr）、`toString()`（`类名: 消息`）。

```java
// ExceptionDemo.java
import java.io.IOException;
import java.util.*;

public class ExceptionDemo {
    public static void main(String[] args) {
        checkedVsUnchecked();
        hierarchy();
        commonRuntimeExceptions();
        chaining();
        errorsAreDifferent();
    }

    // checked 异常: 必须在签名里用 throws 声明
    static String readConfig(String name) throws IOException {
        if (name.isEmpty()) throw new IOException("config name is empty");
        return "config:" + name;
    }

    // unchecked 异常: 不需要声明
    static int parsePort(String s) {
        int port = Integer.parseInt(s);        // 可能抛 NumberFormatException
        if (port < 1 || port > 65535) throw new IllegalArgumentException("port out of range: " + port);
        return port;
    }

    static void checkedVsUnchecked() {
        // checked: 编译器强制处理 -- 要么 catch, 要么在自己的签名上继续 throws
        try {
            System.out.println(readConfig("app"));
            System.out.println(readConfig(""));
            System.out.println("not reached");             // 异常发生后, try 块里剩下的代码不再执行
        } catch (IOException e) {
            System.out.println("caught IOException: " + e.getMessage());
        }

        // unchecked: 不处理也能编译, 异常会一直向上传播
        System.out.println(parsePort("8080"));
        try {
            parsePort("70000");
        } catch (IllegalArgumentException e) {
            System.out.println(e.getMessage());
        }
    }

    // 沿着继承链往上走, 看看异常的"家谱"
    static String chain(Class<?> c) {
        StringBuilder sb = new StringBuilder(c.getSimpleName());
        for (Class<?> p = c.getSuperclass(); p != null; p = p.getSuperclass()) {
            sb.append(" -> ").append(p.getSimpleName());
        }
        return sb.toString();
    }

    static void hierarchy() {
        System.out.println(chain(NumberFormatException.class));
        System.out.println(chain(ArrayIndexOutOfBoundsException.class));
        System.out.println(chain(IOException.class));
        System.out.println(chain(StackOverflowError.class));
    }

    // 常见的 unchecked 异常, 以及触发它们的写法
    static void commonRuntimeExceptions() {
        Map<String, Runnable> cases = new LinkedHashMap<>();
        cases.put("null 调用方法", () -> { String s = null; s.length(); });
        cases.put("数组越界", () -> { int[] a = new int[1]; a[1] = 0; });
        cases.put("类型转换", () -> { Object o = "text"; Integer i = (Integer) o; });
        cases.put("整数除以 0", () -> { int z = 0; System.out.println(1 / z); });
        cases.put("数字格式", () -> Integer.parseInt("x"));
        cases.put("修改不可变集合", () -> List.of(1).add(2));
        for (var entry : cases.entrySet()) {
            try {
                entry.getValue().run();
            } catch (RuntimeException e) {
                System.out.println(entry.getKey() + " -> " + e.getClass().getSimpleName());
            }
        }
    }

    // 异常链: ≈ Go 的 fmt.Errorf("...: %w", err)
    static void load(String name) {
        try {
            readConfig(name);
        } catch (IOException e) {
            throw new IllegalStateException("load failed for '" + name + "'", e);
        }
    }

    static void chaining() {
        try {
            load("");
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
            System.out.println("cause: " + e.getCause());                    // ≈ errors.Unwrap
            System.out.println(e.getCause() instanceof IOException);         // ≈ errors.As
            Throwable root = e;
            while (root.getCause() != null) root = root.getCause();
            System.out.println("root: " + root.getClass().getSimpleName());
            System.out.println("thrown in: " + root.getStackTrace()[0].getMethodName());
        }
    }

    static int depth = 0;

    static void recurse() {
        depth++;
        recurse();
    }

    static void errorsAreDifferent() {
        try {
            recurse();
        } catch (StackOverflowError e) {                 // Error: 通常不应该捕获, 这里只是演示
            System.out.println("StackOverflowError, depth > 1000: " + (depth > 1000));
        }
    }
}
```

**运行结果：**

```text
config:app
caught IOException: config name is empty
8080
port out of range: 70000
NumberFormatException -> IllegalArgumentException -> RuntimeException -> Exception -> Throwable -> Object
ArrayIndexOutOfBoundsException -> IndexOutOfBoundsException -> RuntimeException -> Exception -> Throwable -> Object
IOException -> Exception -> Throwable -> Object
StackOverflowError -> VirtualMachineError -> Error -> Throwable -> Object
null 调用方法 -> NullPointerException
数组越界 -> ArrayIndexOutOfBoundsException
类型转换 -> ClassCastException
整数除以 0 -> ArithmeticException
数字格式 -> NumberFormatException
修改不可变集合 -> UnsupportedOperationException
load failed for ''
cause: java.io.IOException: config name is empty
true
root: IOException
thrown in: readConfig
StackOverflowError, depth > 1000: true
```


## 27. try / catch / finally 与自定义异常

**和 Go 的差异**

- 语法：`try { ... } catch (A e) { ... } catch (B | C e) { ... } finally { ... }`。
- **多个 `catch` 要把子类写在前面、父类写在后面**，否则编译错误（父类已经把子类拦截了）。
- 多重捕获：`catch (IOException | SQLException e)`，用 `|` 连接互不为子类关系的类型。
- 对 checked 异常，如果 `try` 块里**不可能抛出**它，写 `catch` 会编译报错。
- `finally` 无论是否发生异常都会执行（`System.exit()` 或 JVM 崩溃除外），≈ Go 的 `defer`，但它的作用域是 **try 块**而不是整个函数，也**不能像 `defer` 那样修改已确定的返回值**：`return x;` 里的值在进入 `finally` 之前就已经确定了。
- `finally` 里的两个坑：
  - 在 `finally` 里写 `return` 会**吞掉**正在传播的异常。
  - 在 `finally` 里抛出新异常会**覆盖**原来的异常。
- 在 `try` 块里声明的变量，`catch` 和 `finally` 里看不到。
- **不要吞异常**：空的 `catch {}` 会让错误悄无声息地消失。至少要记录或重新抛出。
- 重新抛出：`throw e;`；包装后抛出：`throw new XxxException("上下文", e);`，**别丢掉原来的 cause**。
- 子类重写方法时，`throws` 的 checked 异常**不能比父类更宽**。
- **自定义异常**：继承 `Exception`（checked）或 `RuntimeException`（unchecked），通常提供 `(message)` 和 `(message, cause)` 两个构造器，也可以带额外字段（如错误码）。≈ Go 里自定义 error 类型 + 哨兵错误；判断时用 `catch` 具体类型，替代 `errors.Is` / `errors.As`。

```java
// TryCatchDemo.java
public class TryCatchDemo {
    public static void main(String[] args) {
        multiCatch();
        finallyDemo();
        customExceptions();
    }

    static void multiCatch() {
        Object[] inputs = {"42", "abc", null, 7};
        for (Object o : inputs) {
            try {
                String str = (String) o;                      // 7 -> ClassCastException
                int n = Integer.parseInt(str.trim());         // null -> NPE, "abc" -> NumberFormatException
                System.out.println("ok " + n);
            } catch (NumberFormatException e) {               // 具体的子类写在前面
                System.out.println("bad number: " + e.getMessage());
            } catch (ClassCastException | NullPointerException e) {   // 多重捕获
                System.out.println("bad input: " + e.getClass().getSimpleName());
            }
        }
    }

    static String order() {
        StringBuilder sb = new StringBuilder();
        try {
            sb.append("try ");
            throw new RuntimeException("x");
        } catch (RuntimeException e) {
            sb.append("catch ");
        } finally {
            sb.append("finally");                             // 无论是否异常都会执行
        }
        return sb.toString();
    }

    static int returnIsFixedBeforeFinally() {
        int x = 1;
        try {
            return x;                                         // 返回值在这里已经确定为 1
        } finally {
            x = 99;                                           // 不影响返回值 (Go 的 defer 可以改命名返回值)
        }
    }

    static int finallyReturnSwallowsException() {
        try {
            throw new RuntimeException("lost");
        } finally {
            return -1;                                        // 坑: finally 里的 return 吞掉了异常
        }
    }

    static void finallyDemo() {
        System.out.println(order());
        System.out.println(returnIsFixedBeforeFinally());
        System.out.println(finallyReturnSwallowsException());

        // 没有 AutoCloseable 的场景, try/finally ≈ defer
        System.out.println("acquire");
        try {
            System.out.println("working");
        } finally {
            System.out.println("release");
        }
    }

    static long withdraw(long balance, long amount) throws InsufficientFundsException {
        if (amount > balance) {
            throw new InsufficientFundsException("insufficient funds", amount - balance);
        }
        return balance - amount;
    }

    static String loadConfig(String key) {
        try {
            return Integer.toString(Integer.parseInt(key));   // 故意让它失败
        } catch (NumberFormatException e) {
            throw new ConfigException("invalid config key '" + key + "'", e);   // 包装并重新抛出
        }
    }

    static void customExceptions() {
        try {
            System.out.println(withdraw(100, 30));
            withdraw(100, 150);
        } catch (InsufficientFundsException e) {              // checked: 必须处理
            System.out.println(e.getMessage() + ", shortfall=" + e.getShortfall());
            System.out.println(e);                            // toString(): 类名: 消息
            System.out.println("thrown in: " + e.getStackTrace()[0].getMethodName());
        }

        try {
            loadConfig("db.url");
        } catch (ConfigException e) {                         // unchecked: 想处理才处理
            System.out.println(e.getMessage() + ", cause=" + e.getCause().getClass().getSimpleName());
        }
    }
}

// checked 自定义异常: 继承 Exception, 带额外字段
class InsufficientFundsException extends Exception {
    private final long shortfall;

    InsufficientFundsException(String message, long shortfall) {
        super(message);
        this.shortfall = shortfall;
    }

    long getShortfall() { return shortfall; }
}

// unchecked 自定义异常: 继承 RuntimeException, 提供带 cause 的构造器
class ConfigException extends RuntimeException {
    ConfigException(String message) { super(message); }
    ConfigException(String message, Throwable cause) { super(message, cause); }
}
```

**运行结果：**

```text
ok 42
bad number: For input string: "abc"
bad input: NullPointerException
bad input: ClassCastException
try catch finally
1
-1
acquire
working
release
70
insufficient funds, shortfall=50
InsufficientFundsException: insufficient funds
thrown in: withdraw
invalid config key 'db.url', cause=NumberFormatException
```


## 28. try-with-resources，对比 defer

**和 Go 的差异**

- Go 用 `defer f.Close()` 释放资源，是**函数级**的，而且需要你自己记得写。Java 用 `try (资源) { ... }`，**块级**：块结束时（无论正常结束还是抛异常）自动调用 `close()`。
- 资源类型必须实现 `AutoCloseable`（文件流、`Scanner`、数据库连接等标准库类型都实现了）。
- **多个资源按声明的相反顺序关闭**，和 `defer` 的后进先出一致。
- 资源在**块结束时就释放**，不用等整个方法返回。在循环里，每次迭代都会关闭一次；Go 里在循环中写 `defer` 会一直堆积到函数结束，是个常见的坑。
- 如果带 `catch` / `finally`，它们在资源**关闭之后**执行。
- **被抑制的异常（suppressed）**：如果块里抛了异常，`close()` 又抛了异常，主异常会保留，`close()` 的异常通过 `e.getSuppressed()` 取出来。Go 里 `defer` 中的错误往往被直接忽略。
- Java 9+ 可以直接把已有的、实际上不会再被赋值的变量放进去：`try (existing) { ... }`。
- 自己的类要当资源用，实现 `AutoCloseable` 并重写 `close()`。
- 不用 `try-with-resources` 又忘了 `close()`，就会泄漏资源（GC 不会可靠地帮你关文件）。
- 对没有实现 `AutoCloseable` 的清理逻辑（如解锁，第 31 章），用 `try / finally`。

```java
// ResourceDemo.java
import java.io.*;

public class ResourceDemo {
    public static void main(String[] args) {
        basics();
        System.out.println("---");
        suppressed();
        System.out.println("---");
        realResource();
    }

    static void basics() {
        // 1. 基本用法: 离开块时自动调用 close()
        try (Res a = new Res("A")) {
            System.out.println("using " + a.name);
        }

        // 2. 多个资源: 按声明的相反顺序关闭 (和 defer 的后进先出一致)
        try (Res a = new Res("B1"); Res b = new Res("B2")) {
            System.out.println("using both");
        }

        // 3. 发生异常时也会关闭, 并且先关闭, 再进入 catch / finally
        try (Res a = new Res("C")) {
            throw new IllegalStateException("boom");
        } catch (IllegalStateException e) {
            System.out.println("caught " + e.getMessage());
        } finally {
            System.out.println("finally");
        }

        // 4. 循环里每次迭代都会关闭 (Go 里循环中的 defer 要等到函数结束才执行)
        for (int i = 1; i <= 2; i++) {
            try (Res r = new Res("loop" + i)) {
                System.out.println("iteration " + i);
            }
        }

        // 5. Java 9+: 直接使用已有的 (实际上不会再被赋值的) 变量
        Res existing = new Res("existing");
        try (existing) {
            System.out.println("using existing");
        }
    }

    static void suppressed() {
        // close() 本身也抛异常: 主异常保留, close 的异常被"抑制"
        try (Res bad = new Res("D", true)) {
            throw new IllegalStateException("main failure");
        } catch (IllegalStateException e) {
            System.out.println("main: " + e.getMessage());
            for (Throwable s : e.getSuppressed()) {
                System.out.println("suppressed: " + s.getMessage());
            }
        }
    }

    static void realResource() {
        // 标准库里的资源: BufferedReader 实现了 AutoCloseable (读取字符串, 不需要文件)
        try (BufferedReader br = new BufferedReader(new StringReader("line1\nline2"))) {
            String line;
            while ((line = br.readLine()) != null) {
                System.out.println("read " + line);
            }
        } catch (IOException e) {                             // readLine / close 抛 checked 的 IOException
            System.out.println("io error: " + e.getMessage());
        }
    }
}

class Res implements AutoCloseable {
    final String name;
    private final boolean failOnClose;

    Res(String name) {
        this(name, false);
    }

    Res(String name, boolean failOnClose) {
        this.name = name;
        this.failOnClose = failOnClose;
        System.out.println("open " + name);
    }

    @Override
    public void close() {                                     // 可以不声明 throws
        System.out.println("close " + name);
        if (failOnClose) throw new IllegalStateException("close failed: " + name);
    }
}
```

**运行结果：**

```text
open A
using A
close A
open B1
open B2
using both
close B2
close B1
open C
close C
caught boom
finally
open loop1
iteration 1
close loop1
open loop2
iteration 2
close loop2
open existing
using existing
close existing
---
open D
close D
main: main failure
suppressed: close failed: D
---
read line1
read line2
```


---

## 第 4 部分小结

1. Go 的错误是返回值，Java 的异常会自动向上传播；**checked 异常**编译器强制处理，**unchecked**（`RuntimeException`）不强制，`Error` 不要捕获。
2. 包装异常时保留 cause（`new XxxException("msg", cause)`），≈ `%w`；别写空的 `catch`。
3. 多个 `catch` 子类在前；`finally` 一定执行，但里面不要 `return` 或抛新异常；`return` 的值在进入 `finally` 前就已确定。
4. 自定义异常：继承 `Exception`（checked）或 `RuntimeException`（unchecked），提供 `(message)` 和 `(message, cause)` 构造器。
5. 资源用 `try-with-resources`：块级、自动关闭、多个资源后进先出，`close()` 的异常会被抑制而不是覆盖主异常。

下一个文件进入第 5 部分：并发（对比 goroutine / channel）：`Thread`、线程池、`synchronized` / `volatile` / `Atomic*`、并发容器与 `BlockingQueue`、`CompletableFuture`、虚拟线程。
