# Java 语法学习手册（面向 Go 开发者）

## 第 3 部分（上）：集合

> 本文件包含第 19–23 章：`List`、`Map`、`Set` 与队列、不可变集合、遍历与并发修改。
> 第 24–25 章（Stream API、Optional）在下一个文件。

**使用约定（同前）**：每个代码块第一行注释是文件名，保存为同名文件后执行 `java 文件名.java` 即可（JDK 21）。带 `main` 的类写在文件最前面，辅助类写在后面。

## Go 容器 → Java 集合对照

| Go | Java | 说明 |
|---|---|---|
| `[]T`（slice） | `List<T>` → `ArrayList<T>` | 动态数组，最常用 |
| `map[K]V` | `Map<K,V>` → `HashMap`（无序）/ `LinkedHashMap`（插入顺序）/ `TreeMap`（按 key 排序） | |
| `map[T]struct{}` | `Set<T>` → `HashSet` / `LinkedHashSet` / `TreeSet` | Go 没有内置 Set |
| 用 slice 模拟队列、栈 | `ArrayDeque<T>` | 队列、栈、双端队列都用它 |
| `container/heap` | `PriorityQueue<T>` | 用起来比 Go 的 heap 简单得多 |
| `container/list` | `LinkedList<T>` | 几乎不用 |

几条贯穿本部分的规则：

- 集合只能存对象，类型参数要写包装类型：`List<Integer>`，不能写 `List<int>`（第 16 章）。
- **变量类型写接口，右边 `new` 实现类**：`List<String> list = new ArrayList<>();`。
- 集合里的元素和 map 的 key，要正确实现 `equals` / `hashCode`（第 17 章）。
- `HashMap` / `HashSet` 的遍历顺序**不保证**。本手册的例子凡是需要打印顺序的，都用 `TreeMap` / `LinkedHashMap` 等有确定顺序的实现。

---

## 19. List（ArrayList / LinkedList），对比 slice

**和 Go 的差异**

- `List` 是接口，常用实现是 `ArrayList`（动态数组）。`LinkedList` 是链表，随机访问是 O(n)，几乎总是应该选 `ArrayList`。
- 没有 `list[i]` 语法，用 `get(i)` / `set(i, v)`。
- `add` 是**原地修改**，没有返回值。Go 里 `s = append(s, x)` 忘记接收返回值的那类 bug，在 Java 里不存在。
- 常用方法：`add` / `add(index, v)` / `get` / `set` / `remove` / `size` / `isEmpty` / `contains` / `indexOf` / `addAll` / `clear` / `sort` / `forEach` / `removeIf`。
- **`remove` 有两个重载**：`remove(int index)` 按下标删，`remove(Object o)` 按值删。`List<Integer>` 上写 `list.remove(1)` 删的是**下标 1**；要按值删必须写 `list.remove(Integer.valueOf(1))`。这是最经典的坑之一。
- `subList(from, to)` ≈ Go 的 `s[a:b]`，左闭右开，而且也是**视图**：修改视图会写穿到原列表。要独立副本就 `new ArrayList<>(...)`。原列表发生结构性修改（增删）后，再用旧的 `subList` 会抛 `ConcurrentModificationException`。
- **`List` 变量可以是 `null`**，对它调用方法会 NPE。Go 的 `nil` slice 可以直接 `append`，Java 的 `null` 列表不行。
- 赋值只复制引用；`new ArrayList<>(other)` 是浅拷贝。
- `list1.equals(list2)` **比较内容**（数组的 `equals` 比引用）。
- `new ArrayList<>(1000)` ≈ `make([]T, 0, 1000)`，预分配容量。
- Java 21 新增了首尾操作：`getFirst()` / `getLast()` / `addFirst()` / `removeLast()` / `reversed()`。

