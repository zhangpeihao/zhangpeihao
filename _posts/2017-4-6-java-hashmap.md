# Java标准库的HashMap实现

熟悉Java泛型和数据结构，最好的办法就是深入研读Java标准库源码。

而HashMap是一个比较有代表性的数据结构。HashMap通过链表实现简单搜索，通过红黑树实现复杂搜索，数据存储使用数组。

了解HashMap的实现，可以帮助我们更好地使用HashMap，更可以为我们自己设计数据结构提供指导。

## 官方说明

### 原文

Hash table based implementation of the `Map` interface. This implementation provides all of the optional map operations, and permits `null` values and the `null` key. (The `HashMap` class is roughly equivalent to Hashtable, except that it is unsynchronized and permits nulls.) This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time.

This implementation provides constant-time performance for the basic operations (`get` and `put`), assuming the hash function disperses the elements properly among the buckets. Iteration over collection views requires time proportional to the "capacity" of the `HashMap` instance (the number of buckets) plus its size (the number of key-value mappings). Thus, it's very important not to set the initial capacity too high (or the load factor too low) if iteration performance is important.

An instance of `HashMap` has two parameters that affect its performance: *initial capacity* and *load factor*. The *capacity* is the number of buckets in the hash table, and the initial capacity is simply the capacity at the time the hash table is created. The *load factor* is a measure of how full the hash table is allowed to get before its capacity is automatically increased. When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is rehashed (that is, internal data structures are rebuilt) so that the hash table has approximately twice the number of buckets.

As a general rule, the default load factor (.75) offers a good tradeoff between time and space costs. Higher values decrease the space overhead but increase the lookup cost (reflected in most of the operations of the `HashMap` class, including get and put). The expected number of entries in the map and its load factor should be taken into account when setting its initial capacity, so as to minimize the number of rehash operations. If the initial capacity is greater than the maximum number of entries divided by the load factor, no rehash operations will ever occur.

If many mappings are to be stored in a `HashMap` instance, creating it with a sufficiently large capacity will allow the mappings to be stored more efficiently than letting it perform automatic rehashing as needed to grow the table.

**Note that this implementation is not synchronized**. If multiple threads access a hash map concurrently, and at least one of the threads modifies the map structurally, it *must* be synchronized externally. (A structural modification is any operation that adds or deletes one or more mappings; merely changing the value associated with a key that an instance already contains is not a structural modification.) This is typically accomplished by synchronizing on some object that naturally encapsulates the map. If no such object exists, the map should be "wrapped" using the `Collections.synchronizedMap` method. This is best done at creation time, to prevent accidental unsynchronized access to the map:

```
  Map m = Collections.synchronizedMap(new HashMap(...));
```

The iterators returned by all of this class's "collection view methods" are *fail-fast*: if the map is structurally modified at any time after the iterator is created, in any way except through the iterator's own `remove` method, the iterator will throw a ConcurrentModificationException. Thus, in the face of concurrent modification, the iterator fails quickly and cleanly, rather than risking arbitrary, non-deterministic behavior at an undetermined time in the future.

Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast iterators throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: *the fail-fast behavior of iterators should be used only to detect bugs*.

### 中文翻译

