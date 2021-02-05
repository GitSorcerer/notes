# 从ReentrantLock对AQS源码分析

## 介绍

AQS全称:AbstractQueuedSynchronizer，从字面意思翻译为 **抽象同步队列** 。AQS的是在Java层面提供线程的同步机制。AbstractQueuedSynchronizer是一个抽象类，它里面提供了一些锁的基础实现，并且还定义了一些抽象方法为子类实现提供了模板。线程同步工具CountDownLatch、CyclicBarrier、Semaphore等是基于AQS的实现。

## 基本概念

在AQS中维护者一个CLH(Craig, Landin, and Hagersten)同步队列，并且是一个FIFO先进先出的双向队列。

> <p>等待队列是“ CLH”（Craig，Landin和Hagersten）锁队列的变体。 CLH锁通常用于自旋锁。相反，我们将它们用于阻塞同步器，但使用相同的基本策略，即将有关线程的某些控制信息保存在其节点的前身中。每个节点中的“状态”字段将跟踪线程是否应阻塞。节点的前任释放时会发出信号。否则，队列的每个节点都充当一个特定通知样式的监视器，其中包含一个等待线程。虽然状态字段不控制是否授予线程锁等。线程可能会尝试获取它是否在队列中的第一位。但是先行并不能保证成功。它只赋予竞争权。因此，当前发布的竞争者线程可能需要重新等待。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    //维护队列的节点
    static final class Node {
        /**共享*/
        static final Node SHARED = new Node();
        /**独占*/
        static final Node EXCLUSIVE = null;
        /** waitStatus值，指示线程已取消 */
        static final int CANCELLED =  1;//waitStatus值为1时表示该线程节点已释放（超时、中断），已取消的节点不会再阻塞。
        /** waitStatus值，指示后续线程需要unpack */
        static final int SIGNAL    = -1;//waitStatus为-1时表示该线程的后续线程需要阻塞，即只要前置节点释放锁，就会通知标识为 SIGNAL 状态的后续节点的线程
        /** waitStatus值，指示线程正在等待条件 */
        static final int CONDITION = -2;//waitStatus为-2时，表示该线程在condition队列中阻塞（Condition有使用）
        /** waitStatus值，指示下一个acquireShared应该无条件传播*/
        static final int PROPAGATE = -3;//waitStatus为-3时，表示该线程以及后续线程进行无条件传播（CountDownLatch中有使用）共享模式下， PROPAGATE 状态的线程处于可运行状态

        /**状态   对应上面的四个常量*/
        volatile int waitStatus;

        /**上一个节点*/
        volatile Node prev;

        /**下一个节点*/
        volatile Node next;

        /**当前节点对应的线程*/
        volatile Thread thread;

        /**链接到等待条件的下一个节点，或者链接到特殊值SHARED。由于条件队列仅在以独占模式保存时才被访问，因此我们只 需要一个简单的链接队列即可在节点等待条件时保存节点。然后将它们转移到队列中以重新获取。由于条件只能是 互斥的，因此我们使用特殊值来表示共享模式来保存字段。条件队列使用到*/
        Node nextWaiter;
        //...
    }
    /**队列的头节点*/
    private transient volatile Node head;

    /**队列的尾节点*/
    private transient volatile Node tail;

    /**同步状态 0  未加锁 >=1 以加锁（>1表示重入） -1 禁止中断*/
    private volatile int state;
    //...
}
```

在 **AbstractQueuedSynchronizer** 中存在内部类 **Node** ，就是 **CLH** 队列，里面定义了线程的某些控制信息。

## 源码解析

以下对ReentrantLock加锁、解锁进行分析

```java
//Sync是ReentrantLock抽象内部类 继承了AbstractQueuedSynchronizer
private final Sync sync;
//默认构造方法
public ReentrantLock() {
	sync = new NonfairSync();
}
//加锁
public void lock() {
    sync.lock();
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    /**定义加锁的抽象方法 有实现类去实现*/
	abstract void lock();
    //...其他方法省略
}
//非公平锁
static final class NonfairSync extends Sync {
    //加锁 该方法是对Sync中的实现
    final void lock() {
        if (compareAndSetState(0, 1))//修改状态
            setExclusiveOwnerThread(Thread.currentThread());//修改成功 当前线程就占有锁
        else
            acquire(1);//
    }
    //尝试加锁 该方法是对AbstractQueuedSynchronizer中的实现
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
//...其他方法省略
```

当我们创建ReentrantLock时，默认使用的是 **NonfairSync非公平锁** (后面介绍公平锁与非公平锁的区别)。NonfairSync是Sync子类，Sync又是AbstractQueuedSynchronizer子类。AbstractQueuedSynchronizer和Sync提供一些基础方法的实现，也对部分方法(抽象方法)提供模板，共子类实现。

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\AQS1.png)

### lock 加锁

根据上述代码以及图片可知当我们加锁的时，首先调用的是NonfairSync(非公平锁)中的lock方法

```java
//java.util.concurrent.locks.ReentrantLock.NonfairSync#lock
final void lock() {
    if (compareAndSetState(0, 1))//修改状态
        setExclusiveOwnerThread(Thread.currentThread());//修改成功 当前线程占有锁
    else
        acquire(1);//继续尝试加锁，不成功就入队
}
```

- compareAndSetState(0, 1)  使用AQS尝试将0修改为1,0表示没有线程持有锁，1表示有线程持有锁。修改成功，通过setExclusiveOwnerThread 将当前线程变为持有锁的线程
- 修改状态失败后就通过acquire方法继续处理，acquire是AQS内部方法

**acquire 获取锁**

```java
//java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();//如果打断返回为true  重新产生中断(导致不可中断)
}
```

**tryAcquire 尝试获取锁**

tryAcquire在NonfairSync中实现

```java
//java.util.concurrent.locks.ReentrantLock.NonfairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);//委托给Sync中nonfairTryAcquire进行处理
}
//java.util.concurrent.locks.ReentrantLock.Sync#nonfairTryAcquire
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();//当前线程
    int c = getState();
    if (c == 0) {//0 表示内有对象持有锁
        if (compareAndSetState(0, acquires)) {//尝试修改状态
            setExclusiveOwnerThread(current);//设置持有锁的线程
            return true;//获取到锁直接返回
        }
    }
    else if (current == getExclusiveOwnerThread()) {//判断当前线程是否是持有锁的线程 线程重入判断
        int nextc = c + acquires;//之前状态值+acquires(1)
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);//设置状态 还是当前线程持有锁 所以可以直接修改
        return true;
    }
    return false;
}
```

nonfairTryAcquire会再次尝试修改状态获取锁，并且会对 **可重入锁** 进行处理(状态值变为之前的值+传入的参数值(1))。

**在acquireQueued之前我们先看看addWaiter方法**

```java
/**
 * Creates and enqueues node for current thread and given mode.
 * 为当前线程和给定模式创建并排队节点。
 *
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * Node.EXCLUSIVE用于独占，Node.SHARED用于共享
 * @return the new node 返回创建的新节点
 */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#addWaiter