```java
// ListDemo.java
import java.util.*;

public class ListDemo {
    public static void main(String[] args) {
        // 变量类型写接口, 右边写实现类
        List<Integer> nums = new ArrayList<>();      // ≈ var nums []int
        nums.add(10);                                // ≈ nums = append(nums, 10), 原地修改, 没有返回值
        nums.add(20);
        nums.add(30);
        nums.add(1, 15);                             // 在下标 1 处插入
        System.out.println(nums + " size=" + nums.size());
        System.out.println(nums.get(0) + " " + nums.contains(20) + " " + nums.indexOf(30));
        nums.set(0, 11);                             // ≈ nums[0] = 11
        System.out.println(nums);

        // 遍历
        int total = 0;
        for (int n : nums) total += n;               // ≈ for _, v := range
        for (int i = 0; i < nums.size(); i++) total += nums.get(i);   // 需要下标时
        nums.forEach(n -> System.out.print(n + ";"));
        System.out.println(" total=" + total);

        // remove 的坑: remove(int index) 按下标, remove(Object o) 按值
        List<Integer> xs = new ArrayList<>(List.of(5, 6, 7, 1));
        xs.remove(1);                                // 参数是 int -> 按下标删, 删掉的是 6
        System.out.println(xs);
        xs.remove(Integer.valueOf(1));               // 参数是 Integer -> 按值删, 删掉的是 1
        System.out.println(xs);

        // subList ≈ s[a:b], 但是视图: 修改会写穿到原列表
        List<Integer> big = new ArrayList<>(List.of(0, 1, 2, 3, 4, 5));
        List<Integer> view = big.subList(1, 4);      // 左闭右开
        System.out.println(view);
        view.set(0, 99);
        System.out.println(big);
        List<Integer> copy = new ArrayList<>(big.subList(1, 4));   // 要独立副本就复制
        copy.set(0, -1);
        System.out.println(big + " " + copy);

        // 排序与常用工具
        List<String> words = new ArrayList<>(List.of("pear", "apple", "fig"));
        Collections.sort(words);                                   // 自然顺序
        System.out.println(words);
        words.sort(Comparator.comparing(String::length));          // 按长度, 稳定排序
        System.out.println(words);
        Collections.reverse(words);
        System.out.println(words + " max=" + Collections.max(words));

        // Java 21: 首尾操作
        System.out.println(words.getFirst() + " " + words.getLast() + " " + words.reversed());

        // equals 比内容 (数组的 equals 比引用)
        System.out.println(List.of(1, 2).equals(List.of(1, 2)));

        // 转数组
        Integer[] arr = nums.toArray(new Integer[0]);
        System.out.println(arr.length + " " + arr[0]);

        // null 列表: Go 的 nil slice 可以直接 append, Java 的 null List 不行
        List<String> nothing = null;
        try {
            nothing.add("x");
        } catch (NullPointerException e) {
            System.out.println("null list -> NPE");
        }
    }
}
```

**运行结果：**

```text
[10, 15, 20, 30] size=4
10 true 3
[11, 15, 20, 30]
11;15;20;30; total=152
[5, 7, 1]
[5, 7]
[1, 2, 3]
[0, 99, 2, 3, 4, 5]
[0, 99, 2, 3, 4, 5] [-1, 2, 3]
[apple, fig, pear]
[fig, pear, apple]
[apple, pear, fig] max=pear
apple fig [fig, pear, apple]
true
4 11
null list -> NPE
```


## 20. Map（HashMap / TreeMap / LinkedHashMap），对比 map

**和 Go 的差异**

