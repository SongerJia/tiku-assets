# 【源码解析】ArrayList 扩容与迭代器

## 动态数组的实现细节：grow、System.arraycopy 与 modCount

<span style="color:#888">Java 语言核心 · 第 11 讲延伸 · 源码专题</span>

---

## 一、概述：最常用的集合，最该读的源码

ArrayList 是 Java 使用频率最高的集合类。理解它的**扩容机制**和**迭代器实现**，是写对代码、读得懂报错的前提：

- `grow()` 扩容策略：为什么是 1.5 倍而不是 2 倍？
- `Itr` 迭代器：`modCount` 怎么实现 fail-fast？
- `subList`：为什么是视图而不是副本？
- `toArray()`：为什么不能强转成 `String[]`？

---

## 二、数据结构：Object[] + size

```java
public class ArrayList<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, Serializable {

    // 默认容量
    private static final int DEFAULT_CAPACITY = 10;
    // 空数组（new ArrayList() 时用）
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    // 空数组（new ArrayList(0) 时用）
    private static final Object[] EMPTY_ELEMENTDATA = {};

    // 存储数组（transient：默认序列化不走数组，按 size 个元素写）
    transient Object[] elementData;
    // 逻辑元素个数
    private int size;
}
```

**关键点**：

- `size` 是逻辑个数，`elementData.length` 是物理容量，两者不等
- `transient Object[]`：序列化时只写 `size` 个元素，不写整个数组（省空间）
- `RandomAccess`：标记接口，告诉工具类"支持随机访问，用 index 遍历更快"

---

## 三、add() 流程与扩容

```java
public boolean add(E e) {
    modCount++;                          // 结构性修改计数 +1
    add(e, elementData, size);
    return true;
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)         // 容量满了
        elementData = grow();            // 扩容
    elementData[s] = e;                  // 直接写入
    size = s + 1;
}

private Object[] grow() {
    return grow(size + 1);               // 至少装下 size+1
}

private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);   // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;                        // 不够就取最小
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);          // 上限保护
    return elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**为什么 1.5 倍而不是 2 倍？**

- 倍数越大，扩容次数越少，但浪费的空间越多
- 1.5 倍是"扩容次数"与"空间浪费"的平衡点
- 扩容是 `Arrays.copyOf`（`System.arraycopy` 拷贝），均摊下来每次 add 是 **O(1)**

---

## 四、remove() 与 System.arraycopy

```java
public E remove(int index) {
    Objects.checkIndex(index, size);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;     // 后面要移动的元素数
    if (numMoved > 0)
        System.arraycopy(elementData, index + 1,
                         elementData, index, numMoved);   // 整体前移
    elementData[--size] = null;          // 置空，帮助 GC
    return oldValue;
}
```

**关键点**：

- 中间删除 = `System.arraycopy` 整体前移，**O(n)**
- 末尾元素置 null：防止对象滞留（内存泄漏隐患）
- 这也是为什么"中间频繁增删"用 LinkedList 更合适

---

## 五、subList：视图而非副本（高频坑）

```java
public List<E> subList(int fromIndex, int toIndex) {
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList<>(this, fromIndex, toIndex);   // 视图！
}
```

**视图语义**（SubList 内部持有父列表引用）：

- `sub.set(i, x)` 会改父列表
- 父列表结构性修改后，子列表所有操作抛 `ConcurrentModificationException`
- 想要独立副本：`new ArrayList<>(list.subList(a, b))`

---

## 六、迭代器与 fail-fast

```java
public Iterator<E> iterator() {
    return new Itr();
}

private class Itr implements Iterator<E> {
    int cursor;                          // 下一个元素下标
    int expectedModCount = modCount;     // 创建时的快照

    public E next() {
        checkForComodification();        // 每次 next 检查
        ...
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }

    public void remove() {
        ...
        expectedModCount = modCount;     // 迭代器自己删除会同步快照
        // → 所以迭代器 remove 不抛 CME
    }
}
```

**fail-fast 机制**：

- 迭代器创建时记录 `modCount` 快照
- 每次 `next()` 检查 `modCount` 是否变化
- 其他线程/for-each 里 `list.remove()` 会改 `modCount` → 抛 `ConcurrentModificationException`
- 迭代器自身 `remove()` 会同步 `expectedModCount` → 合法

> fail-fast 不是线程安全机制，而是"尽快暴露并发修改"的检测手段。真正并发用 `CopyOnWriteArrayList`（快照迭代器，弱一致）。

---

## 七、toArray() 的坑

```java
// 无参版返回 Object[]：
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

// 带类型版：
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;                  // 预分配比 size 大，尾部置 null
    return a;
}
```

**坑**：

- `(String[]) list.toArray()` → `ClassCastException`（返回的是 `Object[]`）
- 正确：`list.toArray(new String[0])`（JDK 11+ 推荐，反射优化比 `new String[size]` 快）
- 预分配数组比 size 大 → 尾部 `null`

---

## 小结

- **结构**：`Object[] + size`，容量与个数分离
- **扩容**：1.5 倍 + `Arrays.copyOf`，均摊 O(1)
- **删除**：`System.arraycopy` 前移，O(n)
- **subList**：视图，不是副本
- **迭代器**：`modCount` 快照实现 fail-fast
- **toArray**：传类型数组，别强转

> ArrayList 的源码是整个集合框架的入门钥匙：读懂了它，LinkedList/HashMap 的源码就是同一套思维。
