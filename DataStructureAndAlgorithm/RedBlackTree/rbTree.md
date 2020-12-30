# 1.红黑树增删查相关操作

## 1.介绍

​      由于**AVL**树虽然查询快，但是插入删除的时候需要不断的操作来维持自身平衡，导致性能下降。而**红黑树（R-B Tree）**是**黑色平衡**的，在操作的时候只需维持黑色平衡即可(当然还有其他的特性)。红黑树是由**2-3-4树**变换而来，如果需要了解红黑树详细的操作流程，需要对2-3-4树有一定的了解。红黑树在**JDK1.8**中的**HashMap**、**ConcurrentHashMap**、**TreeMap**等，以及**Linux内核.**.....

## 2.特性

> **1.根节点为黑色。**
>
> **2.所有节点非黑即红。**
>
> **3.红色节点的子节点一定是黑色节点(或为空节点，空节点也为黑色节点)。**
>
> **4.从根节点到每一个子节点黑色数量是相等的（黑色平衡）。**

## 3.操作

### 1.基础代码

```java
public class RedBlackTree<K extends Comparable<K>, V> {
    private RBNode<K, V> root;//根节点
    
    //红黑树结构
    static class RBNode<K, V> {
        private RBNode<K, V> left;//左节点
        private RBNode<K, V> right;//右节点
        private RBNode<K, V> parent;//父节点
        private boolean red = true;//颜色  默认为红色
        private K kay;//key
        private V val;//value
        //省略其他的get set 以及默认构造方法   
        public RBNode(K key, V val) {
            this.key = key;
            this.val = val == null ? (V) key : val;
        } 
     }
}
```

**RedBlackTree**中**K**继承**Comparable**接口，用于**比较K值的大小**，判断值插入的位置是**左节点**还是**右节点**。每个节点**默认的颜色是红色**，在有参构造里面如果val为空，插入的val就是key

### 2.旋转

#### 1.说明

想要写出左旋右旋的代码，就需要先了解旋转的时候会发生变化的节点。 在旋转的过程发生的变化(一下以**左旋**为准，右旋变换左右即可)

> 三个节点的父节点发生改变：旋转点 、旋转点的右节点 、旋转点的右节点左子节点 
>
> 三个节点的子节点发生变化：旋转点 、旋转点的右节点 、旋转点的父节点

根据以上变化逐一判断即可

#### 2.左旋

![left](\assets\left.gif)

**左旋：以某个(h)节点左旋，旋转点(h)右节点的左子节点变为旋转点的右节点，旋转点之前的右节点变为父节点。**

```java

/**
 * 左旋
 *             hp                        hp
 *            /                         /
 *           h                        hr
 *         /  \                     /  \
 *        hl   hr                  h    hlr
 *             / \                / \
 *           hll hlr            hl  hll
 *
 * @param h 旋转点
 */
private void rotateLeft(RBNode<K, V> h) {
    if (h != null) {
        // pr h节点的右节点
        RBNode<K, V> hr = h.right;
        //h右节点的左节点变为h的右节点
        h.right = hr.left;
        if (hr.left != null) {//判断h的右节点是否存在左节点  存在就把它作为
            hr.left.parent = h;
        }
        //h的父节点 变为 hr的父节点
        hr.parent = h.parent;
        RBNode<K, V> hp;
        if ((hp = h.parent) != null) {//如果h就是根节点就会为空  否则将hr挂在hp下面
            if (h == pp.left) {
                hp.left = hr;
            } else {
                hp.right = hr;
            }
        } else {
            root = hr;
        }
        //h之前的右节点变为h的父节点
        h.parent = hr;
        hr.left = h;
    }
}
```



#### 3.右旋

![right](\assets\right.gif)

**右旋：以某个点（h）旋转，旋转点(h)左节点的右子节点变为旋转点的左节点，旋转点之前的左节点变为父节点**