- 三个常用实现：`HashMap`（无序，最快）、`LinkedHashMap`（保持插入顺序）、`TreeMap`（按 key 排序，还提供 `firstKey` / `floorKey` / `headMap` 等范围查询）。
- 读写：`put(k, v)` ≈ `m[k] = v`，`get(k)` ≈ `m[k]`，`remove(k)` ≈ `delete(m, k)`，`containsKey(k)` ≈ `_, ok := m[k]`。
- **`get` 找不到 key 时返回 `null`，不是零值**，也没有 `v, ok` 这种写法。用 `containsKey` 判断，或者用 `getOrDefault(k, 默认值)`。
- 值是 `Integer` 时，`int x = map.get(k)` 遇到 `null` 会拆箱 NPE（第 16 章）。
- **计数惯用法**：Go 的 `m[k]++` → `m.merge(k, 1, Integer::sum)`。
- **分组惯用法**：Go 的 `m[k] = append(m[k], v)` → `m.computeIfAbsent(k, x -> new ArrayList<>()).add(v)`。
- 其他常用方法：`putIfAbsent`、`compute`、`replaceAll`、`forEach`、`keySet()`、`values()`、`entrySet()`。
- 遍历键值对：`for (Map.Entry<K, V> e : map.entrySet())` ≈ `for k, v := range m`。
- `HashMap` 允许一个 `null` key 和任意个 `null` value；`TreeMap` 不允许 `null` key。
- **不要用可变对象当 key**：放进去之后如果 key 的内容变了，`hashCode` 就变了，再也找不到这个条目。
- 遍历 map 的同时删除元素，会抛 `ConcurrentModificationException`，Go 里这样做是安全的。正确做法见第 23 章。
- Go 的 map 值如果是 struct，`m[k].field = x` 不能编译；Java 里 value 是引用，`map.get(k).setField(x)` 可以直接改。

```java
// MapDemo.java
import java.util.*;

public class MapDemo {
    public static void main(String[] args) {
        Map<String, Integer> ages = new HashMap<>();     // ≈ make(map[string]int)
        ages.put("Alice", 30);                           // ≈ m["Alice"] = 30
        ages.put("Bob", 25);
        ages.put("Alice", 31);                           // 同一个 key 会覆盖
        System.out.println(ages.get("Alice") + " " + ages.size());

        // 找不到返回 null, 不是零值; 没有 v, ok 的写法
        System.out.println(ages.get("Zed"));
        System.out.println(ages.containsKey("Zed") + " " + ages.getOrDefault("Zed", 0));
        try {
            int age = ages.get("Zed");                   // null 拆箱 -> NPE
        } catch (NullPointerException e) {
            System.out.println("unboxing null -> NPE");
        }

        ages.remove("Bob");                              // ≈ delete(m, "Bob")
        System.out.println(ages);

        // 计数: ≈ m[w]++
        String[] words = {"go", "java", "go", "rust", "java", "go"};
        Map<String, Integer> count = new TreeMap<>();    // TreeMap 按 key 排序, 方便看结果
        for (String w : words) {
            count.merge(w, 1, Integer::sum);             // 不存在就放 1, 存在就用 Integer::sum 合并
        }
        System.out.println(count);

        // 分组: ≈ m[k] = append(m[k], v)
        Map<Integer, List<String>> byLen = new TreeMap<>();
        for (String w : words) {
            byLen.computeIfAbsent(w.length(), k -> new ArrayList<>()).add(w);
        }
        System.out.println(byLen);

        // 其他常用方法
        count.putIfAbsent("zig", 0);                     // 不存在才放入
        count.compute("go", (k, v) -> v == null ? 1 : v * 10);
        System.out.println(count);

        // 遍历 ≈ for k, v := range m
        for (var e : count.entrySet()) {                 // var 省掉冗长的 Map.Entry<String, Integer>
            System.out.println(e.getKey() + " -> " + e.getValue());
        }
        System.out.println(count.keySet() + " " + count.values());

        // 三种实现
        Map<String, Integer> linked = new LinkedHashMap<>();   // 保持插入顺序
        linked.put("z", 1);
        linked.put("a", 2);
        linked.put("m", 3);
        System.out.println(linked);
        TreeMap<String, Integer> tree = new TreeMap<>(linked); // 按 key 排序, 支持范围查询
        System.out.println(tree + " " + tree.firstKey() + " " + tree.lastKey());
        System.out.println(tree.headMap("m") + " " + tree.ceilingKey("b"));

        // 可变对象做 key 的坑: 修改之后就找不到了
        List<Integer> key = new ArrayList<>(List.of(1));
        Map<List<Integer>, String> m = new HashMap<>();
        m.put(key, "v");
        key.add(2);
        System.out.println(m.get(key) + " " + m.containsKey(key) + " " + m.size());
    }
}
```

