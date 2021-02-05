# HashMap1.8源码解析

## 1.介绍

之前写了一篇[HashMap 1.7源码解析](https://blog.csdn.net/chianz632/article/details/109234509)，本片将对HashMap1.8进行分析。后续再写一篇1.7和1.8中HashMap的区别,之前1.7中出现的一些代码我就不赘述了。**HashMap1.8是由：数组(Node)+链表(Node)+红黑树(TreeNode)** 组成，而且**TreeNode继承与Node** ，如下图所示。

![](\assets\HashMap1.8.png)

## 2.源码解析

​        一般我们在使用**HashMap**的时候，最好给一个默认的初始化容量，也就是调的**HashMap(int initialCapacity)**构造方法。在我们给出具体容量的时候就可以防止HashMap自动扩容，以免影响效率。

### 2.1 插入

### 2.1.1  putVal 源码

```java
/**
 * Implements Map.put and related methods
 *
 * @param hash         hash for key
 * @param key          the key
 * @param value        the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict        if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K, V>[] tab;
    Node<K, V> p;
    int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)//首次初始化的时候table为null
        n = (tab = resize()).length;//对HashMap进行扩容(初始化)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else { //如果存放的位置已经有值
        Node<K, V> e;
        K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))//如果插入位置key重复
            e = p;
        else if (p instanceof TreeNode)//如果插入的位置正好是红黑树
            e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value); 
        else {//如果以上条件均不满足  就说明还是链表
            for (int binCount = 0; ; ++binCount) {//遍历链表
                if ((e = p.next) == null) { //遍历到链表的尾节点
                    p.next = newNode(hash, key, value, null); 
                    if (binCount >= TREEIFY_THRESHOLD - 1)  //长度>=8
                        treeifyBin(tab, hash); //转变为红黑树
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;//在循环的过程中遇到了key值相同的节点，跳出循环
                p = e;//和前面for中的p.next对应，进行循环遍历链表。
            }
        }
        if (e != null) { // existing mapping for key   如果e!=null  说明存在相同的key
            V oldValue = e.value;//记录下原来的key对应的value
            if (!onlyIfAbsent || oldValue == null)//当onlyIfAbsent为false或者旧值为空
                e.value = value;//替换新的value并返回旧的value
            afterNodeAccess(e);
            return oldValue;//返回旧值，这样就处理掉了前面的当key值相同时的情况了。
        }
    }
    ++modCount;// 每次容器发生变化记录
    if (++size > threshold)// 实际大小大于阈值则扩容
        resize();//如果当前HashMap的容量超过threshold则进行扩容
    afterNodeInsertion(evict);
    return null;
}
```

2.1.2 介绍

首先我们需要知道HashMap在put的时候的逻辑，然后逐行分解。

![](\assets\put-xmd.png)

### 2.1.3 核心操作

### 2.1.3.1 resize() 扩容

**resize()**可以说是HashMap最重要的方法之一，**初始化**以及**size>threshold(阈值=容量*负载因子)**的时候都会**调用该方法**。首先我们来分析一下这个方法的作用，紧接着贴上代码。

> 1.重新生成容量以及阈值
>
> 2.将在之前table上的数据转移到扩容后的table上 
>
> 1. 转移红黑树
> 2. 转移链表

### 2.1.3.1 重新生成容量以及阈值

```java
int oldCap = (oldTab == null) ? 0 : oldTab.length;
int oldThr = threshold;//默认构造器的情况下为0
int newCap, newThr = 0;//新容量 新阈值
if (oldCap > 0) {//table扩容过
    if (oldCap >= MAXIMUM_CAPACITY) { //当前table容量大于最大值得时候返回当前table 
        threshold = Integer.MAX_VALUE;
        return oldTab;
    } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
               oldCap >= DEFAULT_INITIAL_CAPACITY) 
        newThr = oldThr << 1; // double threshold   table的容量乘以2，threshold的值也乘以2
} else if (oldThr > 0) // initial capacity was placed in threshold
    newCap = oldThr;//使用带有初始容量的构造器时，table容量为初始化得到的threshold
else {               // zero initial threshold signifies using defaults//默认构造器下进行扩容
    newCap = DEFAULT_INITIAL_CAPACITY;
    newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
}
if (newThr == 0) { //使用带有初始容量的构造器在此处进行扩容
    float ft = (float) newCap * loadFactor;
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ?
              (int) ft : Integer.MAX_VALUE);
}
threshold = newThr;
```

从以上代码可看出：

> 1.oldCap>0    容量扩大为原来的两倍
>
> ​       oldCap>=MAXIMUM_CAPACITY(1 << 30)   之前容量>=最大容量
>
>    		无法扩容，只能改变阈值(threshold = Integer.MAX_VALUE)，直接返回
>
> ​	   **newCap = oldCap << 1(扩容两倍 )** 后的值<=MAXIMUM_CAPACITY  并且 之前容量>最大容量  
>
> 2.oldThr > 0    调用有参构造方法 
>
> ​    newCap=oldThr  // new Hash(6) newCap=8; new Hash(12) newCap=16 详细可看 tableSizeFor方法的作用
>
> 3.默认构造方法(无参构造)
>
> ​    newCap=DEFAULT_INITIAL_CAPACITY  16
>
> ​    newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY)   16*0.75=12
>
> 4.确保newThr不为0

### 2.1.3.2 数据转移

```java
for (int j = 0; j < oldCap; ++j) {
    Node<K, V> e;
    if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        if (e.next == null)//
            newTab[e.hash & (newCap - 1)] = e;//(n - 1) & hash
        else if (e instanceof TreeNode)// 如果当前数组下对应的下标 是一个红黑树
            ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
        else { // preserve order
            Node<K, V> loHead = null, loTail = null;//低位
            Node<K, V> hiHead = null, hiTail = null;//高位
            Node<K, V> next;
            do {
                next = e.next;
                if ((e.hash & oldCap) == 0) { // 扩容后不需要移动的链表（这个决定放在原来位置还是原来位置加上原来数组长度。lo开头的是放在原来索引上，hi开头的是放在原来索引加上数组长度上。）
                    if (loTail == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                } else {// 扩容后需要移动的链表
                    if (hiTail == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                }
            } while ((e = next) != null);
            if (loTail != null) {
                loTail.next = null;
                newTab[j] = loHead;
            }
            if (hiTail != null) {
                hiTail.next = null;
                newTab[j + oldCap] = hiHead;// 扩容长度为当前index位置+旧的容量
            }
        }
    }
}
```