```java

/**
 * 右旋
 * <p>
 *            hp                      hp
 *            /                       /
 *           h                       hl
 *         /  \                    /   \
 *        hl   hr               hpl     h
 *        / \                          / \
 *       hll hlr                      hlr  hr
 *
 * @param h 旋转点
 */
private void rotateRight(RBNode<K, V> h) {
    if (h != null) {
        //hl  h的左节点
        RBNode<K, V> hl = h.left;
        //首先将h左孩子节点的右节点挂到h的左节点上
        h.left = hl.right;
        //判断hl的右节点是否存在
        if (hl.right != null) {//存在的话 将hlr变成p的左节点
            hl.right.parent = h;
        }
        //h的父节点变为hl的父节点
        hl.parent = h.parent;
        if (h.parent != null) {
            //h是父亲的左节点
            if (h == h.parent.left) {
                h.parent.left = hl;
            } else {
                h.parent.right = hl;
            }
        } else {
            root = hl;
        }
        //h变为左节点变为h的父节点
        hl.right = h;
        h.parent = hl;//将hl换成h的父节点
    }
}

```



### 3.新增

描述：当我们在插入节点的时候，首先**根据大小来找到插入节点对应的位置**，然后再插入数据。在插入节点的时候**默认是红色**，如果插入节点的**父节点是红色**这个时候就违反了红黑色的规则(第三条)，这个时候就需要根据旋转以及变色来维持平衡。

#### 1.寻找插入节点的位置

```java
/**
 * 插入节点
 *
 * @param key
 * @param val
 * @return
 */
public RBNode<K, V> put(K key, V val) {
    if (key == null) {
        return null;
    }
    //根节点
    RBNode<K, V> _root = this.root;
    if (_root == null) {
        this.root = new RBNode<K, V>(key, val);
        this.root.red = false;//根节点变黑
        return null;
    }
    //插入节点的父节点
    RBNode<K, V> parent;
    //循环找到插入节点的父节点
    do {
        parent = _root;
        if (key.compareTo(_root.key) < 0) {//key < _root.key  往左边找
            _root = _root.left;//根据找到节点的左节点继续循环
        } else if (key.compareTo(_root.key) > 0) {//key > _root.key   往右边找
            _root = _root.right;//根据找到节点的右节点继续循环
        } else {//相等的话 就换成新的val值 并返回
            _root.val = val;
            return _root;
        }
    } while (_root != null);//左节点 或者 右节点没有节点了

    //创建新节点
    RBNode<K, V> child = new RBNode<>(key, val);
    //插到parent下面
    child.parent = parent;
    if (key.compareTo(parent.key) < 0) {//判断插入的是parent 的左边还是有右边
        parent.left = child;
    } else {
        parent.right = child;
    }

    //根据插入节点调整平衡
    balanceInsertion(child);

    return null;
}
```

插入数据的时候首先我们需要判断根节点是否为空，**为空**插入的节点就是**根节点**。否则就**根据key(key实现了Comparable接口)的大小来找到插入节点的父节点**，然后将节点插入父节点的下面。最后根据插入的节点**调整平衡**。

#### 2.根据插入的节点调整平衡

插入的**节点默认是红色**的，.如果插入节点的父节点是红色，这个时候及违反了红黑树的特性。

**PS:以下我们根据插入节点的父节点是爷爷节点的左节点分析,另一种情况(插入节点的父节点是爷爷节点的右节点)操作完全相反.只要掌握其中一种情况即可.**

##### 1.插入节点(P)的父节点和叔叔节点都是红色

![](\assets\1607759199.png)

> 1.**父节点和叔叔节点变黑,爷爷节点变红**。然后将**P节点指向爷爷节点**，继续**往上递归**
>
> 2.p节点为**根节点**，直接将p节点**变红**

##### 2.插入节点(P)的父节是红色,并且插入节点是父节点的左节点

![](\assets\1607760457.png)

