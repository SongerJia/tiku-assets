# 【源码解析】StringBuilder 与拼接优化

## 从字节码到 JEP 280，拆解 Java 拼接为什么快

<span style="color:#888">Java 语言核心 · 第 06 讲延伸 · 源码专题（二）</span>

---

## 拼接在编译期和运行期分别怎么发生？

`String` 不可变，所以 `"a" + "b"` 不能原地改——编译器必须把它变成"造新对象"的代码。但**编译期能算的**和**运行期才算的**走的是两条完全不同的路：

- **编译期常量折叠**：`"a" + "b" + "c"`（全是字面量/`final`）→ 编译器直接算成 `"abc"`，字节码只有一条 `ldc`（见主题文档块 13）。
- **运行期变量拼接**：`a + b + c`（含变量）→ 编译器合成 `new StringBuilder().append(a).append(b).append(c).toString()`（见主题文档块 14 字节码）。

为什么变量拼接要绕一圈 StringBuilder？因为 `String` 不可变，无法就地追加，只能"新造一个 StringBuilder 当临时缓冲区，append 完再 toString 成新 String"。

> 关键点：**每处运行期 `+` 都暗搓搓 new 了一个 StringBuilder**。单次无感，循环里就是性能灾难（主题文档块 20）。

---

## StringBuilder 扩容源码逐行

`StringBuilder` 继承自 `AbstractStringBuilder`，底层是 `char[] value`（JDK 8）。容量管理是拼接性能的核心：

```java
// AbstractStringBuilder（JDK 8）
char[] value;
int count;          // 已用字符数

public AbstractStringBuilder append(String str) {
    if (str == null) str = "null";
    int len = str.length();
    ensureCapacityInternal(count + len);   // ① 先确保容量够
    str.getChars(0, len, value, count);     // ② 拷贝进 value
    count += len;
    return this;
}

private void ensureCapacityInternal(int minimumCapacity) {
    if (minimumCapacity - value.length > 0)    // 不够就扩容
        expandCapacity(minimumCapacity);
}

void expandCapacity(int minimumCapacity) {
    int newCapacity = value.length * 2 + 2;     // ③ 翻倍 + 2
    if (newCapacity < minimumCapacity)          // ④ 还不够就直接取所需
        newCapacity = minimumCapacity;
    value = Arrays.copyOf(value, newCapacity);  // ⑤ 复制旧数组到新数组
}
```

**扩容公式**：`newCapacity = oldCap * 2 + 2`，若仍不够则直接取 `minimumCapacity`。实测轨迹（探针，默认 16）：`16 → 34 → 70`（16×2+2=34，34×2+2=70）。

> 边界：`Arrays.copyOf` 是 O(n) 复制——**扩容越多次，浪费越大**。拼大文本前 `new StringBuilder(预估大小)` 一次到位，能省下所有中间复制（主题文档块 17）。

---

## StringBuilder.toString() 的真相

`append` 一堆后，`toString()` 怎么变回 `String`？

```java
// AbstractStringBuilder.toString()
public String toString() {
    return new String(value, 0, count);   // 复制 [0, count) 区间成新 String
}
```

注意：**不是直接把内部 `value` 暴露出去**（那会破坏不可变），而是 `new String(value, 0, count)` 复制出一段独立数组。所以 `toString` 后改 StringBuilder 不影响已生成的 String，反之亦然。

> 边界：这也解释了"拼接产生中间 String"——每个 `toString` 都是一次数组复制。循环里反复 `+=` 就是反复 `new StringBuilder + toString`，中间 String 一堆，GC 压力陡增。

---

## StringBuffer 为什么线程安全

`StringBuilder` 和 `StringBuffer` 几乎共用同一套 `AbstractStringBuilder` 实现，唯一区别：`StringBuffer` 的每个方法加了 `synchronized`。

```java
// StringBuffer（JDK 8）
public final class StringBuffer extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence {
    private transient char[] toStringCache;   // 缓存上次 toString 结果

    @Override
    public synchronized StringBuffer append(String str) {
        toStringCache = null;            // 任何修改都清缓存
        super.append(str);
        return this;
    }
    @Override
    public synchronized String toString() {
        if (toStringCache == null)
            toStringCache = Arrays.copyOfRange(value, 0, count);
        return new String(toStringCache, 0, count);  // 复用缓存
    }
}
```

