# HashMap 1.7源码解析

## 1.介绍

### 1.描述

​        **HashMap**在我们平常开发中使用非常广泛,本文就从**JDK1.7** 分析**HashMap**相关源码（后续再加上1.8）. 在**JDK1.7**中**HashMap**底层是由**数组**+**链表**实现的,每次在插入数据的时候,会根据**key**来计算对应的**Hash**.使用各种位操作将**Hash**值转换成对应的数组下标,根据下标来找到数组(**Entry**)对应位置.如果当前位置对应的**Entry**对象不为空,则以**头插**的方式将数据插入到链表中.如果为空的话,直接将数据插入到**Entry**数组中.

### 2.线程安全问题

​        **HashMap**是**非线程安全**的.在**多线程**的情况下,**HashMap**在**扩容**的时候,会生成一个新的数组,将之前数组上的数据转移到新的数组上.在这个过程中可能会造成**循环链表**.所以在使用**HashMap**的时候最好是**指定容量**.一是为了防止多线程带了的问题,二是为了减少扩容带来不必要的损耗

### 3.图解

 

## ![HashMap1.7](.\assets\HashMap1.7.png)

## 2.源码解析

### 1.创建HashMap，初始化阈值和负载因子

```java
public HashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}

 public HashMap(int initialCapacity, float loadFactor) {
     //判断初始化容量
     if (initialCapacity < 0)
         throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
     //判断阈值
     if (initialCapacity > MAXIMUM_CAPACITY)
         initialCapacity = MAXIMUM_CAPACITY;
     if (loadFactor <= 0 || Float.isNaN(loadFactor))
         throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);

     this.loadFactor = loadFactor;
     threshold = initialCapacity;
     init();
}
```

> HashMap在初始化的时候，如果传入了相应**容量**和**负载因子**就是用传入的,否则使用默认的. **DEFAULT_INITIAL_CAPACITY**  默认值 **16**  **DEFAULT_LOAD_FACTOR** 默认 **0.75f**,**init()** 在**LinkedHashMap**中有具体实现

### 2.put

```java
public V put(K key, V value) {
    //判断数组是否是空
    if (table == EMPTY_TABLE) {
        //如果为空 初始化数组
        inflateTable(threshold);
    }
    //对key为null的处理
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    //根据hash值获取数组的下标
    int i = indexFor(hash, table.length);
    //根据下标获取对应的Entry对象,遍历链表,判断是否有重复的key,如果有将value替换,返回之前对应的value值
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        //判断key是否存在...
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    //如果key不重复
    
    //记录修改的次数(快速失败(fail—fast)会使用到)
    modCount++;
    //存入到Entry中,下面分解
    addEntry(hash, key, value, i);
    return null;
}
```

#### 1.inflateTable初始化Entry数组

```java
private void inflateTable(int toSize) {
    // Find a power of 2 >= toSize 确保数组的容量 >=2的n次幂
    int capacity = roundUpToPowerOf2(toSize);

    //阈值 = 容量*加载因子
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    //该方法可以在启动时传入参数,修改hashSeed,干涉hash值的生成
    initHashSeedAsNeeded(capacity);
}
```

##### 1.roundUpToPowerOf2  数组的容量 >=2的n次幂

方法主要是,让数组的容量 >=2的n次幂.比如:当我传入toSize= 1, capacity=2;toSize= 3, capacity=4;toSize= 16, capacity=16.就是这样的对应关系.

##### 2.initHashSeedAsNeeded hashSeed(hash种子)

```java
final boolean initHashSeedAsNeeded(int capacity) {
    //判断hashSeed 是否等于0(默认为0)
    boolean currentAltHashing = hashSeed != 0;
    //检查VM是否启动(true) 并且 容量 >= Holder.ALTERNATIVE_HASHING_THRESHOLD
    boolean useAltHashing = sun.misc.VM.isBooted() &&
        (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    //将currentAltHashing   useAltHashing 做异或操作
    boolean switching = currentAltHashing ^ useAltHashing;
    //如果switching == true 
    if (switching) {
        hashSeed = useAltHashing
            ? sun.misc.Hashing.randomHashSeed(this)
            : 0;
    }
    return switching;
}


```

综上分析 switching 值由 hashSeed 和 Holder.ALTERNATIVE_HASHING_THRESHOLD 值决定

###### 1.hashSeed 作用