Hash表格提供了`Map`接口的实现。这个实现提供了所有可选的map操作，并且允许`null`值和`null`键。(`HashMap`类基本相当于`Hashtable`，除了非同步化和允许null。）这个类不保证map的顺序；而且，不保证在一定时间内顺序不变。

这个实现为基础操作（`get`和`put`）提供了常量时间的性能，假设hash函数把元素合理地分散到桶中。在集合视图（Collection Views）上进行迭代所需要的时间是与`HashMap`的“容量”加上其大小（键值映射数量）成正比的。因此，如果迭代性能
很重要的话，很重要的一点是不要将初始容量设置得太高（或者负载因子太低）。

一个`HashMap`实例有两个参数影响其性能： *初始容量* 和 *负载因子* 。 *容量* 是指hash table中的桶数，初始容量是在创建hash table时简单赋予的容量。 *负载因子* 是一个衡量hash table允许加到多满就自动扩充容量的系数。当hash table中的条目数超过了负载因子和当前容量的乘积，hash table会 *重新hash* （也就是，内部数据结构被重新构建）所以hash table具有大概两倍数量的桶。

作为一般规则，默认负载因子（.75）提供了时间和空间成本之间的良好折中。较高的值会降低空间开销，但会增加查找成本（在`HashMap`的大多数操作中都有反应，包括`get`和`put`）。在设置初始容量时，应当考虑Map中预期的条目数量和其负载因子，以便最小化重新hash（rehash）的数量。如果初始容量大于最大条目数除以负载因子，就不会发生重新hash（rehash）操作。

如果很多映射要被保存到一个`HashMap`实例中，用一个充足的大容量来创建实例将让这些映射被存储得更高效，相比之下根据需要自动重新hash（rehash）来增加table不太高效。

**注意：这个实现不提供线程同步** 。如果多个线程同时访问hasp map，并且至少有一个线程在结构上修改了映射，那么它 *必须* 在外部进行同步。（结构修改是指添加或删除一个或者多个映射的任何操作；仅仅改变一个实例已经包含的Key对应的值不属于结构修改。）通常用对一些原生封装了这个map的对象进行同步来实现。如果不存在这种对象，这个map需要用`Collections.synchronizedMap`方法来进行包装。这最好在创建时完成，以防止不小心用非同步方式访问map：

```
  Map m = Collections.synchronizedMap(new HashMap(...));
```

这个类的所有“集合视图方法”返回的迭代器都是 *fail-fast* ：如果映射在迭代器创建之后的任何时间被结构化修改，除了迭代器自己的`remove`方法之外的任何情况，迭代器将抛出一个`ConcurrentModificationException`。因此，面对并发修改，迭代器失败是快速地、干净地，而不是在未来某个不确定的时间，不确定方式出现的任意风险。

注意，迭代器的fail-fast行为方式并不能作为保证，一般来说，在不同步并发修改的情况下，无法做出任何硬性保证。fail-fast迭代器尽力抛出`ConcurrentModificationException`。因此，编写代码时，依赖这个异常来保证正确性是不正确的： *迭代器的fail-fast行为只能用来检测错误* 。

### 理解

官方说明主要提到了以下几点：

* HashMap的两个重要参数： *初始容量* 和 *负载因子* 
* 多线程下的安全
* fail-fast的说明

## 代码分析

### 继承与实现

`HashMap`类继承了`AbstractMap`类，实现了`Map`接口

```java
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

### Holder

HashMap有两种hash算法，两种算法切换的阀值是通过内嵌静态Holder类从jdk参数中读取。

```
    private static class Holder {

        /**
         * Table capacity above which to switch to use alternative hashing.
         */
        static final int ALTERNATIVE_HASHING_THRESHOLD;

        static {
            String altThreshold = java.security.AccessController.doPrivileged(
                new sun.security.action.GetPropertyAction(
                    "jdk.map.althashing.threshold"));

            int threshold;
            try {
                threshold = (null != altThreshold)
                        ? Integer.parseInt(altThreshold)
                        : ALTERNATIVE_HASHING_THRESHOLD_DEFAULT;

                // disable alternative hashing if -1
                if (threshold == -1) {
                    threshold = Integer.MAX_VALUE;
                }

                if (threshold < 0) {
                    throw new IllegalArgumentException("value must be positive integer.");
                }
            } catch(IllegalArgumentException failed) {
                throw new Error("Illegal value for 'jdk.map.althashing.threshold'", failed);
            }

            ALTERNATIVE_HASHING_THRESHOLD = threshold;
        }
    }
```

### 构造与初始化

HashMap的构造主要设置 *初始容量* 和 *负载因子*。

并切在构造函数里预留了一个init函数调用，为子类提供初始化扩展。

另外，HashMap还提供了一个从其它Map复制的构造函数，这个构造函数使用源Map数量作为参考来计算初始容量。所以，必须注意，如果新的Map会进行增加操作，最好不要用这个构造函数，这样有可能会导致HashMap多做一次重Hash（rehash）。可以按照新Map的预计容量构造一个HashMap然后再进行插入。

### 存储

HashMap使用数组保存数据，数组的长度必须是2的幂次。

```
    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
```

### Hash算法

```
    /**
     * Retrieve object hash code and applies a supplemental hash function to the
     * result hash, which defends against poor quality hash functions.  This is
     * critical because HashMap uses power-of-two length hash tables, that
     * otherwise encounter collisions for hashCodes that do not differ
     * in lower bits. Note: Null keys always map to hash 0, thus index 0.
     */
    final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
```

如果打开了althashing hashing，字符串类型的Key使用`sun.misc.Hashing.stringHash32`进行hash，能够提供更好的hash分布。

其它类型使用`Object.hashCode`进行hash。

### put方法

```
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```

首先，通过`key`取得hash值，然后将hash值映射到table上。在对应位置上遍历查找，如果找到了同样key，更新value，如果找不到，则通过`addEntry`加新的value。

### get方法

```
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
```

当hash到Entry之后，遍历所有项

### resize

当capacity大于table的大小时，进行resize。

```
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

resize先构建新table数组空间，然后调用`transfer`函数将原table中的数据取出，重新计算hash，插入到新table中。

### Entry

HaspMap使用单向链表保存相同hash值的数据。

Entry除了保存key和value之外，还保存了hash值和next指针。