根据以上代码分析可以看出是在**循环遍历之前table(oldTab)的数据**，对应**oldTab(oldTab[j])**下面如果不为空，就会出现一下三种情况。**e = oldTab[j] 也就是数组(table)下标对应的位置**

> 1. **e.next=null 说明当前位置下面既没有红黑树也没有链表。那么就将根据 e.hash & (newCap - 1) 求出当前值在新table中的位置。**
> 2. **e instanceof TreeNode  说明当前位置下面挂着一颗红黑树，然后根据TreeNode的split方法进行操作。**
> 3. **不满足以上两种条件  说明当前位置下面是一条链表，那么就遍历链表元素 移动到新的链表上。**

### 2.1.3.3  hash值计算下标原理

首先我们先了解**e.hash & (newCap - 1) 、e.hash & (oldCap - 1)、e.hash & oldCap** 这个三个操作，**&**位与：对二进制操作，**都为1则为1**。

下面我们假设**oldCap=16、newCap =32**，模拟了三个hash值**7862、9812、6798**对上面的操作进行求值。

![](\assets\位&1.png)

![](\assets\位&2.png)

**不管是oldCap还是newCap都是2的n次幂(2^n),而 e.hash & (2^n- 1) 的结果实质上就是取hash值的低n位。**

假设**oldCap =16(2^4)**,那么对任意**hash值进行e.hash & (oldCap - 1)**操作的时候会得到**低4位，假设为 qwer ,（16-1）二进制 1111**    

**PS:前面的0省略**

> qwer      
>
>    &
>
> 1111
>
>   ||
>
> qwer

**newCap=32(2^5)**,取低5位会有两种情况

> 1qwer
>
> 0qwer

**一. 0qwer就表示扩容后计算的值(下标)没有变化**

***二. 1qwer=oldCap(1 0000)+qwer* ----> *oldCap+之前计算的值(下标)*，对应的代码就是  oldCap+j** 

在**(e.hash & oldCap) == 0**的时候，说明第五位一定为0。扩容后下标不会变化，也就是上面第一种情况，反之就是第二种

> 0 qwer  e.hash
>
> ​     &
>
> 1 0000   16(这是没有-1的结果)
>
> ​    ||
>
> 0 0000

综上图片以及文字分析：在扩容的时候，扩容后数据的位置会有两种情况

> 1. 依然在当前下标对应的位置
> 2. 当前下标+之前数组的长度

**像我上图一样自己可以模拟几组数据试一下，就知道其中的奥秘**

**如果对二进制转换不是很熟，可以借助windows自带的工具*计算器* 将左上角切换成*程序员*，即可看到各种进制的转换。**

### 2.1.3.4 链表扩容

当我们搞清楚了根据hash值计算下标的原理后，来看扩容就很清楚了。

> ```java
> Node<K, V> loHead = null, loTail = null;//低位
> Node<K, V> hiHead = null, hiTail = null;//高位
> ```

首先定义了四个节点，高低位的头尾节点

**循环遍历当前下标对应的链表,循环结束后会形成两个链表。loHead指向低位的头节点，loTail指向低位的尾节点；hiHead指向高位的头节点，hiTail指向高位的尾节点。**

> ```java
> if ((e.hash & oldCap) == 0) {//低位 扩容后位置不变
>     if (loTail == null)
>     	loHead = e;
>     else
>    		loTail.next = e;
>     loTail = e;//每次将loTail(尾部)指向e
> } 
> ```

如果满足  **e.hash & oldCap == 0** 这一条件，说明**扩容后节点的位置没有发生变化**。就将**loHead指向它们的头部**，**loTail指向尾部**。否者就用高位的头尾节点指向它们。

> ```java
> if (loTail != null) {//低位
>     loTail.next = null;
>     newTab[j] = loHead;//还是在j下标下面 直接将低位整个链表转移过去
> }
> if (hiTail != null) {//高位
>     hiTail.next = null;
>     newTab[j + oldCap] = hiHead;//扩容后下标变为 j+oldCap 直接将整个高位链表转移过去
> }
> ```

**链表扩容总结：链表在扩容的时候，会形成两条新的链表。分别是低位((e.hash & oldCap) == 0)和高位,扩容后直接将低位对应的链表放在新数组当前下标(newTab[j])下面，高位对应的链表放在新数组中j+oldCap下面。**

### 2.1.3.5 split 红黑树扩容

介绍：

> 红黑树的扩容在计算下标的时候也和链表类似，红黑树也会**形成两个链表但是这两个链表是双向的**。并且会判断**高低位链表的长度是否<=6**,如果小于会将**双向链表转换为链表**，否者会将**双向链表树化**，最后插入到对应的位置。

**2.1.3.5.1 TreeNode.split()方法,扩容后转化成链表或者红黑树**

```java
//                                      扩容后的table  求出来的下标    之前的容量
final void split(HashMap<K, V> map, Node<K, V>[] tab, int index, int bit)
```

**TreeNode继承了Node**的属性，自身还有一个**prev**元素。所以**TreeNode不仅是一颗红黑树，还是双向链表**

```java
TreeNode<K, V> b = this;//当前红黑树的根节点  如果进到这一步 说明table下面挂的一定是一棵红黑树
// Relink into lo and hi lists, preserving order
TreeNode<K, V> loHead = null, loTail = null;//低位
TreeNode<K, V> hiHead = null, hiTail = null;//高位
int lc = 0, hc = 0;//lc hc 分别记录高位和低位双向链表长度  用于后面的树化 或者链表化判断
```

**lo，hi 用来区扩容后低位和高位双向链表的头尾节点**

```java
if ((e.hash & bit) == 0) {//扩容后位置不发生变化 放在低位
    if ((e.prev = loTail) == null)//上一个节点指向尾节点 
        loHead = e;//第一次进来 loHead指向头节点
    else
        loTail.next = e;//下一个节点指向 e
    loTail = e;//尾节点变为e节点 继续递归
    ++lc;//记录长度
}
```

高位操作和这个类似，遍历结束后，就会形成两条链表：**loHead, loTail指向低位头和尾的双向链表；hiHead， hiTail指向高位头和尾的双向链表。并且会用lc，hc记录想要双向链表的长度**

```java
if (loHead != null) {//低位
    if (lc <= UNTREEIFY_THRESHOLD)//如果低位长度<=6
        tab[index] = loHead.untreeify(map);//位置不变 将双向链表 变为 链表
    else {
        tab[index] = loHead;//将头插入到 table中
        if (hiHead != null) // (else is already treeified)
            loHead.treeify(tab);//如果高位头不为空 将低位树化
    }
}
if (hiHead != null) {//高位
    if (hc <= UNTREEIFY_THRESHOLD)//如果高位长度<=6
        tab[index + bit] = hiHead.untreeify(map);//在 index+bit 位置退化成链表 
    else {
        tab[index + bit] = hiHead;
        if (loHead != null)//低位头不为空  
            hiHead.treeify(tab);//转换成树
    }
}
```