> 首先**父节点变黑**,**爷爷节点变红**, 然后根据**p的父节点右旋**.

##### 3.插入节点(P)的父节是红色,并且插入节点是父节点的右节点

![](\assets\1607761166.png)

> 直接根据p的父节点左旋，转换为第二种情况,然后通过第二种情况处理

#### 3.具体实现

```java
/**
 * 插入数据后平衡操作
 * <p>
 * 分析： 插入的节点默认为红色，如果父节点为红色 那么就不满足红黑树的条件(红色节点有两个黑子节点)。所以当父节点为红色的时候需要调整，主要分以下三种情况(以p的父节点是爷爷的左子节点为例)
 * 一.p的父节点和叔叔节点都是红的
 * 解决：将p的父节点和叔叔节点变为黑色，p的爷爷节点变为红色。但p的爷爷节点的父节点可能也是红的那就又不满足红黑树的原则了，此时就需要把p的爷爷节点当作p节点继续往上递归了
 * 二.p的叔叔节点不是红色的(或者没有叔叔节点)，并且p是父亲的左节点
 * 解决：以p的爷爷节点为支点向右旋
 * 三.p的叔叔节点不是红色的(或者没有叔叔节点)，并且p是父亲的右节点
 * 解决：先以p的父亲节点左旋，变成了情况二了，以p的爷爷节点为支点向右旋
 *
 * @param p 插入的节点
 */
private void balanceInsertion(RBNode<K, V> p) {
    //p不是根节点 并且p的父节点是红色的时候需要调整
    while (p != null && p != root && p.parent.red) {
        //父节点在左边
        if (p.parent == p.parent.parent.left) {
            //叔叔节点
            RBNode<K, V> u;
            //如果叔叔节点是红色的（情况一）
            if ((u = p.parent.parent.right) != null && u.red) {
                //父节点和叔叔节点变黑  爷爷节点变红  然后以爷爷节点继续往上递归
                u.red = false;
                p.parent.red = false;
                p.parent.parent.red = true;
                //以爷爷节点继续往上递归
                p = p.parent.parent;
            } else {
                //p是父亲的右节点（情况三）
                if (p == p.parent.right) {
                    p = p.parent;
                    //以p的父亲节点左旋
                    rotateLeft(p);
                }
                //爷爷变红色
                p.parent.parent.red = true;
                //父亲边黑色
                p.parent.red = false;
                //以p的爷爷节点右旋
                rotateRight(p.parent.parent);

            }
        } else {//剩下的情况刚好与上相反 左改右 右改左
            RBNode<K, V> u;
            if ((u = p.parent.parent.left) != null && u.red) {
                u.red = false;
                p.parent.red = false;
                p.parent.parent.red = true;
                p = p.parent.parent;
            } else {
                if (p == p.parent.left) {
                    p = p.parent;
                    rotateRight(p);
                }
                p.parent.parent.red = true;
                p.parent.red = false;
                rotateLeft(p.parent.parent);

            }
        }
    }
    //根节点变黑色 这一布很重要(第一中情况后 就需要将根节点变黑)
    root.red = false;
}
```

### 4.查询

根据某个key查找树中对应的元素

```java
/**
 * 根据key找到对应的节点
 *
 * @param key 寻找的key
 * @return 找到的节点值
 */
private RBNode<K, V> getNodeByKey(K key) {
    if (key != null) {
        RBNode<K, V> temp = root;
        do {
            //根据大小往叶节点循环查找
            if (key.compareTo(temp.key) < 0) {
                temp = temp.left;
            } else if (key.compareTo(temp.key) > 0) {
                temp = temp.right;
            } else {//找到对应的值
                return temp;
            }
        } while (temp != null);
    }
    return null;
}
```

### 5.删除

