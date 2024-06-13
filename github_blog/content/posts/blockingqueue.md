---
title: "阻塞队列-双锁实现"
date: 2023-06-16T16:14:17+08:00
draft: true
categories: ["JUC"]
weight: 10
---

### 前提摘要

在阻塞队列-单锁的实现中，**消费 （poll）** 和 **生产 （offer）** 两个方法都是用了同一个锁，也就是会发生队列中有元素，但消费方却有可能等待获取锁，从而不能消费的情况。 为了要让消费 的时候可以生产，生产的时候可以消费 从而有了 阻塞队列-双锁实现



 

### 写法一（会产生死锁）

  ![](deadlock.png)



在offer方法中向队列添加完元素后 ，唤醒 Condition headWaits

在poll方法中向队列消费元素后，唤醒 Condition tailWaits



但因为上面这种写法  在获取了锁A且没有释放的情况下，尝试获取锁B，以及 在获取了锁B且没有释放的情况下，尝试获取锁A 这种情况下会产生死锁。



### 写法二 （两个锁改为平级后）

  ![](2.png)

将两个锁改为平级后，可以防止死锁。但并没有减少 在offer方法后 对headLock加锁的次数。希望要尽量减少。



### 最终写法（级联通知）

尽量减少offer方法中对headLock的加锁次数，在poll方法中自己唤醒自己。

offer方法中改为了元素个数从0->1的时候，即空->非空的时候，才会对headLock的加锁然后唤醒。poll方法还可以消费元素的话，poll方法的最后面还要自己signal()

poll方法中改为了元素个数从满到不满 array.length->array.length-1 的时候，才会对tailLock的加锁然后唤醒。offer方法还可以添加元素的话，offer方法的最后面 还要自己signal自己

队列从满->不满时   由poll唤醒等待不满的 offer 线程

如果从0变为非空，由offer这边唤醒等待非空的poll线程

 



```
import java.util.Arrays;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 双锁实现
 * @param <E> 元素类型
 */
@SuppressWarnings("all")
public class BlockingQueue2<E> implements BlockingQueue<E> {

    private final E[] array;
    private int head;
    private int tail;
    private AtomicInteger size = new AtomicInteger();

    private ReentrantLock tailLock = new ReentrantLock();
    private Condition tailWaits = tailLock.newCondition();

    private ReentrantLock headLock = new ReentrantLock();
    private Condition headWaits = headLock.newCondition();

    public BlockingQueue2(int capacity) {
        this.array = (E[]) new Object[capacity];
    }

    private boolean isEmpty() {
        return size.get() == 0;
    }

    private boolean isFull() {
        return size.get() == array.length;
    }

    @Override
    public String toString() {
        return Arrays.toString(array);
    }

    @Override
    public void offer(E e) throws InterruptedException {
        int c; // 添加前元素个数
        tailLock.lockInterruptibly();
        try {
            // 1. 队列满则等待
            while (isFull()) {
                tailWaits.await(); //  offer2
            }

            // 2. 不满则入队
            array[tail] = e;
            if (++tail == array.length) {
                tail = 0;
            }

            // 3. 修改 size
            /*
                size = 6
             */
            c = size.getAndIncrement();
            if (c + 1 < array.length) {
                tailWaits.signal();
            }
            /*
                1. 读取成员变量size的值  5
                2. 自增 6
                3. 结果写回成员变量size 6
             */
        } finally {
            tailLock.unlock();
        }

        // 4. 如果从0变为非空，由offer这边唤醒等待非空的poll线程
        //                       0->1   1->2    2->3
        if(c == 0) {
            headLock.lock(); // offer_1 offer_2 offer_3
            try {
                headWaits.signal();
            } finally {
                headLock.unlock();
            }
        }
    }

    @Override
    public E poll() throws InterruptedException {
        E e;
        int c; // 取走前的元素个数
        headLock.lockInterruptibly();
        try {
            // 1. 队列空则等待
            while (isEmpty()) {
                headWaits.await(); // poll_4
            }

            // 2. 非空则出队
            e = array[head];
            array[head] = null; // help GC
            if (++head == array.length) {
                head = 0;
            }

            // 3. 修改 size
            c = size.getAndDecrement();
            // 3->2   2->1   1->0
            // poll_1 poll_2 poll_3
            if (c > 1) {
                headWaits.signal();
            }
            /*
                1. 读取成员变量size的值 5
                2. 自减 4
                3. 结果写回成员变量size 4
             */
        } finally {
            headLock.unlock();
        }

        // 4. 队列从满->不满时 由poll唤醒等待不满的 offer 线程
        if(c == array.length) {
            tailLock.lock();
            try {
                tailWaits.signal(); // ctrl+alt+t
            } finally {
                tailLock.unlock();
            }
        }

        return e;
    }

    @Override
    public boolean offer(E e, long timeout) throws InterruptedException {
        return false;
    }

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue2<String> queue = new BlockingQueue2<>(3);
        queue.offer("元素1");
        queue.offer("元素2");

        new Thread(()->{
            try {
                queue.offer("元素3");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }, "offer").start();

        new Thread(()->{
            try {
                queue.poll();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }, "poll").start();
    }
}
```
