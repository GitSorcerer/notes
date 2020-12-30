# HashMap1.8源码解析

## 1.介绍

之前写了一篇[HashMap 1.7源码解析](https://blog.csdn.net/chianz632/article/details/109234509)，本片将对HashMap1.8进行分析。后续再写一篇1.7和1.8中HashMap的区别,之前1.7中出现的一些代码我就不赘述了。**HashMap1.8是由：数组(Node)+链表(Node)+红黑树(TreeNode)** 组成，而且**TreeNode继承与Node** ，如下图所示。

![](\assets\HashMap1.8.png)

## 2.源码解析

​        一般我们在使用**HashMap**的时候，最好给一个默认的初始化容量，也就是调的**HashMap(int initialCapacity)**构造方法。在我们给出具体容量的时候就可以防止HashMap自动扩容，以免影响效率。

### 2.1 插入

#### 2.1.1  putVal 源码

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

#### 2.1.2 介绍

首先我们需要知道HashMap在put的时候的逻辑，然后逐行分解。

![](\assets\put-xmd.png)

#### 2.1.3 核心方法

##### 2.1.3.1 resize() 扩容

**resize()**可以说是HashMap最重要的方法之一，**初始化**以及**size>threshold(阈值=容量*负载因子)**的时候都会**调用该方法**。首先我们来分析一下这个方法的作用，紧接着贴上代码。

> 1.重新生成容量以及阈值
>
> 2.将在之前table上的数据转移到扩容后的table上 
>
> 1. 转移红黑树
> 2. 转移链表

###### 2.1.3.1 重新生成容量以及阈值

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

###### 2.1.3.2 数据转移

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

###### 2.1.3.3  hash值计算下标原理

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

###### 2.1.3.4 链表扩容

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

###### 2.1.3.5 红黑树扩容

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

###### 2.1.3.6 红黑树 treeify(Node<K, V>[] tab)树化

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

###### 2.1.3.7 comparableClassFor、compareComparables、tieBreakOrder 判断插入位置

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

###### 2.1.3.8 balanceInsertion 红黑树插入平衡

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