以上就是根据高低位链表的长度，决定将链表树化，还是链表化。**需要注意的是，在树化的时候加了hiHead != null、loHead!= null判断。**

原因：

>在红黑树扩容的时候，扩容后会分成高位和低位。扩容后可能出现，整颗树的下标没有发生变化，也可能整棵树下标全都改变。就会出现低位或者高位为空这种情况。
>
>hiHead != null  为fasle高位为空,扩容后下标没发生改变，直接转移不进行树化。
>
>loHead != null  为false低位为空,扩容后下标全都改变了，直接转移不进行树化。

**PS:可能有人会由疑惑，在for循环的时候,为什么需要使用双向链表;以及红黑树结构会不会改变。**

**1.在扩容后红黑树可能会改变(for循环不会)，导致根节点发生变化，使用红黑树方便后面重新调整根节点的位置**

**2.在上面for循环产生双向链表的时候并不会破坏红黑树，循环的时候只是next、prev发生变化，红黑树的left，rigth等元素都没发生变化。所以在高位或者低位为空的时候可以直接将头放到table下面。**



**2.1.3.5.2 Node<K, V> untreeify(HashMap<K, V> map) **

```java
/**
 * Returns a list of non-TreeNodes replacing those linked from
 * this node.
 */
final Node<K, V> untreeify(HashMap<K, V> map) {// 高位或低位头.untreeify(map)
    Node<K, V> hd = null, tl = null;//头尾节点    
    for (Node<K, V> q = this; q != null; q = q.next) {//this 当前红黑树的根节点
        Node<K, V> p = map.replacementNode(q, null);//将红黑树 节点 替换为 Node节点
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```

**untreeify()是TreeNode中的方法,TreeNode继承自Node。该方法作用是将红黑树转换为链表，在前面已经将需要转换的红黑树已近转变为双向链表，所以只需遍历双向链表，转换成链表即可。**

### 2.1.3.6 红黑树 treeify(Node<K, V>[] tab)树化

```java
final void treeify(Node<K, V>[] tab) {
    TreeNode<K, V> root = null;//树的根节点
    for (TreeNode<K, V> x = this, next; x != null; x = next) {//遍历链表  x = next 迭代
        next = (TreeNode<K, V>) x.next;
        x.left = x.right = null;//之前红黑树节点的 左右节点制空
        if (root == null) {//确认根节点 第一次循环才会进来
            x.parent = null;//父节点制空
            x.red = false;//变为黑色
            root = x;//x变为根节点
        } else {
            K k = x.key;// x 是需要插入到树下面的元素  x的key
            int h = x.hash;//x hash值
            Class<?> kc = null;//k的class类型
            for (TreeNode<K, V> p = root; ; ) {//遍历二叉树寻找插入位置
                int dir, ph;//dir 用于判断子节点插入的位置（左边还是有i按） ph插入节点父节点的hash值
                K pk = p.key;//父节点的key
                if ((ph = p.hash) > h)//以下根据hash值计算x插入的位置
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&//如果hash值相等 就进行一下操作
                        (kc = comparableClassFor(k)) == null) ||//第一次判断
                        (dir = compareComparables(kc, k, pk)) == 0)//第二次判断
                    dir = tieBreakOrder(k, pk);//保险栓 上面两次判读都行不通就用该方法（下文介绍）

                TreeNode<K, V> xp = p;//x的父节点
                if ((p = (dir <= 0) ? p.left : p.right) == null) {//判断插入的位置是否为空，不为空就继续进行内层循环
                    x.parent = xp;
                    if (dir <= 0)//插入左边
                        xp.left = x;
                    else
                        xp.right = x;//右边
                    root = balanceInsertion(root, x);//调整红黑树
                    break;//找到插入的位置结束
                }
            }
        }
    }
    moveRootToFront(tab, root);//确保root的节点是tab数组(桶)的第一个节点
}
```

分析以上的代码，可以看到会有**两次for循环。第一次for循环遍历链表，第二次for循环遍历红黑树，将当前链表对应的元素插入到红黑树中。**

![](\assets\treeify.png)

由于在调用treeify方法之前，已经将红黑树拆成了两条双向链表，之前的红黑树已经破坏（所以需要x.left = x.right = null，将之前的左右节点制空），变成了双向链表。treeify方法的目的就是将双向链表变为红黑树。所以就需要遍历双向链表，以链表的第一个节点为根节点不断的向下插入数据,因此需要第二次for循环来遍历树，找到插入位置。

### 2.1.3.7 comparableClassFor、compareComparables、tieBreakOrder 判断插入位置

红黑树插入的时候，会根据hash值大小判断往左边还是右边插入节点，以及插入后保持红黑树平衡操作，当hash值相等时，就会进行一系列判断。

```java
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c;
        Type[] ts, as;
        Type t;//Type 是  所有类型 的公共高级接口。它们包括原始类型、参数化类型(泛型)、数组类型、类型变量和基本类型。
        ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        if ((ts = c.getGenericInterfaces()) != null) {//实现的接口类型 返回  Type[]
            for (int i = 0; i < ts.length; ++i) {
                if (((t = ts[i]) instanceof ParameterizedType) &&//参数化类型(也就是泛型)
                        ((p = (ParameterizedType) t).getRawType() ==//返回 Type 对象，表示声明此类型的类或接口。  返回最外层<>前面那个类型，例如Comparable<T>，返回的是Comparable类型。
                                Comparable.class) &&
                        (as = p.getActualTypeArguments()) != null &&//返回表示此类型实际类型参数的 Type 对象的数组 也就是获得参数化类型中<>里的类型参数的类型
                        as.length == 1 && as[0] == c) // type arg is c   如果只有一个泛型 直接取第0个 例如Comparable<Integer>  c=Integer.class
                    return c;
            }
        }
    }
    return null;
}
```

**comparableClassFor的作用就是判断插入的key是否实现了Comparable，如果没有实现直接返回空；否则继续往下判断**

> 1. 如果插入节点key的Class类型是String，返回String.class
>
> 2. 取出插入节点key实现的的接口，循环判断实现接口
>
>     如果对应的接口的RawType类型为Comparable.class,并且Type只有一个，那么直接取出那一个然后返回
>
>    例如：key实现了Comparable<Integer>这个接口，那么就会返回Integer.class