```java
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

 **hashSeed** 在HashMap出现了三个次(地方):一次**初始化** 一次i**nitHashSeedAsNeeded方法**中 一次**hash方法**中   **hashSeed** 决定**hash值**的生成       

###### 2.Holder.ALTERNATIVE_HASHING_THRESHOLD初始化位置

```java
// Holder.ALTERNATIVE_HASHING_THRESHOLD  初始化
static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;
private static class Holder {
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
//

```

​        从代码中可以看出,判断在**启动参数**上是否加了**jdk.map.althashing.threshold=XX** 的信息,如果加了(不为空)就会将其转换为int赋给**threshold**,最后将**threshold** 有赋值给 **ALTERNATIVE_HASHING_THRESHOLD**.如果没有加入该启动参数(或者加了启动参数为复数),**threshold** 就会等于**ALTERNATIVE_HASHING_THRESHOLD_DEFAULT** 也就是( **Integer.MAX_VALUE** (2147483647)).

​        显然我在启动的时候并没有加上 **-D jdk.map.althashing.threshold**  ,所以Holder.ALTERNATIVE_HASHING_THRESHOLD=2147483647

![initHashSeedAsNeeded](.\assets\initHashSeedAsNeeded-1603197201409.png)

**initHashSeedAsNeeded作用:默认情况下，当容量>=Integer.MAX_VALUE或者在启动参数(-D jdk.map.althashing.threshold）传入一个非零的值，switcing才会为true,hashSeed才会改变。假如说你觉得HashMap中hash算法分布不够散列.那么你可以自己传入参数，干扰hash值的生成。**

#### 2.putForNullKey key为空的情况

```java
private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```

在**putForNullKey**中，可以看出**HashMap**会把**key==null**的数据放入**数组为0**的位置。获取数组的第一个位置，然后遍历数组对应的链表，如果**存在**的话**key==null**的情况，就将**现有的value**替换**原有的value**，并且返回**原有的value**。如果不存。就执行**addEntry**,以**头插**的方式将数据放入链表（后面详细解析）。

#### 3.indexFor 根据hash值以及数组长度，获取当前key在数组中下标

```java
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}
```

​        在上一步的**hash方法**中算出了**key**对应的**hash值**，在这里根据**hash值**和**数组长度(length)**使用**与（&）**操作获取**key在数组中的下标**，这个操作的作用类似于计算中的%(求模)。源码中也写提到：**length必须为2的非零幂**，在**roundUpToPowerOf2** 方法得到确保，也就是**数组初始化长度为2的非0次幂**。

#### 4.遍历数组对应的链表，判断是否存在key重复

```java
for (Entry<K,V> e = table[i]; e != null; e = e.next) {
    Object k;
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
        V oldValue = e.value;
        e.value = value;
        e.recordAccess(this);
        return oldValue;
    } 
}
```

​        根据上一步获取的**数组下标**得到**链表的头节点**，**遍历链表**中的数据。如果存在**key，hash值相等**，那么就会**覆盖value**值，并且**返回之前的value值**

#### 5.addEntry 将数据以头插的方式放入链表中

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}
```

首先会判断**size是否大于等于阈值**以及**table当前下标是不为空**。当我们**初始化HashMap(new HashMap())**时，没有传入默认参数。**第一次put**的时候，**size=0**而**threshold**在**初始化HashMap**的时候是**等于初始化容量**，但是在**第一次put**的时候会**初始化table**，所以threshold=16*0.75=12

![img](.\assets\initSize.png)

​       上图我们是根据HashMap调用**默认构造方法**，**第一次put**数据来分析的。**threshold=12**，第一次put数据时候,整个table数组都是空的只是初始化了长度(16)。显然以上条件，都是不成立的(PS:此处是以**默认构造方法**并且**第一次put数据**以此条件分析的)。从上面我们可以得到**阈值=容量*负载因子**

​      上述方法如果过后会执行**createEntry**，创建**Entry**对象以**头插**方式插入到链表

##### 1.resize 扩容

```java
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
```

​        从上面分析可以知道，当**size大于等于阈值**并且**当前下标(根据hash计算出来的)对应的table(数组)不为空**，此时HashMap就会扩容，并且每次在之前数组长度*2(2 * table.length)。

​        首先会判断之前的容量是否等于最大的容量，如果等于阈值就等于Integer最大值，并返回。不等于则创建一个创建一个新的数组大小是原来的两倍，执行transfer(扩容具体方法)，table指向newTable，重新计算阈值

###### 1.transfer 将之前table(数组)的数据转移到新的数组上

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            //获取e指向的Entry
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            //e指向扩容后的下标对应的newTable
            e.next = newTable[i];
            //将e放入数组中，头插
            newTable[i] = e;
            //将next赋给e
            e = next;
        }
    }
}
```

​       **initHashSeedAsNeeded方法：默认情况下，当容量>=Integer.MAX_VALUE或者在启动参数(-D jdk.map.althashing.threshold）传入一个非零的值，switcing才会为true。rehash才会等于true** ，所以**一般情况下都是fasle**，**详细**initHashSeedAsNeeded描述在**上面有讲解**

**PS:在多线程的时候此处可能会出现循环链表**

​       通过for循环遍历之前的table，然后通过while遍历数组上的链表。这样可以获取HasMap中所有的数据，然后根据每个Entry的hash值和新的长度(扩容后的数组长度)求出新的下标(i)，再将逐个元素转移到新的数组上。![transfer_20201023004139](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\transfer.gif)

​        **根据上述图片分析,假设原来table长度为4，新扩容后的table为原来的2倍即8。在table上下标为1的位置开始(应该是从0的位置开始遍历，这里笔者为了方便演示)为e，e指向next，然后根据新的容量求出e在newTable中的下标。接着e指向求出下标对应的位置(刚开始newTable[i]=null)，然后将e放到数组中(头插)。最后将e指向table中的下一个节点next，继续迭代。**

##### 2.createEntry 创建新的Entry对象

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

​       这里的代码就比较简单了，先**取出e(当前下标位置对应的Entry)**,然后**创建一个新的Entry对象指向e**,并**插入e对应位置**,依然使用的是**头插法**。

### 3.get

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}
```