private Node addWaiter(Node mode) {//Node EXCLUSIVE=null so  mode=null
    Node node = new Node(Thread.currentThread(), mode);//创建node节点  下一个等待节点
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;//尾节点
    if (pred != null) {//尾节点不为空 就把创建的节点变为尾节点
        node.prev = pred;//新建节点的上一个节点指向之前的尾节点
        if (compareAndSetTail(pred, node)) {//使用cas尝试将新节点变为尾节点
            pred.next = node;//之前尾节点的下一个节点指向新创建节点
            return node;//返回新建节点
        }
    }
    enq(node);//如果尾节点为空 或者通过cas设置尾节点失败 就会进入enq方法
    return node;
}


 /**
  * Inserts node into queue, initializing if necessary. See picture above.
  * 将节点插入队列，必要时进行初始化。
  * @param node the node to insert 要插入的节点
  * @return node's predecessor 节点的上一个节点
  */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#enq
private Node enq(final Node node) {
    for (;;) {//死循环相当于自旋
        Node t = tail;//尾节点
        if (t == null) { // Must initialize   尾节点为空，这时队列为空，必须初始化
            if (compareAndSetHead(new Node()))//使用cas设置队列头节点
                tail = head;//此时头节点也是尾节点
        } else {//尾节点不为空
            node.prev = t;//新建节点的上一个节点指向尾节点
            if (compareAndSetTail(t, node)) {//使用cas将node节点设为位尾节点 修改失败就继续循环
                t.next = node;//之前尾节点的下一个节点指向新建节点
                return t;//返回新建节点的上一个节点 也就是之前的尾节点
            }
        }
    }
}
```

addWaiter方法主要的作用就是将当前线程节点插入到队列的尾部。如果尾节点不为空，就通过cas将新建的节点改为尾节点；如果尾节点为空(队列为空)或修改失败，那么就调用enq方法初始化队列或自旋进行修改，直至修改成功。

**acquireQueued **

```java
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node 竞争线程对应的节点
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireQueued
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//失败标记
    try {
        boolean interrupted = false;//打断标记
        for (;;) {
            final Node p = node.predecessor();//node的前驱节点
            if (p == head && tryAcquire(arg)) {//如果前驱节点就是头节点 那么就尝试获取锁
                setHead(node);//获取锁后 将当前节点设为头节点
                p.next = null; // help GC 将之前的头节点的下一个节点制空 方便GC
                failed = false;//修改标记
                return interrupted;//返回打断标记
            }
            //获取锁失败
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())//park
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

/**
 * Checks and updates status for a node that failed to acquire.
 * Returns true if thread should block. This is the main signal
 * control in all acquire loops.  Requires that pred == node.prev.
 * 检查并更新无法获取的节点的状态。如果线程应阻塞，则返回true。这是所有采集循环中的主要信号控制。要求pred== node.prev。
 *
 * @param pred node's predecessor holding status 节点的前任保持状态
 * @param node the node 当前节点(竞争的线程所对应节点)
 * @return {@code true} if thread should block
 */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#shouldParkAfterFailedAcquire
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /* 该节点已经设置了状态，要求释放以发出信号，以便可以安全地停放。
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /* 前任已取消。跳过前任并指示重试。
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /* waitStatus必须为0或PROPAGATE。表示我们需要一个信号，但不要park。呼叫者将需要重试以确保在park之前无法获取。
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);////将前驱节点的head中的waitStatus改为-1
    }
    return false;
}

/**
 * Convenience method to park and then check if interrupted
 *
 * @return {@code true} if interrupted
 */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#parkAndCheckInterrupt
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//将当前线程park
    return Thread.interrupted();//判断是否被打断 并清除打断标记（下次还可以继续park）
}
```

**acquireQueued** 中首先判断当前线程节点的上一个节点是否是头节点，如果是的话就尝试加锁，否则调用shouldParkAfterFailedAcquire方法修改状态，将上一个节点的waitStatus修改为Node.SIGNAL(-1),表示前一个节点释放锁后，就可以唤醒下一个节点。第一次调用shouldParkAfterFailedAcquire方法，其中ws默认为0，肯定会走else，将上一个节点waitStatus修改为-1，返回fasle。这时会再次尝试加锁（外面是个for循环），如果加锁失败会再次进入shouldParkAfterFailedAcquire方法中，第二次会直接返回true。并且会调用parkAndCheckInterrupt方法，将当前线程park。

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\AQS2.png)

当第一个线程来，尝试将state修改为1，修改成功后就说明持有锁，并将exclusiveOwnerThread指向为第一个线程(此时队列是空的)。后面的线程来的时候，首先会初始化一个空的Node(头节点)，后面竞争的线程插入到空节点尾部。状态为-1，表示可以唤醒后面等待的线程。如果当前线程节点的上一个节点是头节点，就可以尝试获取锁。这个过程就像排队买票一样，假如你是排队的第二个人，是不是随时可能有机会轮到你在，一是你主动去问前面一个人处理完了没(使用tryAcquire尝试获取几次锁失败就park)，二是你被动的等第一个人结束后叫你(unpark)。至于你后面的，只有等你结束后才有资格，所以都park了。



### unlock 解锁

```java
/**
 * Attempts to release this lock.
 * 尝试释放此锁
 *
 * <p>If the current thread is the holder of this lock then the hold
 * count is decremented.  If the hold count is now zero then the lock
 * is released.  If the current thread is not the holder of this
 * lock then {@link IllegalMonitorStateException} is thrown.
 *
 * @throws IllegalMonitorStateException if the current thread does not
 *         hold this lock
 */