```java
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable) k).compareTo(x));
}
```

**如果x=null或者两个节点的class不相等，那么就不能无法比较，返回0；否则通过compareTo返回对应的比较后的值。**

```java
static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
            (d = a.getClass().getName().
                    compareTo(b.getClass().getName())) == 0)//通过获取key的className判断
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                -1 : 1);//通过系统的identityHashCode 进行判断
    return d;
}
```

**如果通过以上的两个方法计算的kc==null或者dir==0,那么就会通过tieBreakOrder进行最终的判断**

**如果a，b不为空，那么就先比较a，b的类的全路径(String字符串实现了Comparable)进行比较，如果还是相同...**

**那么系统可能急了(WTM都急了)，直接调用系统自带的System.identityHashCode进行判断，identityHashCode是native方法，返回对象对应的hash地址值,最终完成判断。**

### 2.1.3.8 balanceInsertion 红黑树插入平衡

其中我们只分析**插入节点的父节点是插入节点爷爷节点的左子节点**进行分析，另外一种情况完全想法，笔者这里就不赘述了。

```java
/**
 * 
 * @param root 根节点
 * @param x 插入节点
 * @param <K>
 * @param <V>
 * @return
 */
static <K, V> TreeNode<K, V> balanceInsertion(TreeNode<K, V> root,
                                              TreeNode<K, V> x) {
    x.red = true;//插入节点默认为红色    xp x父节点 xpp x爷爷节点 xppl x左叔叔节点  
    for (TreeNode<K, V> xp, xpp, xppl, xppr; ; ) { //xppr x的右叔叔节点
        if ((xp = x.parent) == null) {//x的父节点如果为空  说明暂无节点
            x.red = false;//x变为黑色
            return x;//x 就是根节点 返回
        } else if (!xp.red || (xpp = xp.parent) == null)//xp(x的父节点是黑色) 或者 xp的父节点为空(插入的节点的爷爷节点为null，即插入的节点的父节点是根节点，直接插入即可（因为根节点肯定是黑色）)
            return root;//插入的节点的父节点是黑色节点，则不需要调整，因为插入的节点会初始化为红色节点，红色节点是不会影响树的平衡的
        if (xp == (xppl = xpp.left)) {//插入的节点父节点和祖父节点都存在，并且其父节点是爷爷节点的左节点
            if ((xppr = xpp.right) != null && xppr.red) {//插入节点的叔叔节点是红色
                xppr.red = false;//叔叔节点变黑
                xp.red = false;//父节点变黑
                xpp.red = true;//爷爷节点变红
                x = xpp;//x指向爷爷节点继续向上递归
            } else {//插入节点的叔叔节点是黑色或不存在
                if (x == xp.right) {//若插入节点是其父节点的右孩子，则将其父节点左旋
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {//若为左孩子，则将其父节点变成黑色节点，将其爷爷节点变成红色节点，然后将其爷爷节点右旋
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        } else {//父节点是祖父节点的右节点
            //...省略
        }
    }
}
```

**PS:以下我们根据插入节点的父节点是爷爷节点的左节点分析,另一种情况(插入节点的父节点是爷爷节点的右节点)操作完全相反.只要掌握其中一种情况即可.**

**1.插入节点X的父节点和叔叔节点都是红色**

![](\assets\insert_RbTree1.png)

> 1.**父节点和叔叔节点变黑,爷爷节点变红**。然后将**x节点指向爷爷节点**，继续**往上递归**
>
> 2.x节点为**根节点**，直接将x节点**变红**

**2.插入节点x的父节是红色,并且插入节点是父节点的左节点**

![](\assets\insert_RbTree2.png)

> 首先**父节点变黑**,**爷爷节点变红**, 然后根据**x的父节点右旋**.

**3.插入节点x的父节是红色,并且插入节点是父节点的右节点**

![](\assets\insert_RbTree3.png)

> 直接根据x的父节点左旋，转换为第二种情况,然后通过第二种情况处理

插入平衡总结

**一. x节点的父节点为空，也就是判断是否存在根节点，不存在就直接将插入节点变黑并成为根节点；**

**二.插入节点爷爷节点为空，也就是父节点是根节点，是的话不需要调整平衡(应为插入节点默认为红色，根节点黑色)；**

**三.插入的节点父节点和祖父节点都存在，以父节点是祖父节点的左节点进行分析**

**1.叔叔节点是红色，x指向爷爷节点，变色，继续往上递归。**

**2.叔叔节点是黑色（或空），此时x是父节点的左节点。根据爷爷节点右旋。**

**3.x是父节点的右子节点，根据x父节点左旋，变为图上的【情况二】**