`toStringCache` 是 `StringBuffer` 独有优化：只要没被修改，多次 `toString` 复用同一份拷贝。但 `append` 一改就清空——保证缓存一致性靠的就是 `synchronized` 互斥。

> 边界：锁在 `this` 上，**每次 append 都争同一把锁**。单线程下这把锁纯开销；多线程共享同一实例才体现价值，但现实中这种场景极少（主题文档块 19）。

---

## JDK 9+ 的拼接优化：invokedynamic + StringConcatFactory

JDK 8 的拼接是"编译器硬编码 `new StringBuilder`"，不够灵活。JDK 9 的 **JEP 280** 改成：拼接编译成 `invokedynamic`，运行时由 `StringConcatFactory` 动态决定拼接策略。

```java
// JDK 9+ 编译 a + b + c 近似形态
String s = invokedynamic StringConcatFactory.makeConcat("{}", a, b, c);
//                       ↑ 第一次调用走引导方法，之后直接执行优化后的拼接
```

`StringConcatFactory` 会按参数个数选择策略：
- 少参数：直接 `new String` + 手工拼接（比 StringBuilder 还快）
- 多参数：仍用 `StringBuilder`
- 策略可通过 `-Djava.lang.invoke.stringConcat` 调（如 `BC_SB` 强制 StringBuilder）

收益：
- **少一次 StringBuilder 对象分配**（热路径更轻）
- 拼接策略可随 JVM 版本演进，无需重编译

> 边界：这是 JDK 9 的优化，对本篇 JDK 8 实测的"每处 + 都 new StringBuilder"结论在 JDK 9+ 已优化，但**循环陷阱的本质没变**——变量拼接仍要造新 String，循环里手动 `StringBuilder` 依然最快。

---

## append 各类型的实现差异

`append` 重载了几乎所有类型，性能差异值得记：

| 类型 | 实现要点 | 成本 |
|---|---|---|
| `String` | `getChars` 直接拷贝 char 数组 | 低（纯拷贝） |
| `char`/`int`/`long` | 转成字符再拷贝 | 低 |
| `Object` | 先 `String.valueOf(obj)` 再当 String 拼 | 中（多一次 toString） |
| `boolean` | 拷贝 "true"/"false" | 低 |
| `char[]` | `System.arraycopy` | 低 |

最贵的两种：
1. **`append(Object)`** 会先调其 `toString()`——若 toString 本身很重（如大集合），拼接成本被 toString 主导。
2. **循环里反复 `append` 触发扩容**（见块二）——O(n) 复制累积。

> 边界：拼对象时优先拼它已经算好的字符串字段，别把整个对象丢进去让 append 去调 toString，可控性更好。

---

## 拼接性能量级（实测与文献）

量级参考（JDK 8，10 万次拼接，秩序级非精测）：
- `String +=` 循环：**数百 ms**，产生约 10 万个中间对象，YGC 频繁。
- `StringBuilder` 循环：**数 ms**，零中间对象。
- 差距 **数十倍到上百倍**。
- 预分配容量的 `StringBuilder` 比默认再快一截（少 2~3 次扩容复制）。

`String.join` / `StringJoiner` / 流式 `Collectors.joining` 底层都是 `StringBuilder`，性能和手写 `StringBuilder` 同量级，且代码更干净——**日常拼接优先它们**（主题文档块 21）。

> 边界：这些是"量级"不是"精确数字"——具体取决于串长、循环次数、JVM。但"循环里 + 远慢于 StringBuilder"这个结论是稳固的，足够指导选型。

---

## 一条主线串起拼接优化

- 拼接的本质：**不可变 String 无法直接追加 → 必须造临时缓冲区（StringBuilder）→ toString 复制成新 String**。
- JDK 8：`+` 编译成 `new StringBuilder` 链；JDK 9+：改成 `invokedynamic` + `StringConcatFactory` 动态策略（更快但仍造新 String）。
- 性能命脉在**扩容复制**和**中间对象**：预分配容量 + 循环外用 StringBuilder，是两条铁律。
- `StringBuffer` 只是 `synchronized` 版 StringBuilder，单线程纯浪费，多线程共享才用。
- 选型口诀：**"不变用 String，要拼用 StringBuilder，多线程共享才轮到 StringBuffer"**（主题文档块 29/30）。
