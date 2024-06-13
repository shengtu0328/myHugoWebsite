---
title: "AQS"
date: 2022-10-27T14:29:43+08:00
draft: true
---

CountDownLatch->AQS->Park->Unsfae



### **1.** **概述**



**全称是 AbstractQueuedSynchronizer，是阻塞式锁和相关的同步器工具的框架**



1. 用 state 属性来表示资源的状态（分独占模式和共享模式），子类需要定义如何维护这个状态，控制如何获取

    锁和释放锁

    getState - 获取 state 状态

    setState - 设置 state 状态

    **compareAndSetState - cas 机制设置 state 状态**

    独占模式是只有一个线程能够访问资源，而共享模式可以允许多个线程访问资源

2. 如果加锁成功

    用cas 机制来设置state 来获取锁 和释放锁   state 属性由子类维护状态，state表示释放锁和获取锁

3. 如果加锁失败

    提供了基于 FIFO 的等待队列，类似于 Monitor 的 EntryList

    把没有获取到锁的线程添加到队列中,用LockSupport.park(this);来实现此线程阻塞（park是由Unsafe实现）

4. 释放锁时，如何唤醒其他锁？

    用LockSupport.unpark(s.thread);唤醒其他线程

5. await流程？

    条件变量来实现等待、唤醒机制，支持多个条件变量，类似于 Monitor 的 WaitSet

    实现类是ConditionObject条件变量  也是用 一个 单向链表存储，链表元素 关联了Node ，一个Node=（线程+Node.CONDITION 状态）

    await 方法   

    持有取锁时，条件不满足进入等待。   1.此线程加入ConditionObject队列，2.释放掉锁（可重入的次数全部释放掉） 3.唤醒第等待获取锁的队列的线程

    signal 方法   

    持有取锁时，1.让ConditionObject第一个元素断开后，加入等待获取锁的队列尾部，

**ReentrantLock实现**，ReentrantLock首先用到了aqs的 5 步，其次还实现公平和非公平锁

1. 公平锁   获取锁的时候直接去抢锁，会去检查aqs队列，队列中其他的线程在等待就排队

2. 非公平锁   获取锁的时候直接去抢锁，不会去看等待队列

3. 可重入 就是 state++来实现

4. 不可打断  等待线程被inerrupt 也能正常执行.

   可打断    等待线程被inerrupt 抛出异常











子类主要实现这样一些方法（默认抛出 UnsupportedOperationException）

tryAcquire

tryRelease

tryAcquireShared

tryReleaseShared

isHeldExclusively





### 获取锁的姿势

// 如果获取锁失败

if (!tryAcquire(arg)) {

 // 入队, 可以选择阻塞当前线程 park unpark 来实现

}

// 如果释放锁成功

if (tryRelease(arg)) {

 // 让阻塞线程恢复运行

}







```
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

import static cn.itcast.n2.util.Sleeper.sleep;

@Slf4j(topic = "c.TestAqs")
public class TestAqs {
    public static void main(String[] args) {
        MyLock lock = new MyLock();
        new Thread(() -> {
            lock.lock();
            try {
                log.debug("locking...");
                sleep(1);
            } finally {
                log.debug("unlocking...");
                lock.unlock();
            }
        },"t1").start();

        new Thread(() -> {
            lock.lock();
            try {
                log.debug("locking...");
            } finally {
                log.debug("unlocking...");
                lock.unlock();
            }
        },"t2").start();
    }
}

// 自定义锁（不可重入锁）
class MyLock implements Lock {

    // 独占锁  同步器类
    class MySync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            if(compareAndSetState(0, 1)) {
                // 加上了锁，并设置 owner 为当前线程
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override // 是否持有独占锁
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        public Condition newCondition() {
            return new ConditionObject();
        }
    }

    private MySync sync = new MySync();

    @Override // 加锁（不成功会进入等待队列）
    public void lock() {
        sync.acquire(1);
    }

    @Override // 加锁，可打断
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override // 尝试加锁（一次）
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override // 尝试加锁，带超时
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override // 解锁
    public void unlock() {
        sync.release(1);
    }

    @Override // 创建条件变量
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```







### ReentrantLock





公平锁 与 非公平锁



cas失败 添加一个node 到队列里