//java.util.concurrent.locks.ReentrantLock#unlock
public void unlock() {
    sync.release(1);
}

/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#release
public final boolean release(int arg) {
    if (tryRelease(arg)) {//修改状态 重入则为false 不会走下面 只对state-1
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);//缓存后面节点 h为头节点
        return true;
    }
    return false;//可重入锁 则返回false
}

//解锁 返回是否是重入锁 重入返回 false
//java.util.concurrent.locks.ReentrantLock.Sync#tryRelease
protected final boolean tryRelease(int releases) {//此时releases为1
    int c = getState() - releases;//状态- 1 解决可重入锁的清空 
    if (Thread.currentThread() != getExclusiveOwnerThread())//如果当前线程不是持有锁的线程 抛出异常
        throw new IllegalMonitorStateException();
    boolean free = false;//可重入标记
    if (c == 0) {//不是可重入锁
        free = true;
        setExclusiveOwnerThread(null);//直接将持有锁的线程制空
    }
    setState(c);
    return free;//返回是否可重入
}

/**
 * Wakes up node's successor, if one exists.
 * 唤醒节点的后继节点(下一个节点)（如果存在）。
 *
 * @param node the node
 */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#unparkSuccessor
private void unparkSuccessor(Node node) {//node为头节点
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)//ws为-1  前面通过shouldParkAfterFailedAcquire修改过 表示可以唤醒后面的线程
        compareAndSetWaitStatus(node, ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;//头节点的下一个节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;//找到唤醒的节点
    }
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒头节点后面的节点
}
```

解锁里面操作相对简单一些，修改state的值，其中有对可重入锁解锁的支持。如果是可重入锁，exclusiveOwnerThread持有的线程肯定不能直接制空，需要层层解锁，最后state==0才制空。全都解锁完毕(重入锁)，通过unpark唤醒头节点的下一个节点。

在解锁里面节点是没有发生变化的：唤醒下一个节点后，下一个节点应该变为头节点。之前的头节点应该干掉。

LockSupport.unpark(s.thread)后，s节点对应的线程就会被唤醒，该线程之前是被park了，unpark后继续执行。

```java
////java.util.concurrent.locks.AbstractQueuedSynchronizer#parkAndCheckInterrupt
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();//判断是否被打断 并清除打断标记（下次还可以继续park）
}
```

所以unpark后，之前的线程就会执行Thread.interrupted()这段代码，判断是否被打断，并且清除打断标记。显然没被打断，就继续执行acquireQueued中的for循环，刚好是第二个节点，继续尝试获取锁。

```java
//java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireQueued  
for (;;) {
    final Node p = node.predecessor();//node的前驱节点
    if (p == head && tryAcquire(arg)) {//如果前驱节点就是头节点 那么就尝试获取锁
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
    }
    if (shouldParkAfterFailedAcquire(p, node) &&
        parkAndCheckInterrupt())//park
        interrupted = true;
}
//java.util.concurrent.locks.AbstractQueuedSynchronizer#setHead
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