描述:在删除节点的时候,首先需要**查找对应的节点**.然后判断该节点**是否是根节点**,如果是**根节点直接删除**,否则找到对应的**前驱**或者**后继**节点,**交换位置或者值**.然后判断前驱或后继节点**是否有子节点**,如果有,**子节点上去**(替换前驱或后继节点).然后删除最后替换的节点(叶子节点).删除后,如果**删除的是黑色**,则需要**调整**.

### 1.找替换节点

红黑树里面,整棵树**从左往右**肯定是**从小到大**的.所以想找到某个节点**前驱(小于本身的最大节点)**,肯定是从**左节点最右节点**.某个节点**后继(大于本身的最小节点)**,肯定是从**右节点最左节点**.如下图所示

![](F:\GoodGoodStudent\notebook\Typora\DataStructureAndAlgorithm\RedBlackTree\assets\1607765243.png)

#### 1.前驱节点

**前驱节点:小于当前节点的最大值对应的节点**

```java
/**
 * 找到前驱节点(小于当前节点的最大值)
 * 	
 * @param node
 * @return
 */
private RBNode<K, V> predecessor(RBNode<K, V> node) {
    if (node != null) {
        //判断node左节点是否存在
        if (node.left != null) {
            RBNode<K, V> plr;
            if ((plr = node.left.right) != null) {
                //找到左节点的最右节点
                while (plr.right != null) {
                    plr = plr.right;
                }
                return plr;
            }
            return node.left;
        } else {//如果不存在  在删除节点是不会走这一步的  p.left != null && p.right != null
            RBNode<K, V> child = node;
            RBNode<K, V> p = node.parent;
            while (p != null && child == p.left) {
                child = p;
                p = p.parent;
            }
            return p;
        }
    }
    return null;
}
```



#### 2.后继节点

**前驱节点:大于当前节点的最小值对应的节点**

```java
/**
 * 找到后继节点(大于当前节点的最小值)
 *
 * @param node
 * @return Successor
 */
private RBNode<K, V> successor(RBNode<K, V> node) {
    if (node != null) {
         //判断node右节点是否存在
        if (node.right != null) {
            RBNode<K, V> pll;
            if ((pll = node.right.left) != null) {
                //找到最左节点
                while (pll.left != null) {
                    pll = pll.left;
                }
                return pll;
            }
            return node.right;
        } else {
            RBNode<K, V> child = node;
            RBNode<K, V> p = node.parent;
            while (p != null && child == p.right) {
                child = p;
                p = p.parent;
            }
            return p;
        }
    }
    return null;
}
```

### 2.删除节点

​        **删除的时候，找到对应的前驱或者后继节点后，判断其是否有子节点。如果有，且可能只有一个子节点，将前驱或者后继节点和子节点替换(转变为删除子节点)。没有的话就说明删除的就是子节点或者根节点。最后删除子节点如果为黑就重新调整平衡。**

> 1. 没有子节点，就说明删除的就是子节点，根据删除的节点调整平衡后删除即可。
> 2. 如果是根节点，那么直接将root制空。
> 3. 如果存在一个子节点，将对应的子节点与前驱(后继)节点替换，然后删除替换后的子节点，调整平衡。

```java
/**
 * 根据key移除某个节点
 *
 * @param key
 */
public void remove(K key) {
    //根据key查找对应的值
    RBNode<K, V> p = getNodeByKey(key);

    if (p == null) return;
    //如果p的左右节点都存在
    if (p.left != null && p.right != null) {
        //找到前驱节点
        RBNode<K, V> node = predecessor(p);
        //将node值赋给p
        p.key = node.key;
        p.val = node.val;
        p = node;
    }
    //p节点的替换节点（也就是p的子节点，如果存在只可能有一个）
    RBNode<K, V> replace = p.left != null ? p.left : p.right;
    if (replace != null) {
        RBNode<K, V> pp;
        //如果p节点的父节点是根节点 那么替换节点变为根节点
        if ((pp = p.parent) == null) {
            root = replace;
        } else if (p == pp.left) {
            pp.left = replace;
        } else {
            pp.right = replace;
        }
        replace.parent = pp;

        //删除p节点
        p.left = p.right = p.parent = null;
        //删除的是黑色节点需要调整
        if (!p.red) {
            //对replace 进行平衡处理
            balanceDeletion(replace);
        }
    } else if (p.parent == null) {
        root = null;
    } else {//p就是根节点，这个时候需要先调整平衡 再做删除
        if (!p.red) {
            balanceDeletion(p);
        }
        //删除p节点
        if (p.parent.left == p) {
            p.parent.left = null;
        } else {
            p.parent.right = null;
        }
        p.parent = null;
    }
}
```

