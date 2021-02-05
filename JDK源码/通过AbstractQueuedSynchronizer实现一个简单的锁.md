# 通过AbstractQueuedSynchronizer实现一个简单的锁

## 介绍

仿照ReentrantLock，实现一个简单的自定义锁，包含加锁、解锁。了解AbstractQueuedSynchronizer中一些基本概念 

## 实现Lock接口

Lock接口中定义了一些基本操作锁的方法

> lock()   获取锁，获取失败就进入阻塞队列
>
> lockInterruptibly()  获取可打断锁
>
> tryLock()  尝试获取锁，获取不到就返回false
>
> tryLock(long time, TimeUnit unit)  带超时时间的获取锁，获取不到就返回false
>
> unlock()  解锁
>
> newCondition()  条件队列



## 定义内部类，继承AbstractQueuedSynchronizer

AbstractQueuedSynchronizer也即是我们平时所说的AQS，其中定义了一些操作锁的基础方法，以及一些抽象方法，供子类扩展。本章不对AQS进行过多讲解，只实现功能，后面对其详解。

在AQS中会有个 **state** 属性，标识是否有线程持有锁。默认为0，没有线程持有锁；为1时，有锁以及被持有，就需要进入队列中等待。

我们只实现加锁解锁逻辑，所以只用实现tryAcquire和tryRelease这两个抽象方法。

```java
static class LockSync extends AbstractQueuedSynchronizer {
    /**
     * 能获取锁返回true，获取不到返回false
     */
    @Override
    protected boolean tryAcquire(int arg) {
        //尝试将状态从0，改为1
        if (compareAndSetState(0, arg)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    /**
     * 解锁，state改为0，ownerThread制空
     */
    @Override
    protected boolean tryRelease(int arg) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    /**
     * 条件队列
     */
    public Condition newCondition() {
        return new ConditionObject();
    }
}
```

加锁的时候会通过，CAS将state0改为1，如果成功就表示获取锁，将持有锁的线程改为当前线程

解锁的时候，当前线程已经持有锁，所以直接将持有锁的线程改为空，状态改为0

返回条件队列，因为Lock中有newCondition()，所以就在LockSync中直接new一个ConditionObject

具体实现

```java
public class CustomizeLock implements Lock {
    private LockSync sync;
    public CustomizeLock() {
        sync = new LockSync();
    }
    static class LockSync extends AbstractQueuedSynchronizer {
        /**能获取锁返回true，获取不到返回false*/
        @Override
        protected boolean tryAcquire(int arg) {
            //尝试将状态从0，改为1
            if (compareAndSetState(0, arg)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        /**解锁，state改为0，ownerThread制空*/
        @Override
        protected boolean tryRelease(int arg) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        /**条件队列*/
        public Condition newCondition() {
            return new ConditionObject();
        }
    }
    /** 获取锁 获取不到就入队*/
    @Override
    public void lock() {
        sync.acquire(1);//aqs自带的acquire方法
    }
    /**可打断锁*/
    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);//aqs自带的acquireInterruptibly方法
    }
    /** 尝试获取锁，获取不到就返回false*/
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }
    /**带超时时间的获取锁*/
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));//aqs自带的tryAcquireNanos方法
    }
    /**解锁*/
    @Override
    public void unlock() {
        sync.release(1);
    }
    /** 条件*/
    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

## 测试

```java
CustomizeLock lock = new CustomizeLock();
new Thread(() -> {
    lock.lock();
    System.out.println(Thread.currentThread().getName() + ":running..."+System.currentTimeMillis());
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println(Thread.currentThread().getName() + ":running end...");
    lock.unlock();
}, "t1").start();
new Thread(() -> {
    lock.lock();
    System.out.println(Thread.currentThread().getName() + ":running..."+System.currentTimeMillis());
    lock.unlock();
    System.out.println(Thread.currentThread().getName() + ":running end...");
}, "t2").start();
```

创建两个线程分别进行加锁，第一个线程获取锁后睡眠3s，然后释放锁。第二个线程获取锁后继续输出

输出：

> t1:running...1611819479953
> t1:running end...
> t2:running...1611819482955
> t2:running end...

相差3002ms，差不多就是3秒

## 原理

### 加锁

lock会调用，AQS中的acquire方法

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();//如果打断返回为true  重新产生中断(导致不可中断)
}
```

tryAcquire是抽象方法，所以会调用LockSync中自己实现的tryAcquire，尝试加锁，如果获取失败就会调用acquireQueued方法。acquireQueued中会再次尝试加锁，如果获取失败就会加入到等待队列(当然还会有其他的操作)。selfInterrupt() 中只有线程获取到锁并且被打断后才会执行里面的方法

### 解锁

unlock会调用，AQS中的release方法

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

tryRelease在LockSync也有自己的实现，tryRelease后会调用unparkSuccessor叫醒等待队列中的线程（AQS中维护了一个双向链表，unparkSuccessor也就是叫醒链表中的后继节点）