**运行结果：**

```text
31 2
null
false 0
unboxing null -> NPE
{Alice=31}
{go=3, java=2, rust=1}
{2=[go, go, go], 4=[java, rust, java]}
{go=30, java=2, rust=1, zig=0}
go -> 30
java -> 2
rust -> 1
zig -> 0
[go, java, rust, zig] [30, 2, 1, 0]
{z=1, a=2, m=3}
{a=2, m=3, z=1} a z
{a=2} m
null false 1
```


## 21. Set 与 Queue / Deque / PriorityQueue

**和 Go 的差异**

- **Set**：Go 没有内置 Set，通常用 `map[T]struct{}`。Java 有 `HashSet`（无序）、`LinkedHashSet`（插入顺序）、`TreeSet`（排序，支持 `first` / `last` / `floor` / `higher` 等）。
- `add` 返回 `boolean`：元素是新加入的返回 `true`，已存在返回 `false`，很适合做去重判断。
- 集合运算：并集 `addAll`，交集 `retainAll`，差集 `removeAll`，子集判断 `containsAll`。这些方法都是**原地修改**，先复制一份再操作。
- 列表去重并保持顺序：`new ArrayList<>(new LinkedHashSet<>(list))`。
- **队列、栈、双端队列**：Go 标准库没有，通常拿 slice 模拟。Java 统一用 `ArrayDeque`：
  - 队列（FIFO）：`offer` 入队，`poll` 出队，`peek` 看队首。
  - 栈（LIFO）：`push` / `pop` / `peek`，都操作队首。不要用老的 `Stack` 类。
  - 双端：`offerFirst` / `offerLast` / `pollFirst` / `pollLast`。
- 每个操作有两套：`offer` / `poll` / `peek` 在失败时返回 `false` / `null`，`add` / `remove` / `element` 在失败时抛异常。
- `ArrayDeque` 不允许放 `null`。
- **`PriorityQueue`** 是堆，默认最小堆，传 `Comparator` 可以改成最大堆或按字段排序。**只有 `poll` 保证顺序，直接打印或遍历看到的是堆内部的顺序，不是排序结果。**

```java
// SetQueueDemo.java
import java.util.*;

public class SetQueueDemo {
    public static void main(String[] args) {
        // Set ≈ Go 的 map[T]struct{}
        Set<String> set = new HashSet<>();
        System.out.println(set.add("go") + " " + set.add("java") + " " + set.add("go"));   // add 返回是否新增
        System.out.println(set.size() + " " + set.contains("java"));
        set.remove("java");

        // 三种实现
        Set<Integer> linked = new LinkedHashSet<>(List.of(3, 1, 2));    // 插入顺序
        TreeSet<Integer> tree = new TreeSet<>(List.of(3, 1, 2, 10));    // 排序
        System.out.println(linked + " " + tree);
        System.out.println(tree.first() + " " + tree.last() + " " + tree.floor(5) + " " + tree.higher(3));

        // 集合运算 (都是原地修改, 所以先复制)
        Set<Integer> a = new TreeSet<>(List.of(1, 2, 3, 4));
        Set<Integer> b = new TreeSet<>(List.of(3, 4, 5));
        Set<Integer> union = new TreeSet<>(a);
        union.addAll(b);
        Set<Integer> inter = new TreeSet<>(a);
        inter.retainAll(b);
        Set<Integer> diff = new TreeSet<>(a);
        diff.removeAll(b);
        System.out.println(union + " " + inter + " " + diff);

        // 列表去重并保持顺序
        List<Integer> unique = new ArrayList<>(new LinkedHashSet<>(List.of(3, 1, 3, 2, 1)));
        System.out.println(unique);

        // 队列 (FIFO)
        Queue<String> queue = new ArrayDeque<>();
        queue.offer("a");
        queue.offer("b");
        queue.offer("c");
        System.out.println(queue.peek() + " " + queue.poll() + " " + queue);
        System.out.println(new ArrayDeque<String>().poll());          // 空队列 poll 返回 null
        try {
            new ArrayDeque<String>().remove();                        // remove 在空队列上抛异常
        } catch (NoSuchElementException e) {
            System.out.println("remove() on empty -> NoSuchElementException");
        }

        // 栈 (LIFO): 同样用 ArrayDeque, 不要用老的 Stack 类
        Deque<Integer> stack = new ArrayDeque<>();
        stack.push(1);
        stack.push(2);
        stack.push(3);
        System.out.println(stack.peek() + " " + stack.pop() + " " + stack);

        // 双端队列
        Deque<Integer> dq = new ArrayDeque<>();
        dq.offerFirst(1);
        dq.offerLast(2);
        dq.offerFirst(0);
        System.out.println(dq + " " + dq.pollFirst() + " " + dq.pollLast());

        // PriorityQueue: 默认最小堆, 只有 poll 保证顺序
        PriorityQueue<Integer> pq = new PriorityQueue<>();
        pq.addAll(List.of(5, 4, 3, 2, 1));
        System.out.println(pq);                                       // 堆内部顺序, 不是排序结果
        StringBuilder sb = new StringBuilder();
        while (!pq.isEmpty()) sb.append(pq.poll()).append(' ');
        System.out.println(sb.toString().trim());

        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());   // 最大堆
        maxHeap.addAll(List.of(5, 1, 4, 2));
        System.out.println(maxHeap.poll());

        // 按字段排序: 方法内可以定义局部 record
        record Task(String name, int prio) {}
        PriorityQueue<Task> tasks = new PriorityQueue<>(Comparator.comparingInt(Task::prio));
        tasks.add(new Task("deploy", 3));
        tasks.add(new Task("write", 1));
        tasks.add(new Task("test", 2));
        System.out.println(tasks.poll().name() + " " + tasks.poll().name() + " " + tasks.poll().name());
    }
}
```