​        首先判断**key是否是nu**ll，如果**是空**的话就执行并返回**getForNullKey**，否则根据**key从getEntry获取到Entry对象**，最后判断返回的是否为空，来**返回null或者Entry对象的value**。

#### 1.getForNullKey key为空的情况

```java
private V getForNullKey() {
    if (size == 0) {
        return null;
    }
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value;
    }
    return null;
}
```

​        在前面的**putForNullKey** 方法中提到过，**HashMap**会将**key为null**的值存到**数组的第一个位置**(**table[0]**),因此只需遍历**table[0]下面的链表**来获取到对应的**value**

#### 2.getEntry 根据key获取Entry

```java
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

​        使用**key**获取对应的**hash值**，根据**hash值**以及**数组的长度(table.length)**来获取到在**table中的下标**。遍历**数组对应下标的链**表。判断，找出对应的key并返回，没有找到就返回null。

### 4.remove 快速失败

#### 1.问题

在使用HashMap**移除元素**的时候，如果私用自带的**map.remove()**方法会可能出现**ConcurrentModificationException异常**，但是使用**迭代器中的remove()**方法就不会出现问题，这是为什么呢？

#### 2.分析

首先我们看看**ConcurrentModificationException**在HashMap中位置吧，两次都出现在抽象内部类**HashIterator **的 **nextEntry**  **remove**方法中

![image-20201023022336631](.\assets\concurrentModificationException.png)

构造方法:在里面对expectedModCount进行了复制

![image-20201023024204875](.\assets\HashIteratorCus.png)

​        由此分析出现异常的原因就是**modCount != expectedModCount**，**modCount**想必大家都见过，在put方法出现过，每次put一个值都会**modCount++** 。不仅如此，而且在**putForNullKey**，**removeEntryForKey**，**removeMapping**，**clear**都有**modCount++**。**modCount**的作用就是**记录HashMap中增删改(变更)的次数**，每次+1。



  **HashIterator**有三个内部类继承，**ValueIterator**，**KeyIterator**，**EntryIterator**分别来看看

![image-20201023023506717](.\assets\HashIterator.png)

​       仔细一看这不正是我们在使用迭代器获取**key** **value** **下一个Entry对象**的相应实现吗,就是说我们每次在迭代的时候都会去比较**modCount != expectedModCount**

##### 1.HashMap.remove方法

![image-20201023024528298](.\assets\hashmap-remove.png)

**遍历链表移除对应元素，modCount++**

##### 2.Iterator.remove方法

![image-20201023024821289](.\assets\iterator-remove.png)

可以看出依然是调的**HashMap.this.removeEntryForKey**方法，但是在后面有**expectedModCount = modCount**，将改变后的**modcCount**赋给**expectedModCount**，那么下一次迭代的时候判断就是相等的，就不会出现异常。

#### 3.结论

**综上分析**: **HashMap在要迭代的之前，初始化迭代器调用HashIterator()并赋值expectedModCount = modCount。然后每次迭代的时候判断modCount != expectedModCount。使用HashMap自带的remove方法，只会删除数据expectedModCount不会还原。Iterator的remove调用HashMap中的方法还加上了expectedModCount = modCount，如此以来下次迭代判断的时候不会出现问题。**

HaspMap之所以要这样，我的理解是:**HashMap本来就不是线程安全的**，所以他要尽可能的避免在获取数据的同时修改数据，特意加上了这种弥补机制(**快速失败**)。



## 3.HashMap相关面试题

**坐等各位大佬留言，到时同步上去**

## 4.总结

​       **在分析HashMap的源码中，使用了大量的位操作：数组容量的确认，计算hash值，根据hash值确认数组下标等等。就是这些操作也同时决定了一些东西，数组的容量为什么是>=2的n次幂，数组为什么每次扩容2倍（我想都可能和位操作有关系）。使用HashMap的时候也应该尽可能避免扩容，最好给定指定的容量和负载因子。因为在多线程的情况扩容可能会造成循环链表，而且老数组的数据需要一个个的移到新数组上，开销是比较大的。**



如果哪个地方写的有问题还希望大家毫不吝啬指出来，谢谢啦！