**详细操作可查看我之前写的博客[红黑树增删查操作](https://blog.csdn.net/chianz632/article/details/111243829)其中的插入操作**

### 2.1.3.9 rotateLeft左旋

```java
/**
 * 左旋
 *             pp                        pp
 *            /                         /
 *           p                        r
 *         /  \                     /  \
 *              r                  p     
 *             / \                / \
 *            rl                     rl
 *
 * @param root  根节点
 * @param p     旋转点
 * @param <K>
 * @param <V>
 * @return 返回根节点  因为根节点在旋转的过程中可能会改变 就需要返回改变后的
 */
static <K, V> TreeNode<K, V> rotateLeft(TreeNode<K, V> root,
                                        TreeNode<K, V> p) {
    TreeNode<K, V> r, pp, rl;//r p的右节点 pp p的父节点 rl r的左节点
    if (p != null && (r = p.right) != null) {//判断p的右节点是否存在 并且给r赋值
        if ((rl = p.right = r.left) != null)//如果r的左节点(rl)存在 并且把r的左节(rl)点变成p的右节点
            rl.parent = p;//p 变成 rl的父节点
        if ((pp = r.parent = p.parent) == null)//判断p的父节点是否存在 并且p的父节点赋值给r的父节点
            (root = r).red = false;//如果p的父节点为空(p就是根节点) 则r变黑色(变为新的根节点) 并赋给root节点
        else if (pp.left == p)//如果p的父节点(pp)不为空 并且p为pp的左节点
            pp.left = r;//将r 变成pp的左节点
        else
            pp.right = r;//否则 r变成pp的右节点
        r.left = p;//将 p节点 变为 r 左节点
        p.parent = r;//r变成 p的父节点
    }
    return root;// 返回根节点
}
```

**左旋:旋转点(p)变为其右节点(r)的左节点,右节点的左节点(rl)变为旋转点(p)的右节点.**

### 2.1.3.10 rotateRight 右旋

```java
/**
 * 右旋
 *            pp                      pp
 *            /                       /
 *           p                       l
 *         /  \                    /   \
 *        l                             p
 *       / \                           / \
 *          lr                        lr   
 *
 * @param root  根节点
 * @param p     旋转点
 * @param <K>
 * @param <V>
 * @return 返回根节点  因为根节点在旋转的过程中可能会改变 就需要返回改变后的
 */
static <K, V> TreeNode<K, V> rotateRight(TreeNode<K, V> root,
                                         TreeNode<K, V> p) {
    TreeNode<K, V> l, pp, lr;//l p的左节点 pp p的父节点 lr l的右节点
    if (p != null && (l = p.left) != null) {//p的左节点是否存在  并给l赋值
        if ((lr = p.left = l.right) != null)//判断l的右节点是否存在 并且 把l的右节变为p的左节点  
            lr.parent = p;//p变为lr的父节点
        if ((pp = l.parent = p.parent) == null)//判断p的父节点是否存在  把p的父节点变为l的父节点 并给pp赋值
            (root = l).red = false;//如果p的父节点为空(p就是根节点) 则l变黑色(变为新的根节点) 并赋给root节点
        else if (pp.right == p)//p为pp的右节点
            pp.right = l;//l变成pp的右节点
        else
            pp.left = l;//l变成pp的左节点
        l.right = p;//p变成l的右节点
        p.parent = l;//l变成p的父节点
    }
    return root;// 返回新根节点
}
```

**右旋:旋转点(p)变为其左节点(l)的右节点,左节点的右节点(lr)变为旋转点(p)的左节点.**

### 2.1.3.11  moveRootToFront 重新调整树的根节点

```java
static <K, V> void moveRootToFront(Node<K, V>[] tab, TreeNode<K, V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;//计算出下标
        TreeNode<K, V> first = (TreeNode<K, V>) tab[index];//frist就是之前链表中的XXHead
        if (root != first) {//如果root不是第一个节点，则将root放到第一个首节点位置
            Node<K, V> rn;//root的下一个节点
            tab[index] = root;//root节点赋值给tab对应的位置
            TreeNode<K, V> rp = root.prev;//root的上一个节点
            if ((rn = root.next) != null)//root的下一个节点不等于空
                ((TreeNode<K, V>) rn).prev = rp;
            if (rp != null)//root的上一个节点不为空
                rp.next = rn;
            if (first != null)
                first.prev = root;
            root.next = first;
            root.prev = null;
        }
        assert checkInvariants(root);//检查红黑树
    }
}
```

**HashMap中table下面如果是红黑树的那么table[i]这个位置放的肯定是红黑树的root节点。而在split方法中将链表的头放入table[i]中(tab[index] = loHead) 。 在进行树化的过程中，遍历双向链表树化(treeify方法介绍过),这个过程红黑树的root不一定是就是链表的头部,这个时候就需要通过moveRootToFront方法调整了.**

**PS:双向链表遍历树化的过程中,双向链表是不会发生变化的.因为next和prev没发生改变.而生成的红黑树需要不断的调整平衡,root可能发生变化.**

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\root_first.png)

**如上图所示,此时这条双向链表对应着一颗红黑树.moveRootToFront做的操作就是将root节点对应的位置变成双线链表的头,与此同时红黑树也不会发生变化,变化的只是双向链表.这样table[i]对应的就是红黑树的root节点，也是双向链表的头.**

**PS:这个时候就体现了双向链表的优点了,移动元素快,因为它记录了前后节点的位置.**

### 2.1.3.2  table下面为空，将节点直接插到数组上

```java
if ((p = tab[i = (n - 1) & hash]) == null)//根据hash值来确认存放的位置。如果当前位置是空直接添加到table中 PS: 多线程情况下可能出现值覆盖
            tab[i] = newNode(hash, key, value, null);
```

如果hash计算的数组下标对应的数组下面为空，就创建一个Node节点插入到tbale对应的位置。

### 2.1.3.3 table下面是树，调用putTreeVal方法

```java
/**
 * Tree version of putVal.
 * @param map HashMap
 * @param tab table数组
 * @param h   hash值
 * @param k   key
 * @param v   value
 * @return    返回重复的数据
 */
final TreeNode<K, V> putTreeVal(HashMap<K, V> map, Node<K, V>[] tab,
                                int h, K k, V v) {
    Class<?> kc = null;// 用于比较的class类别
    boolean searched = false;
    TreeNode<K, V> root = (parent != null) ? root() : this;//获取根节点
    for (TreeNode<K, V> p = root; ; ) {//从根节点遍历 寻找插入的位置
        int dir, ph;//dir 判断插入的位置 < 0 左边 > 0 右边  
        K pk;
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))//插入了相同的key 返回之前
            return p;
        else if ((kc == null &&
                (kc = comparableClassFor(k)) == null) ||//获取可比较类型
                (dir = compareComparables(kc, k, pk)) == 0) {//根据可比较类型进行比较
            if (!searched) {//查询 只会查询一次
                TreeNode<K, V> q, ch;
                searched = true;
                if (((ch = p.left) != null && //如果左边不为空 从左边开始寻找
                        (q = ch.find(h, k, kc)) != null) ||
                        ((ch = p.right) != null && //右边不为空 从右边开始寻找
                                (q = ch.find(h, k, kc)) != null))
                    return q; //返回找到的值
            }
            dir = tieBreakOrder(k, pk); // 必杀技 求出dir值 上文提到过
        }

        TreeNode<K, V> xp = p;
        if ((p = (dir <= 0) ? p.left : p.right) == null) {//如果x插入的位置(左或右)不为空
            Node<K, V> xpn = xp.next;//插入节点x父节点的上一个节点
            TreeNode<K, V> x = map.newTreeNode(h, k, v, xpn);//将xpn放在新节点x的下一个节点
            if (dir <= 0)
                xp.left = x;//插入到左边
            else
                xp.right = x;//插入到右边
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)//判断p节点对应的链表是否存在下一个节点
                ((TreeNode<K, V>) xpn).prev = x;  //将下一个节点插入到x后面
            moveRootToFront(tab, balanceInsertion(root, x));//调整红黑树 并重新定位根节点位置
            return null;
        }
    }
}
```

根据以上代码分析：

**1.寻找插入的位置**

首先获取插入红黑树的根节点，遍历根节点。在遍历的过程中根据hash值判断插入的位置，如果可以判断hash值的大小(要么大于或者小于)，那么就确认插入的节点在当前节点的左右边；hash值不等,紧接着判断key时候存在，存在即返回当前节点；hash值相等，没有相等key，判断key是否实现了Comparable，对key进行比较。如果还是无法比较大小，就以当前节点往下面查找。无法找到就通过tieBreakOrder判断dir的值。