### 3.调整平衡

删除节点的时候，需要和2-3-4树对照才能知道其中的所以然。由上可知在删除节点的时候，**最终都转换为删除子节点**。如果删除的子节点颜色是黑色那么就需要调整。

> 1.找兄弟节点接(兄弟节点的子节点都为空，或者都为黑色节点，借出去就会影响平衡)，兄弟节点没得借，那么就直接将兄弟节点变红以父节点继续往上找。
>
> 2.兄弟节点有的借...

#### 1.2-3-4树

> 2节点:有一个元素的节点，有两个子节点
>
> 3节点:有两个元素的节点，有三个子节点
>
> 4节点:有三个元素的节点，有四个子节点

![](\assets\2-3-4 tree.png)



首先我们看看红黑树和2-3-4树对应的关系

![](\assets\redTreeAnd2-3-4Tree.png)

依次按照1-10插入10个元素，得到上图的红黑树和2-3-4树(忽略2-3-4的颜色)。从上可以看出**红黑树其实就是2-3-4树演变而来**。同时，如果**红黑树的每个红色节点向上合并也是可以得到2-3-4树**。



#### 2.删除调整平衡

了解了红黑树嘿2-3-4树的基本关系，以下我们还是分析左边的情况(**删除的节点是父节点的左节点**),**假设删除的节点为X节点**。

##### 找到真正的兄弟节点

如果**兄弟节点存在，并且为红色，说明它不是真正的兄弟节点**，红色节点需要往上合并(自己的理解)。

![](\assets\del1.png)

这个时候我们就需要**根据X的父节点左旋**，找到兄弟节点。

![](\assets\del2.png)

##### 1.兄弟节点没得借

> 1.兄弟节点为空
>
> 2.兄弟节点都为黑色

在这两中情况下是没有办法借的，只能**将兄弟节点变为红色**X指向父节点,以X**父节点继续向上循环**判断。

##### 2.兄弟节点有得借

**如果兄弟节点的右节点为黑色**，**需要对兄弟节点进行右旋**，个人理解如下:

1. 兄弟结点为黑结点，说明是真正的兄弟结点。把红黑树转换为2-3-4树，可以看出，兄弟结点为3结点(上黑下红)，属于兄弟结点有得借的情况（可以借出去一个当父结点）。
2. 这时，为了能够让兄弟结点借出去一个结点，假设让父结点左旋，那么就会发生红节点脱钩，使得本该为3结点的兄弟结点强制被拆除，显然这方法不正确的，需要先调整兄弟结点再进行结点外借。
3. 由于兄弟结点是3结点，调整兄弟结点可以先对兄弟结点进行右旋，再变色以维持3结点上黑下红的状态。这时，可以发现，如果对兄弟结点进行结点外借，不会发生红结点脱钩的现象了。
4. 最后对父结点进行左旋，使得兄弟结点的其中一个结点成为新的父结点，这时，兄弟结点成功借出去一个。

![](\assets\del3.png)

**兄弟节点的右节点不为黑色**，根据**父节点左旋**

##### 3.具体实现

