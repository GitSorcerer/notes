# HashMap-JDK1.7&1.8区别

在上两篇对HashMap在JDK1.7与1.8原源码进行了分析，了解了HashMap中的实现原理。在阅读源码的时候，看了底层的设计和操作以及思想无不佩服HashMap的作者**Doug Lea**，JUC也是出自他之手。本片我们来对比HashMap在1.7与1.8的差异性



## 1.JDK1.7 HashMap

### 1.1 数据结构

**数组+链表,类型都为Entry**

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\HashMap1.7.png)

### 1.2 扩容

#### 1.2.1 扩容条件

**(size >= threshold) && (null != table[bucketIndex])**,当容量>阈值并且table对应下标位置为空，就开始扩容

#### 1.2.2 扩容方式

每次容量扩大为原来的两倍，重新根据hash值和新的容量确定数据在新的table中的位置，以头插法的方式将数据插入到table下标对应链表中。

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {//遍历table
        while(null != e) {//遍历table下面的链表
            Entry<K,V> next = e.next;
            if (rehash) {//默认为false
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);//计算hash值在新table中的位置
            e.next = newTable[i];//头插
            newTable[i] = e;
            e = next;
        }
    }
}
```

以上代码就是HashMap1.7扩容的核心代码，在多线程的情况下会出现循环链表。

### 1.3 插入数据

每次以头插的方式插入数据

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];//获取到tables上的节点
    table[bucketIndex] = new Entry<>(hash, key, value, e);//创建新的节点插入到table上，并指向之前的e
    size++;
}
```

## 2.JDK1.8 HashMap

### 2.1 数据结构

**数组+链表+红黑树，数组和链表为Node类型，红黑树为TreeNode类型**

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\HashMap1.8.png)

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\node-type.png)

Node

```java
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    V value;
    Node<K, V> next; //下一个节点
    //......
}
```

TreeNode

```java
static final class TreeNode<K, V> extends LinkedHashMap.Entry<K, V> {
    TreeNode<K, V> parent;  // red-black tree links  父节点
    TreeNode<K, V> left; //左节点
    TreeNode<K, V> right;//右节点
    TreeNode<K, V> prev;    // needed to unlink next upon deletion 上一个节点
    boolean red;//颜色
    //......
}
```

从上面两张图和类型我们基本可以看出**TreeNode是Node的子类**，**Node的基本结构是一个单向链表**，**TreeNode是双向链表+红黑树**

### 2.2 扩容

扩容方法:resize()方法，是HashMap1.8最重要的方法。在resize()方法既可以初始化容量，又可以完成扩容。该方法涉及到链表、红黑树、双向链表的一系列操作(虽然没有直接操作)。

#### 2.2.1 扩容条件

**容量>阈值、table为空或长度为0、table长度<MIN_TREEIFY_CAPACITY(64),treeifyBin()树化方法中出现。**

满足上面三种条件的一种都会发生扩容，出现最多的还是第一种(容量>阈值)

#### 2.2.2 扩容方式

在扩容的时候因为容量总是2的n次幂，扩容之后的数据会有两个规律：还是在当前table下面(下标不变)、位置变为下标+之前table容量。

**PS：上述所说的下标指的是根据hash值和容量求出数据在table中的位置**

根据上面规律，不变那些数据放到低位，变动的放在低位，最后只需转移高低位链表的头即可(红黑树比较复杂)

##### 2.2.2.1 链表扩容

遍历table下面的单向链表，分为高位和低位。低位还是当前位置(newTab[j] = loHead),将低位的头节点插到newTabde的当前下标(j)位置。高位(newTab[j + oldCap] = hiHead),高位头节点插入到newTab的j+oldCap位置。

##### 2.2.2.2 红黑树扩容

依然遍历当前table下面的数据，但是形成的高位和低位的双向链表。还是以上述方式转移过去，其中会出现链表化或树化的过程，因为链表发生变化红黑树势必也会发生变化，这个时候就需要根据高低位双向链表长度判断是否需要重新树化或链表化。当双向链表长度<=6就会链表化

### 2.3 插入数据

插入数据的时候首先需要判断table下面是否为空，为空就将数据直接放到table下面(tab[i] = newNode(hash, key, value, null))。table下面有数据，就需要判断是链表还是红黑树。红黑树就以红黑树的方式插入(比较复杂)。链表就以尾插法的方式插入数据，插入后如果链表长度>=8,就需要树化。

```java
if ((p = tab[i = (n - 1) & hash]) == null)//根据hash值来确认存放的位置。如果当前位置是空直接添加到table中 PS: 多线程情况下可能出现值覆盖
    tab[i] = newNode(hash, key, value, null);
```

## 3.总结

通过上面对HashMap-JDK1.7与1.8中的数据结构、扩容、插入数据进行了单独的分析。以下我们来进行比较。

### 3.1 结构

1.7 数组+链表  结构比较简单，当数据多了查询会变得比较慢，但链表增删快

1.8 中加入了红黑树，提高了查询插入的效率，但是红黑树部分操作比较复杂，让代码也变得比较复杂。

### 3.2 扩容

1.7 中扩容涉及到链表的转移，重新计算下标以头插的方式转移到新的table上。在多线程的情况下会出现循环链表。

1.8 中扩容会将数据分为高低位，然后转移过去。如果是红黑树就会变为双向链表，同时可能会出现链表化，以及重新树化。

### 3.3 插入数据

1.7 将数据以头插的方式插入到链表中

1.8 如果table下面是链表以尾插的方式插入到链表中，如果长度>=8,会将链表树化。如果table下面是红黑树(TreeNode),插入到红黑树中，通过旋转变色调整红黑树平衡(操作比较复杂)



## 链接

[HashMap1.7](https://blog.csdn.net/chianz632/article/details/109234509)

[HashMap1.8](https://blog.csdn.net/chianz632/article/details/112620651)

[红黑树](https://blog.csdn.net/chianz632/article/details/111243829)