获取到锁后，就将当前节点改为头节点。之前的头节点的下一个节点制空，之前的头节点没有引用了，GC回收的时候就会直接回收。

### lockInterruptibly 可打断锁

可中断

```java
//java.util.concurrent.locks.ReentrantLock#lockInterruptibly
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
//java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireInterruptibly
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())//判断是否被打断
        throw new InterruptedException();
    if (!tryAcquire(arg))//尝试获取锁
        doAcquireInterruptibly(arg);//打断处理
}

/**
 * Acquires in exclusive interruptible mode.
 * @param arg the acquire argument
 */
//java.util.concurrent.locks.AbstractQueuedSynchronizer#doAcquireInterruptibly
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);//创建节点
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())//如果被打断 就返回true 直接抛出异常
                throw new InterruptedException();//打断后抛出异常
        }
    } finally {
        if (failed)//上面操作异常 就会执行cancelAcquire方法
            cancelAcquire(node);
    }
}
```

不可中断

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();//如果打断返回为true  重新产生中断(导致不可中断)
}
//对于NEW|TERMINATED 线程，中断完全不起作用；对于RUNNABLE或者BLOCKED线程，只会将中断标志位设为true；WAITING线程对中断较为敏感，会抛出异常，同时会将中断标志位清除变为false。java中的中断设置标志位，接着的事情就留给程序员，根据中断这一条件去执行别的逻辑抑或是不管。
static void selfInterrupt() {
    Thread.currentThread().interrupt();//如果线程处于运行状态，不会受影响，会继续执行下去
}
```

可中断锁：当线程被中断时，直接抛出InterruptedException异常，外面捕获进行处理。所以是可中断的。

不可中断：线程中断后，interrupted = true，获取到锁过后，acquire中就会调用selfInterrupt方法。对于RUNNABLE或者BLOCKED线程，只会将中断标志位设为true。所以这是线程会正常运行，不会被打断。

### 公平锁与非公平锁

```java
/**
 * Fair version of tryAcquire.  Don't grant access unless
 * recursive call or no waiters or is first.
 */
//java.util.concurrent.locks.ReentrantLock.FairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&//先检查队列是否又等待的线程
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

//java.util.concurrent.locks.AbstractQueuedSynchronizer#hasQueuedPredecessors
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&//队列中有节点
        ((s = h.next) == null || s.thread != Thread.currentThread());//队列中有第二个节点，并且不是当前线程
}
//java.util.concurrent.locks.ReentrantLock.Sync#nonfairTryAcquire
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //可重入代码省略...
    return false;
}
```

公平锁：来了一个线程首先会判断等待队列中是否会有等待线程，如果有的话，就不用通过cas修改状态了。

非公平锁：来了一个线程就通过compareAndSetState,尝试去修改状态，看能否加锁成功。

## 总结

本文主要对AQS中一些基本原理以及可重入锁、公平锁、非公平锁、是否可打断锁等基本操作进行了分析。在源码中很多操作也是提供了注释，可以根据以上描述，跟踪源码进行逐行分析。

![](F:\GoodGoodStudent\notebook\Typora\JDK源码\assets\mmt.gif)