**运行结果：**

```text
true true false
2 true
[3, 1, 2] [1, 2, 3, 10]
1 10 3 10
[1, 2, 3, 4, 5] [3, 4] [1, 2]
[3, 1, 2]
a a [b, c]
null
remove() on empty -> NoSuchElementException
3 3 [2, 1]
[0, 1, 2] 0 2
[1, 2, 4, 5, 3]
1 2 3 4 5
5
write test deploy
```


## 22. 不可变集合：List.of 与 Arrays.asList 的坑

**和 Go 的差异**

- Go 没有"不可变集合"的概念，通常靠约定或返回副本。Java 提供了专门的工厂方法。
- **`List.of` / `Set.of` / `Map.of`**（Java 9+）：真正不可变，任何修改都抛 `UnsupportedOperationException`。注意：
  - 不允许 `null` 元素，写了会 NPE；对 `List.of(...)` 调用 `contains(null)` 也会 NPE。
  - `Set.of` / `Map.of` 遇到重复元素 / 重复 key 抛 `IllegalArgumentException`。
  - **`Set.of` / `Map.of` 的遍历顺序每次运行都可能不同**，不要依赖，也不要直接打印后对比。
  - 键值对多的 Map 用 `Map.ofEntries(Map.entry(k, v), ...)`。
- **`Arrays.asList(array)`** 的行为很特殊：
  - 长度固定：`add` / `remove` 抛 `UnsupportedOperationException`。
  - 但允许 `set`，而且是**数组的视图**：改 list 会写穿到数组，改数组 list 也跟着变。
  - **传入基本类型数组（`int[]`）时，整个数组被当成一个元素**，得到的是长度为 1 的 `List<int[]>`。
- 视图 vs 副本：
  - `Collections.unmodifiableList(list)` 是**不可变视图**：不能通过它修改，但原列表变了它也变。
  - `List.copyOf(list)` 是**不可变副本**：之后原列表再变也不受影响。
- 需要一个可修改的初始列表时：`new ArrayList<>(List.of(...))`。
- **防御性复制**：类的构造器接收一个集合时，用 `List.copyOf` 复制一份，避免外部之后修改它。

