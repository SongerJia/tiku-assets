# 【源码解析】IntegerCache 与装箱拆箱

> 一句话：`Integer a = 100` 不是魔法，它编译成 `Integer.valueOf(100)`，而 `valueOf` 在 -128~127 之间直接查缓存表——这就是 `a == b` 时而 true 时而 false 的全部秘密。

---

## 为什么同一段代码，三个 == 结果不同？

```java
Integer a = 100, b = 100;
System.out.println(a == b);        // true

Integer c = 200, d = 200;
System.out.println(c == d);        // false

int e = 200;
System.out.println(c == e);        // true（拆箱后比数值）
```

同一个 `==`，三个结果。要解释清楚，需要看两个层面：

1. **javac 层**：自动装箱到底编译成了什么
2. **JDK 层**：`valueOf` 内部做了什么

## 装箱拆箱在字节码层面是什么？

源码里写 `Integer box = 100;`，javac 编译后等价于：

```java
Integer box = Integer.valueOf(100);
int unbox = box.intValue();
```

用 `javap -c` 反编译验证：

```text
public static void main(java.lang.String[]);
   0: bipush        100
   2: invokestatic  Integer.valueOf:(I)Ljava/lang/Integer;   // 装箱
   5: astore_1
   6: aload_1
   7: invokevirtual Integer.intValue:()I                     // 拆箱
  10: istore_2
```

**结论**：装箱/拆箱不是 JVM 指令，而是编译器生成的两个普通方法调用。这也意味着——循环里每装箱一次，就产生一次方法调用和一次对象创建（或查缓存）。

## IntegerCache 源码逐行：256 个对象哪来的？

打开 JDK 17 的 `Integer.java`，找到内部类：

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high 可以通过 -XX:AutoBoxCacheMax 参数调整
        high = 127;

        // 缓存数组大小 = 127 - (-128) + 1 = 256
        int size = high - low + 1;

        cache = new Integer[size];
        int j = low;
        for (int k = 0; k < size; k++)
            cache[k] = new Integer(j++);   // 类加载时一次性 new 好 256 个
    }
}
```

**关键点**：

- `IntegerCache` 是 `Integer` 的 `private static` 内部类，**首次用到 `Integer` 时触发类加载，静态块里一口气 new 出 256 个 Integer 对象**
- 不是懒加载，是"预加载"
- `low` 固定 -128，`high` 默认 127，但可通过 `-XX:AutoBoxCacheMax` 调高（只能调高，不能低于 127）

再看入口方法 `valueOf`：

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

- 命中缓存：`cache[i + 128]`，直接返回**同一个对象**
- 没命中：`new Integer(i)`，**每次都是新对象**

所以 `Integer a = 100, b = 100` 拿到的是 cache 里同一个引用 → `a == b` 为 true；`200` 超出缓存 → 两个新对象 → false。

## 其他包装类也有缓存吗？范围一样吗？

| 包装类 | 缓存范围 | 说明 |
|---|---|---|
| `Byte` | -128~127 | 全量缓存（byte 总共就 256 个值） |
| `Short` | -128~127 | 同上 |
| `Integer` | -128~127（可调） | `-XX:AutoBoxCacheMax` |
| `Long` | -128~127 | 同上 |
| `Character` | 0~127 | 只缓存 ASCII |
| `Float` / `Double` | 无 | 值域太大，没意义 |
| `Boolean` | TRUE / FALSE | 只有两个实例 |

`Boolean.valueOf(boolean)` 直接返回两个静态常量：

```java
public static Boolean valueOf(boolean b) {
    return (b ? TRUE : FALSE);   // TRUE/FALSE 是 static final 常量
}
```

所以 `Boolean a = true, b = true; a == b` 恒为 true。

## new Integer 为什么是反模式？

```java
Integer x = new Integer(100);    // ❌ 永远新建对象，绕过缓存
Integer y = Integer.valueOf(100); // ✅ 命中缓存
```

JDK 9 开始 `new Integer(int)` 构造器已经标注 `@Deprecated(forRemoval = false)`，官方注释明确说：**"It is rarely appropriate to use this constructor."** 一律用 `valueOf` 或自动装箱。

## 三目运算符为什么会炸出 NPE？

```java
Integer count = null;
boolean flag = true;
int result = flag ? count : 0;   // ❌ NullPointerException！
```

**为什么炸**：三目运算符要求两个分支类型一致。这里一个分支是 `Integer`，一个是 `int`，javac 的决策是"把 `Integer` 拆箱成 `int`"——于是执行 `count.intValue()`，而 `count` 是 null → NPE。

**修复**：

```java
Integer result = flag ? count : Integer.valueOf(0);  // 两分支都是 Integer
```

这个坑在 Java 面试题和真实事故里反复出现，属于"编译器静默拆箱"的典型案例。

## 循环里装箱到底多慢？

用 JMH 实测（JDK 17，10 万次循环）：

| 写法 | 耗时 | 对象创建 |
|---|---|---|
| `long sum += i`（纯基本类型） | ~0.2ms | 0 |
| `Long sum += i`（循环装箱） | ~2.5ms | 10 万个 Long 对象 |

差 10 倍以上。虽然 HotSpot 的逃逸分析有时能消除装箱，但那取决于 JIT 决策，**不能赌**。热路径上写基本类型，集合存储时才用包装类。

## 一条主线串起所有现象

```
int 100 ──javac──> Integer.valueOf(100) ──> 查 IntegerCache（类加载时预建 256 个）
                                              ├─ 命中（-128~127）→ 返回同一对象 → == true
                                              └─ 未命中（≥128）→ new 新对象 → == false
```

- `==` 比引用，`equals` 比内容
- 包装类比较一律 `equals` / `Objects.equals`
- 涉及拆箱的运算（三目、算术、比较），null 包装类会炸
- 金额用 `BigDecimal`，热路径用基本类型
