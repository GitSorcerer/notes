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
            hr.parent = h.parent;

            RBNode<K, V> hp;
            if ((hp = h.parent) != null) {
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

描述：在插入节点的时候**默认是红色**，如果插入节点的**父节点是红色**这个时候就违反了红黑色的规则(第三条)，这个时候就需要根据旋转以及变色来维持平衡。

### 4.查询

### 5.删除