```java
// ImmutableDemo.java
import java.util.*;

public class ImmutableDemo {
    public static void main(String[] args) {
        // List.of: 不可变
        List<Integer> fixed = List.of(1, 2, 3);
        try {
            fixed.add(4);
        } catch (UnsupportedOperationException e) {
            System.out.println("List.of -> add: UnsupportedOperationException");
        }
        try {
            List.of(1, null);
        } catch (NullPointerException e) {
            System.out.println("List.of -> null element: NPE");
        }
        try {
            Set.of(1, 1);
        } catch (IllegalArgumentException e) {
            System.out.println("Set.of -> " + e.getMessage());
        }

        // Map.of: 不可变; 遍历顺序不固定, 需要顺序时先放进 TreeMap
        Map<String, Integer> m = Map.of("a", 1, "b", 2);
        System.out.println(m.get("a") + " " + new TreeMap<>(m));

        // Arrays.asList: 长度固定, 是数组的视图
        String[] arr = {"a", "b", "c"};
        List<String> view = Arrays.asList(arr);
        view.set(0, "X");                          // 允许 set, 并且写穿到数组
        System.out.println(arr[0] + " " + view);
        arr[1] = "Y";                              // 改数组, list 也跟着变
        System.out.println(view);
        try {
            view.add("d");                         // 不能改变长度
        } catch (UnsupportedOperationException e) {
            System.out.println("Arrays.asList -> add: UnsupportedOperationException");
        }

        // 基本类型数组的坑
        int[] prim = {1, 2, 3};
        System.out.println(Arrays.asList(prim).size());     // 1: 整个 int[] 被当成一个元素
        Integer[] boxed = {1, 2, 3};
        System.out.println(Arrays.asList(boxed).size());    // 3

        // 不可变视图 vs 不可变副本
        List<Integer> source = new ArrayList<>(List.of(1, 2));
        List<Integer> unmodView = Collections.unmodifiableList(source);   // 视图: source 变了它也变
        List<Integer> snapshot = List.copyOf(source);                      // 副本: 不受 source 影响
        source.add(3);
        System.out.println(unmodView + " " + snapshot);

        // 防御性复制
        List<String> names = new ArrayList<>(List.of("a", "b"));
        Team team = new Team(names);
        names.add("c");                            // 外部之后再修改, 不影响 team
        System.out.println(team.members());
    }
}

class Team {
    private final List<String> members;

    Team(List<String> members) {
        this.members = List.copyOf(members);       // 防御性复制
    }

    List<String> members() {
        return members;                            // 本身不可变, 可以直接返回
    }
}
```

**运行结果：**

```text
List.of -> add: UnsupportedOperationException
List.of -> null element: NPE
Set.of -> duplicate element: 1
1 {a=1, b=2}
X [X, b, c]
[X, Y, c]
Arrays.asList -> add: UnsupportedOperationException
1
3
[1, 2, 3] [1, 2]
[a, b]
```


## 23. 遍历、Iterator 与遍历时修改集合

**和 Go 的差异**

- Go 里遍历 slice 时 `append` / 删除不会报错（但常常有逻辑 bug），遍历 map 时删除是安全的。Java 的集合是 **fail-fast** 的：`for-each` 遍历期间，如果通过集合自己的 `add` / `remove` 修改了结构，下一次迭代会抛 `ConcurrentModificationException`（CME）。
- **fail-fast 只是"尽力而为"**：删除倒数第二个元素时不会抛异常，但循环提前结束、漏掉最后一个元素。所以"没报错"不代表代码正确。
- 遍历时安全删除的几种方式：
  - `list.removeIf(条件)`：最简洁，首选。
  - `Iterator.remove()`：通过迭代器自己删。
  - 倒序下标循环。
  - 先收集要删的元素，遍历结束后再删。
- Map 的安全删除：`map.entrySet().removeIf(...)`、`map.keySet().removeIf(...)`、`map.values().removeIf(...)`。批量改 value 用 `replaceAll`。
- `for-each` 只是语法糖，对任何实现了 `Iterable<T>` 的类型都能用。自己的类实现 `Iterable` 就能被 `for-each` 遍历，作用类似 Go 的 range-over-func。
- `ListIterator` 支持双向遍历，以及遍历中 `set` / `add`。
- 多线程场景请用并发容器（第 32 章）。

