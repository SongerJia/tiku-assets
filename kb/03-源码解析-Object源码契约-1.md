# 【源码解析】Object 源码契约

## equals / hashCode / toString：所有 Java 对象的"基本人权"

<span style="color:#888">Java 语言核心 · 第 03 讲延伸 · 源码专题</span>

---

## Object 到底定义了哪些方法？

Object 是所有类的祖先（隐式继承），它定义了 Java 对象的四个核心契约：

- **equals**：对象如何比较
- **hashCode**：对象如何散列（配合 HashSet/HashMap）
- **toString**：对象如何呈现
- **clone**：对象如何复制

> 这四个方法 99% 的类都要重写（或交给 Lombok/IDE），但理解"契约"比会写更重要——违约的代价是集合失效、缓存错乱、线上事故。

---

## Object 类源码全貌：native 方法是什么？

```java
public class Object {

    private static native void registerNatives();

    // 反射：获取类对象（final：不允许重写）
    public final native Class<?> getClass();

    // 哈希：identity hash（默认按对象地址生成的散列）
    public native int hashCode();

    // 相等：默认就是引用比较（==）
    public boolean equals(Object obj) {
        return (this == obj);
    }

    // 克隆：浅拷贝（需实现 Cloneable 接口）
    protected native Object clone() throws CloneNotSupportedException;

    // 字符串表示：类名@十六进制hashCode
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }

    // 线程协作
    public final native void notify();
    public final native void notifyAll();
    public final native void wait(long timeout) throws InterruptedException;

    // 终结器（JDK 9 起废弃，JDK 18 移除）
    @Deprecated(since = "9")
    protected void finalize() throws Throwable { }
}
```

**关键点**：

- 默认 `equals` = `==`（比较引用地址）
- 默认 `hashCode` 是 identity hash（与地址相关，但不可预测）
- 默认 `toString` = `类名@十六进制hash`（对排查基本没用）
- `getClass()` 是 final native

---

## equals 的 5 条铁律是什么？

```java
// JDK 文档要求的 5 条：
// 1. 自反性：x.equals(x) == true
// 2. 对称性：x.equals(y) == y.equals(x)
// 3. 传递性：x=y && y=z → x=z
// 4. 一致性：对象未修改，结果不变
// 5. 约定：equals 相等 → hashCode 必须相等
```

**最容易违反的坑**：

```java
// 反例：子类 equals 用 getClass()，父类用 instanceof
class Base {
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Base)) return false;   // instanceof 宽松
        return true;
    }
}
class Sub extends Base {
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Sub)) return false;    // 更严格
        return true;
    }
}
// 对称性被破坏：
// new Base().equals(new Sub())  → true
// new Sub().equals(new Base())  → false  ← 违约！
```

**推荐实现模板**（Effective Java 第 10 条）：

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;                        // 1. 引用相同
    if (o == null || getClass() != o.getClass())       // 2. 类型检查
        return false;
    User user = (User) o;                              // 3. 强转
    return age == user.age                             // 4. 逐字段比
        && Objects.equals(name, user.name);            //    （null 安全）
}
```

---

## hashCode 为什么必须和 equals 绑定？

```java
// 契约：
// 1. equals 相等 → hashCode 必须相等（强制！）
// 2. hashCode 相等 → equals 不一定相等（允许碰撞）
// 3. 对象未修改，多次 hashCode 结果相同
```

**为什么必须配套？**

```
HashSet 查找流程：
1. 算 hashCode → 定位桶
2. 桶内用 equals 逐个比较

如果 equals 相等但 hashCode 不同：
→ 进不同桶 → equals 永远碰不到 → Set 里"重复"元素
→ 去重失效 / 缓存越积越多（真实事故！）
```

**推荐散列实现**：

```java
@Override
public int hashCode() {
    int result = Objects.hashCode(name);
    result = 31 * result + Integer.hashCode(age);
    return result;
}
// 或更简洁：
// return Objects.hash(name, age);
```

**为什么用 31？**

- 31 是奇质数，`31 * h + c` 的散列分布好
- `31 * h` 可优化为 `(h << 5) - h`（移位减），JIT 能识别

---

## toString 为什么最值得重写？

```java
// 默认输出：com.demo.User@1a2b3c4d（没用）
@Override
public String toString() {
    return "User{name='" + name + "', age=" + age + '}';
}
```

**价值**：

- 日志：`log.error("user={}", user)` 直接看清对象内容
- 调试：断点里一目了然
- 错误信息友好

**注意**：

- 别打敏感字段（密码/手机号）——日志泄漏
- 循环引用会栈溢出（A→B→A）：深度限制或跳过
- 大集合别整 dump

---

## clone 为什么默认是浅拷贝？

```java
// 默认 clone 是浅拷贝（字段引用共享）
@Override
public Object clone() throws CloneNotSupportedException {
    return super.clone();    // 需实现 Cloneable，否则抛异常
}
```

```java
// 浅拷贝后果：
User u2 = u1.clone();
u2.getTags().add("x");       // ⚠️ u1 的 tags 也被改了！

// 深拷贝方案：
// 1. 手动：return new User(name, new ArrayList<>(tags));
// 2. 序列化 JSON 绕一圈（通用但重）
// 3. 注意：BeanUtils.copyProperties 也是浅拷贝！
```

**建议**：Java 的 clone 设计有缺陷（Cloneable 是标记接口 + Object 默认浅拷贝），**优先用拷贝构造器 / 工厂 / 手动深拷贝**。

---

## 一条主线串起 Object 契约

- 默认 `equals` = `==`，默认 `hashCode` = identity hash
- equals 5 条契约：自反 / 对称 / 传递 / 一致 / 与 hashCode 绑定
- **equals 与 hashCode 必须成对重写**——违反的直接后果是 HashSet/HashMap 失效
- toString 重写是"低成本高回报"
- clone 是浅拷贝，优先手动深拷贝

> Object 的契约是 Java 集合框架的地基：读懂了它，HashMap 的 key 为什么必须是"不可变 + equals/hashCode 正确"就全通了。