**2.插入节点，移动链表**

判断插入节点的位置时候存在节点(左或右节点是否为空)。存在就继续递归；不存在就将x插入到当前位置的左(右)边。改变对应的双向链表，将插入节点x放在当前节点p的后面，如果p下面还有一个节点(对应链表中的)，那么就将p的下一个节点xpn放到插入节点x的后面。

![](\assets\theeNodeInsert.png)

插入节点后需要通过balanceInsertion调整红黑树的平衡，以及moveRootToFront方法重新定位根节点，以确保红黑树中根节点是双向链表的头，并且是table[i]位置(挂在table下面的root节点)。

### 2.1.3.4 table下面是链表，遍历链表插入节点

```java
for (int binCount = 0; ; ++binCount) {//遍历链表
    if ((e = p.next) == null) { //遍历到链表的尾节点
        p.next = newNode(hash, key, value, null); //新建一个链表节点(可以看出HashMap链表是尾插的)
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st 判断链表长度是否大于8(binCount是从0开始)
            treeifyBin(tab, hash);//如果循环了达到了红黑树的阈值，也就是说这里的链表长度大于8，然后就把这个结构树形化。
        break;
    }
    if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
        break;//在循环的过程中遇到了key值相同的节点，跳出循环
    p = e;//和前面for中的p.next对应，进行循环遍历链表。
}
```

从以上方法可以看出，链表是以尾插的方式插入的。在插入链表的节点后如果**链表的长度>=8**,那么就通过**treeifyBin**方法将链表树化。

```java
final void treeifyBin(Node<K, V>[] tab, int hash) {
    int n, index;
    Node<K, V> e;//如果table为空 或者 table长度 < 最小树容量(64)   就进行扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();//扩容
    else if ((e = tab[index = (n - 1) & hash]) != null) {//判断hash对应table的位置是否为空
        TreeNode<K, V> hd = null, tl = null;//双向链表的头尾节点
        do {//循环遍历链表 转换为双向链表
            TreeNode<K, V> p = replacementTreeNode(e, null);//将Node节点变换为TreeNode节点
            if (tl == null)//确认双向链表的头
                hd = p;
            else {//往双向链表尾部插入节点
                p.prev = tl;
                tl.next = p;
            }
            tl = p;//tl指向尾节点
        } while ((e = e.next) != null);//转换成双向链表
        if ((tab[index] = hd) != null)//头部为空就树化
            hd.treeify(tab);//将链表树化
    }
}
```

在将双向链表，转变为红黑树的过程中。首先判断 table是否为空或者table长度是否 < MIN_TREEIFY_CAPACITY,如果满足就扩容。否则就遍历链表，将**链表转换为双向链表**，然后通过**treeify**方法树化。

### 2.1.3.5  其他操作

```java
{//....省略   
    if (e != null) { // existing mapping for key   如果e!=null  说明存在相同的key
        V oldValue = e.value;//记录下原来的key对应的value
        if (!onlyIfAbsent || oldValue == null)//当onlyIfAbsent(putVal默认为fasle)为false或者旧值为空 
            e.value = value;//替换新的value并返回旧的value
        afterNodeAccess(e);//HashMap中暂无实现
        return oldValue;//返回旧值，这样就处理掉了前面的当key值相同时的情况了。
    }
}
++modCount;// 每次容器发生变化记录
if (++size > threshold)// 实际大小大于阈值则扩容
    resize();//如果当前HashMap的容量超过threshold则进行扩容
afterNodeInsertion(evict);//HashMap中暂无实现
return null;
```

判断插入节点是否有重复的key对应的节点为e，如果存在就返回之前key对应的val，并替换新的value。modCount记录HashMap容器中数据改变次数，改变后会++

### 2.2 查询

查询会相对简单，要么在红黑树查找，要么在链表上查找。

```java
final Node<K, V> getNode(int hash, Object key) {
    Node<K, V>[] tab;
    Node<K, V> first, e;//first table上的第一个节点 e最后查询的节点
    int n;
    K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {//如果当前table没有数据的话返回Null
        if (first.hash == hash && // always check first node    //根据当前传入的hash值以及参数key获取一个节点即为first,如果匹配的话返回对应的value值
                ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {//如果参数与first的值不匹配的话
            if (first instanceof TreeNode) //判断是否是红黑树，如果是红黑树的话先判断first是否还有父节点，然后从根节点循环查询是否有对应的值
                return ((TreeNode<K, V>) first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))//如果是链表的话循环拿出数据
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

上述代码，首先判断，table不为空，并且length>0,然后取出table上的Node(first)；如果table上的节点就是查询的key直接返回；否则判断Node节点是否是TreeNode(红黑树)，是就通过**getTreeNode()**在红黑树中查找，不然就遍历链表查询对应的key。

**getTreeNode()方法**中通过**root()方法**获取根节点然后调用**find()方法**查找。root()方法比较简单，就是往上找parent为空的节点，也就是root节点。来看看find()方法。

```java
/**
 * Finds the node starting at root p with the given hash and key.
 * The kc argument caches comparableClassFor(key) upon first use
 * comparing keys.
 * @param h hash值
 * @param k key
 * @param kc k实现comparable接口的class
 * @return
 */
final TreeNode<K, V> find(int h, Object k, Class<?> kc) {
    TreeNode<K, V> p = this;
    do {//循环查找
        int ph, dir;//ph 父节点的hash值   dir 判断后面查找的位置(左或右)
        K pk;//父节点的key
        TreeNode<K, V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h)
            p = pl;//根据p的左节点继续查找
        else if (ph < h)
            p = pr;//根据p的右节点继续查找
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;//找到相等的key并返回 结束循环
        else if (pl == null)//p的左节点为空  下面的代码都是无法根据key判断hash值的大小才执行的
            p = pr;//p的左节点为空 就往右边查找
        else if (pr == null)
            p = pl;//p的右节点为空 就往左边查找
        else if ((kc != null ||
                (kc = comparableClassFor(k)) != null) &&//k实现comparable接口的class
                (dir = compareComparables(kc, k, pk)) != 0) //根据class判断大小
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)//根据p的右节点继续递归查找
            return q;
        else
            p = pl;
    } while (p != null);
    return null;//没找到 返回null
}
```

在查找的时候，根据key的hash值判断往树的左还是右节点查找。如果hash无法判断大小，就根据判断key是否实现comparable接口，如果实现，通过compareTo比较两个key的大小。还是无法判断就根据p的右节点递归查询，否则继续根据p的左节点继续循环。

### 2.3 删除

删除主要通过**removeNode()方法**操作，我们以**remove()**操作为例.

```java
public V remove(Object key) {//根据key删除
    Node<K, V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}