```java
/**
 * 删除调整
 *  左旋是为了把x节点凑成一个3节点或者4节点，这样才能删除x，要不然x单独就是个2节点，在234树里是不能删除的
 * @param x 调整的节点
 */
private void balanceDeletion(RBNode<K, V> x) {
    //p不等于根节点并且p是黑色
    while (x != root && !getColor(x)) {
        //x是父亲的左节点
        if (x == getLeft(getParent(x))) {
            //叔叔节点
            RBNode<K, V> xpr;
            //如果兄弟节点是红色,两个子节点是黑色 不能直接借
            if ((xpr = getRight(getParent(x))) != null && getColor(xpr)) {
                //兄弟点变黑色
                setColor(xpr, false);
                //p父节点变红
                setColor(getParent(x), true);
                //围绕x的父节点左旋
                rotateLeft(getParent(x));
                xpr = getRight(getParent(x));
            }
            //判断兄弟节点是否存在两个黑色节点(或者两个子节点都为空)
            if (!getColor(getLeft(xpr)) && !getColor(getRight(xpr))) {
                //将兄弟点变成红色
                setColor(xpr, true);
                //根据p的父节点继续往上递归
                x = getParent(x);
            } else {
                //兄弟节点的右节点如果是黑色
                if (!getColor(getRight(xpr))) {
                    //兄弟节点变红
                    setColor(xpr, true);
                    //兄弟节点的左节点变黑
                    setColor(getLeft(xpr), false);
                    //将兄弟节点右旋
                    rotateRight(xpr);
                    xpr = getRight(getParent(x));
                }
                //兄弟节点颜色变为p父节点颜色
                setColor(xpr, getColor(getParent(x)));
                //p父节点变黑
                setColor(getParent(x), false);
                //兄弟节点的右节点变黑
                setColor(getRight(xpr), false);
                //根据p的父节点节点左旋
                rotateLeft(getParent(x));
                break;
            }
        } else {
            RBNode<K, V> ppr;
            if ((ppr = getLeft(getParent(x))) != null && getColor(ppr)) {
                setColor(ppr, false);
                setColor(getParent(x), true);
                rotateRight(getParent(x));
                ppr = getLeft(getParent(x));
            }
            if (!getColor(getLeft(ppr)) && !getColor(getRight(ppr))) {
                setColor(ppr, true);
                x = getParent(x);
            } else {
                if (!getColor(getLeft(ppr))) {
                    setColor(ppr, true);
                    setColor(getRight(ppr), false);
                    rotateLeft(ppr);
                    ppr = getLeft(getParent(x));
                }
                setColor(ppr, getColor(getParent(x)));
                setColor(getParent(x), false);
                setColor(getLeft(ppr), false);
                rotateRight(getParent(x));
                break;
            }
        }

    }
    //将p节点变为黑色
    setColor(x, false);
}

/**
 * 获取节点的颜色
 *
 * @param node
 * @return 如果节点为空 返回false(黑色) 否则返回节点的颜色
 */
private boolean getColor(RBNode<K, V> node) {
    return node != null && node.red;
}

private RBNode<K, V> getParent(RBNode<K, V> node) {
    return node != null ? node.parent : null;
}

private RBNode<K, V> getLeft(RBNode<K, V> node) {
    return node != null ? node.left : null;
}

private RBNode<K, V> getRight(RBNode<K, V> node) {
    return node != null ? node.right : null;
}

private void setColor(RBNode<K, V> node, boolean color) {
    if (node != null) {
        node.red = color;
    }
}
```

## 4.总结

**在红黑树操作里面，主要是当黑色节点发生变化的时候，围绕红黑树的特性进行逐一分析处理，最终的结果是维持平衡。而在这些判断中，有很多都是相似的只是操作的方向不同。在本文中我们主要以操作节点在左边这种情况，进行分析的。插入的时候判断操作节点的父节点是左子节点，删除的时候判断操作节点是左子节点。了解其中的一种即可。**

源码地址:https://gitee.com/gaohwh/jdk-sources.git