```java
// IterationDemo.java
import java.util.*;

public class IterationDemo {
    public static void main(String[] args) {
        // 1. for-each 中直接 remove: ConcurrentModificationException
        List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4));
        try {
            for (int x : list) {
                if (x == 1) list.remove(Integer.valueOf(x));
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("CME, list is now " + list);
        }

        // 2. fail-fast 是尽力而为: 删除倒数第二个元素不抛异常, 但漏掉了最后一个元素
        List<Integer> tricky = new ArrayList<>(List.of(1, 2, 3));
        for (int x : tricky) {
            System.out.println("visit " + x);
            if (x == 2) tricky.remove(Integer.valueOf(x));
        }
        System.out.println(tricky);

        // 3. 正确做法 A: removeIf
        List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5, 6));
        nums.removeIf(n -> n % 2 == 0);
        System.out.println(nums);

        // 正确做法 B: Iterator.remove()
        Iterator<Integer> it = nums.iterator();
        while (it.hasNext()) {
            if (it.next() == 3) it.remove();
        }
        System.out.println(nums);

        // 正确做法 C: 倒序下标循环
        List<Integer> rev = new ArrayList<>(List.of(1, 2, 3, 4));
        for (int i = rev.size() - 1; i >= 0; i--) {
            if (rev.get(i) % 2 == 1) rev.remove(i);     // i 是 int, 调用的是 remove(int index)
        }
        System.out.println(rev);

        // Map: 遍历 keySet 时删除也会 CME
        Map<String, Integer> scores = new TreeMap<>(Map.of("a", 90, "b", 40, "c", 75));
        try {
            for (String k : scores.keySet()) {
                if (scores.get(k) < 60) scores.remove(k);
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("CME on map");
        }

        // Map 的正确做法
        scores = new TreeMap<>(Map.of("a", 90, "b", 40, "c", 75));
        scores.entrySet().removeIf(e -> e.getValue() < 60);
        System.out.println(scores);
        scores.replaceAll((k, v) -> v + 1);             // 批量修改 value
        System.out.println(scores);

        // 自定义类实现 Iterable, 就能被 for-each 遍历
        List<Integer> got = new ArrayList<>();
        for (int v : new Countdown(3)) got.add(v);
        System.out.println(got);
    }
}

class Countdown implements Iterable<Integer> {
    private final int from;

    Countdown(int from) { this.from = from; }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            private int cur = from;

            @Override
            public boolean hasNext() { return cur > 0; }

            @Override
            public Integer next() { return cur--; }
        };
    }
}
```

**运行结果：**

```text
CME, list is now [2, 3, 4]
visit 1
visit 2
[1, 3]
[1, 3, 5]
[1, 5]
[2, 4]
CME on map
{a=90, c=75}
{a=91, c=76}
[3, 2, 1]
```


---

## 第 3 部分（上）小结

1. 集合只存对象；变量写接口（`List` / `Map` / `Set`），右边 `new` 实现类；默认选 `ArrayList`、`HashMap`、`HashSet`、`ArrayDeque`。
2. `Map.get` 找不到返回 `null`；计数用 `merge`，分组用 `computeIfAbsent`。
3. `List<Integer>.remove(int)` 按下标，`remove(Object)` 按值，别搞混。
4. `HashMap` / `HashSet` / `Set.of` / `Map.of` 的遍历顺序不保证；`PriorityQueue` 只有 `poll` 保证顺序。
5. `List.of` 不可变且不允许 `null`；`Arrays.asList` 长度固定、是数组视图，传 `int[]` 会得到长度为 1 的列表。
6. 遍历时删除用 `removeIf` 或 `Iterator.remove()`；"没抛 CME"不等于代码正确。

下一个文件（第 3 部分下）：第 24 章 Stream API（`filter` / `map` / `collect` / `groupingBy`）和第 25 章 `Optional`。