```

### 2.3.1 removeNode 删除节点

**注意以上的removeNode方法最后两个参数分别是:false,true**

```java
/**
 *   
 *
 * @param hash       hash for key
 * @param key        the key
 * @param value      如果matchValue则匹配的值，否则忽略
 * @param matchValue 如果为true，则仅在值相等时删除 remove()==>false
 * @param movable    如果为false，则在删除时不要移动其他节点 remove()==>true
 * @return the node, or null if none
 */
final Node<K, V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K, V>[] tab;//table
    Node<K, V> p;//待删除的节点
    int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {//如果table下面不为空(存在链表或者红黑树)
        Node<K, V> node = null, e;
        K k;
        V v;
        if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;//table对应的就是删除的对象
        else if ((e = p.next) != null) {//p后面还有节点
            if (p instanceof TreeNode)//p下面是红黑树
                node = ((TreeNode<K, V>) p).getTreeNode(hash, key);//红黑树查找
            else {
                do {//遍历链表查询待删除的节点
                    if (e.hash == hash &&
                            ((k = e.key) == key ||
                                    (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)//删除红黑树
                ((TreeNode<K, V>) node).removeTreeNode(this, tab, movable);
            else if (node == p)//删除table[i]  node的下一个节点上去
                tab[index] = node.next;
            else//删除p节点
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

以上代码其实首先是查找待删除的节点,然后再判断待删除的节点是在链表上,还是红黑树上.如果再红黑树上调用

removeTreeNode()方法删除红黑树中的节点,如果再链表上直接删除元素,改变链表的next即可.

### 2.3.2 removeTreeNode 删除树节点

```java
/**
 * 删除此调用之前必须存在的给定节点。
 * 这比典型的红黑删除代码更为混乱，因为我们无法将内部节点的内容与由在遍历期间可独立访问的“下一个”指针固定的叶子后继对象交换。
 * 因此，我们交换树链接。如果当前树的节点似乎太少，则将垃圾箱转换回普通垃圾箱。 
 * （该测试触发2到6个节点之间的某个位置，具体取决于树的结构）。
 * @param map 
 * @param tab
 * @param movable
 */
final void removeTreeNode(HashMap<K, V> map, Node<K, V>[] tab,
                          boolean movable) {
    int n;
    if (tab == null || (n = tab.length) == 0)
        return;
    int index = (n - 1) & hash;
    TreeNode<K, V> first = (TreeNode<K, V>) tab[index], root = first, rl;
    TreeNode<K, V> succ = (TreeNode<K, V>) next, pred = prev;
    //=============第一部分-删除双向链表=============
    if (pred == null)//如果上一个节点为空 说明删除的是头节点
        tab[index] = first = succ;//下一个节点成为头节点
    else
        pred.next = succ;//删除当前节点（修改上一个节点的下一个节点）
    if (succ != null)
        succ.prev = pred;//修改下一个节点的上一个节点
    if (first == null)
        return;//只有一个节点
    if (root.parent != null)//如果父节点不为空， 调用root方法重新获取当前树的根节点
        root = root.root();
    if (root == null || root.right == null ||//根节点为空 根节点的右节点为空
            (rl = root.left) == null || rl.left == null) {//根节点的左节点为空  根节点的左节点的左节点为空
        tab[index] = first.untreeify(map);  // too small    太小 满足以上条件该位置长度<6  链表化
        return;
    }
    //=============第二部分-删除红黑树=============
    TreeNode<K, V> p = this, pl = left, pr = right, replacement;//p 当前节点(待删除的节点) pl 当前节点的左节点 pr 当前节点的右节点  replacement 替代的节点
    if (pl != null && pr != null) {//当前节点左右节点都不为空
        TreeNode<K, V> s = pr, sl;//以当前节点的右节点为根节点 找出最左侧（最小的节点） s
        while ((sl = s.left) != null) // find successor
            s = sl;
        boolean c = s.red;
        s.red = p.red;//把最小节点的颜色改为当前节点的颜色
        p.red = c; // swap colors  交换颜色
        TreeNode<K, V> sr = s.right;//最小节点的右节点
        TreeNode<K, V> pp = p.parent;//当前节点的父节点
        if (s == pr) { // p was s's direct parent   p是s的父节点  把p变成s的右节点
            p.parent = s;//s 变为p的父节点
            s.right = p;//p 成为s的右节点
        } else {//交换p s的位置
            TreeNode<K, V> sp = s.parent;
            if ((p.parent = sp) != null) {//s的父节点不为空   sp成为p的父节点
                if (s == sp.left)//判断s是在父节点的左边还是右边(并把p放在s所在的位置)   s是左边节点
                    sp.left = p;//p 成为sp的左节点
                else
                    sp.right = p;//p 成为 sp 的右节点
            }
            if ((s.right = pr) != null)//如果p的右节点存在 那么就放到s的右节点
                pr.parent = s;
        }
        p.left = null;
        if ((p.right = sr) != null)//如果s的右节点存在 那么就放在p的右边
            sr.parent = p;
        if ((s.left = pl) != null)//如果p的右节点存在 那么就放在s的左节点
            pl.parent = s;
        if ((s.parent = pp) == null)//如果p就是根节点
            root = s;//s变成根节点
        else if (p == pp.left)//p 是父节点的左节点
            pp.left = s;
        else//p 是父节点的右节点
            pp.right = s;//      到此位置 p和s的位置已经交换
        if (sr != null)//s存在右节点
            replacement = sr;//右节点变成替换的值
        else
            replacement = p;//否则 p 成为代替换的值
    } else if (pl != null)//p的左边不等于空
        replacement = pl;//p的左节点变为replacement
    else if (pr != null)//p的右节点不为空
        replacement = pr;
    else
        replacement = p;//p没有子节点
    if (replacement != p) {//p或者s存在子节点
        TreeNode<K, V> pp = replacement.parent = p.parent;//
        if (pp == null)
            root = replacement;//replacement 变为根节点
        else if (p == pp.left)
            pp.left = replacement;
        else
            pp.right = replacement;
        p.left = p.right = p.parent = null;//如果 replacement != p  p节点已经被干掉了
    }

    TreeNode<K, V> r = p.red ? root : balanceDeletion(root, replacement);//如果p不是红色 需要重新调整树的结构

    if (replacement == p) {  // detach   如果p不存在子节点
        TreeNode<K, V> pp = p.parent;
        p.parent = null;
        if (pp != null) {
            if (p == pp.left)
                pp.left = null;
            else if (p == pp.right)
                pp.right = null;
        }//到此位置p节点被干掉了
    }
    if (movable)//把根节点(root)变成tab的第一个位置
        moveRootToFront(tab, r);
}
```

removeTreeNode方法比较复杂，我们暂且分为两部分分析。**第一部分为删除双向链表的中的节点；第二部分为删除红黑树中的节点。**

**删除双向链表中的节点**

TreeNode中维护的不仅是红黑树，而且还是一条双向链表。因此在我们删除的时候，首先需要删除双向链表中对应的节点。

![](\assets\removeTreeNode.png)

removeTreeNode是TreeNode内部方法，this就是需要删除的节点。在删除双向链表的时候，删除节点的前后节点next、prev指向需要发生变化；判断root节点是否发生变化，改变后就重新获取根节点；双向链表的节点个数如果太少，退化成单向链表。直接结束，就不需要后面的调整红黑树操作了。

**删除红黑树中的节点**

在删除双向链表的时候，只是移除了节点(只改变了next、prev，并没有改变left，right)，对应的红黑树中仍然存在。**红黑树删除任意节点，都可以转变为删除叶子节点。**通过找到替代节点(前驱或后继节点，HashMap中在左右节点存在的情况下，取得是后继节点)，然后交换位置，删除代替节点。最后调整平衡。

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\successor.png)

```java
if (pl != null && pr != null)
```

判断删除的节点p是否是叶子节点，如果p的左右节点都为空说明p就是叶子节点。

对应图上的1、4、6等节点

```java
while ((sl = s.left) != null) // find successor
    s = sl;
```

通过while循环找到后继节点

```java
p.left = null;
```

此时，p与后继节点以及交换过来了，但是p之前left可能存在节点，所以需要将p.letf=null;

```java
{
    //...
     if (sr != null)//s存在右节点
        replacement = sr;//右节点变成替换的值
    //...
}else if (pl != null)//p的左边不等于空
   replacement = pl;//p的左节点变为replacement
else if (pr != null)//p的右节点不为空
   replacement = pr;
else
   replacement = p;
if (replacement != p) {//p或者s存在子节点
    TreeNode<K, V> pp = replacement.parent = p.parent;//
    if (pp == null)
        root = replacement;//replacement 变为根节点
    else if (p == pp.left)
        pp.left = replacement;
    else
        pp.right = replacement;
    p.left = p.right = p.parent = null;//如果 replacement != p  p节点已经被干掉了
}
```

replacement != p 在替代节点不是叶子节点时会成立(对应上图删除9)，这个时候就需要将p和replacement(sr)交换位置，p就变成了叶子节点，然后将p节点删除。先删除节点，再调整平衡。

```java
TreeNode<K, V> r = p.red ? root : balanceDeletion(root, replacement);
```

如果删除的黑色节点需要通过balanceDeletion方法调整红黑树的平衡

if (replacement == p)  p的替代节点就是叶子节点，先调整红黑树平衡在，再删除p节点。

后面就是判断是否需要重新调整根节点位置，remove方法删除的时候是需要的。

### 2.3.2 balanceDeletion 删除调整红黑树平衡

```java
else if ((xpl = xp.left) == x) {//x是父节点的左节点
    if ((xpr = xp.right) != null && xpr.red) {//如果父节点的右节点(兄弟节点)不等于空 并且为红色
        xpr.red = false;
        xp.red = true;
        root = rotateLeft(root, xp);//左旋
        xpr = (xp = x.parent) == null ? null : xp.right;
    }
    if (xpr == null)
        x = xp;
    else {//如果兄弟节点不为空
        TreeNode<K, V> sl = xpr.left, sr = xpr.right;
        if ((sr == null || !sr.red) &&
                (sl == null || !sl.red)) {//如果兄弟节点没有子节点 或者都为黑节点
            xpr.red = true;
            x = xp;//继续往上递归
        } else {
            if (sr == null || !sr.red) {//如果兄弟节点的右节点是黑色
                if (sl != null)//先变色
                    sl.red = false;
                xpr.red = true;
                root = rotateRight(root, xpr);//后右旋
                xpr = (xp = x.parent) == null ?
                        null : xp.right;
            }//先变色
            if (xpr != null) {
                xpr.red = (xp == null) ? false : xp.red;
                if ((sr = xpr.right) != null)
                    sr.red = false;
            }
            if (xp != null) {
                xp.red = false;
                root = rotateLeft(root, xp);//后左旋
            }
            x = root;
        }
    }
}
```

以下根据**删除节点x是父节点的左节点**进行分析

> 1.判断兄弟节点(xpr)是否为红色。如果成立，说明不是真正的兄弟节点(结合2-3-4树看)，根据x的父节点xp左旋，得到真正的兄弟节点
>
> 2.判断兄弟节点是否存在
>
> ​         2.1 如果成立，根据x的父节点xp继续向上循环
>
> ​         2.2 判断兄弟节点的左右节点(sr,sl)是否是空或黑色.如果成立，兄弟节点变红色，根据x的父节点xp继续向上循环
>
> ​                 2.2.1   判断兄弟节点的右节点(sr)是不是黑色。 如果成立，根据兄弟节点右旋 
>
> ​			     2.2.2   兄弟节点如果不为空。兄弟节颜色点变为父节点的颜色，兄弟节点右节点变黑色 
>
>  				2.2.3  父节点如果不为空，就将父节点变为黑色，并并根据父节点左旋

**1.为什么兄弟节点为红色的时候需要右旋?**

需要结合2-3-4树进行分析。

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\del1.png)

如上图所示

红黑树是由2-3-4树转变而来，左侧的红黑树可以转换为右侧的2-3-4树。当我们删除x节点的时候，首先需要找兄弟节点借，当兄弟节点为红色的时候，显然不是真正的兄弟节点(转2-3-4树时，红节点要向上合并)。这个就需要根据x的父节点左旋得到真正的兄弟节点。

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\del2.png)

**2.兄弟节点的右节点是黑色为什么需要先右旋？**

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\del3.png)

这个也需要结合2-3-4树，进行分析。如上图所示，当兄弟节点的右节点为黑色，左边节点一定为红色(上面判断对兄弟节点的左右节点进行判断过)。如果直接根据x的父节点左旋，那么红色节点会脱钩，会影响2-3-4树中的平衡(红色节点向上合并)。那么就需要根据兄弟节点进行右旋，变色。最后进行左旋的时候就不会影响2-3-4树的平衡了。

