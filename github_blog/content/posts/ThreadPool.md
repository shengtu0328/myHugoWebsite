---
title: "线程池"
date: 2022-12-01T14:14:25+08:00
draft: true
categories: ["JUC"]
weight: 1
---

## 我是零.线程池的主要作用

- **降低资源消耗**：创建线程（分配栈内存）和销毁线程都是消耗资源的，通过线程池可以重复利用已创建的线程，降低线程创建和销毁造成的损耗。

- **线程的管理型**:  无节制的创建线程，会对系统内存占用相当大造成OOM。如果线程特别多，cpu数量硬件固定忙不过来，会造成上下文频繁切换，导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。

- **其他**

  


## 二.自定义线程池

### Stage1.如何自己手写一个最简单的线程池？

{{< admonition >}}
虽然是个简单的线程池，但也是在模拟JUC.ThreadPoolExecutor。写完后可以更容易理解JUC.ThreadPoolExecutor。
{{< /admonition >}}

  ![](threadpool.png)

#### 需要以下组件

1. **线程集合**：线程集合中有着一批可重复利用的线程,专注于(不停的)(996的)(while循环的)处理任务。（任务的来源分两种：线程当前执行的任务和从队列获得的新任务）

2. **阻塞队列**：如果线程集合来不及处理，可以将任务放入队列暂存

      2.1. 但为什么还必须是一个有**阻塞**功能的队列呢？  

      - 向队列中消费任务时，队列中有任务时，会存在多个线程同时去队列获取任务的情况，阻塞队列可以保证消费者的线程安全
      - 向队列中消费任务时，队列中没有任务时，线程集合中的线程可以阻塞住，一旦拿到任务可以继续执行任务
      - 向队列中提交任务时，也需要控制生产者的线程安全
      - 向队列中提交任务时，如果队列任务满了需要阻塞
      
      2.2  队列一般都是要有容量上限，不会无限量的存放任务

#### 重要流程

#####  1）threadPool.execute()-执行流程

![](ThreadPool流程图.png)

```
/**
 * 执行任务，生产任务提交线程池
 *
 * @param task
 */
public void execute(Runnable task) {
    // 当线程集合没满时，创建worker对象并交给他执行
    // 当线程集合满了时，加入任务队列暂存
    synchronized (workers) {
        if (workers.size() < coreSize) {
            MyWorker worker = new MyWorker(task);
            log.debug("新增 worker{}, {}", worker, task);
            workers.add(worker);
            worker.start();
        } else {
            taskQueue.put(task);
        }
    }
}
```

##### 2）线程集合中的两种线程-运行流程

工作线程应该完成当前的任务，如果当前任务执行完了，队列中还有任务的话，去执行队列里的任务

去队列里获取任务的时候，能体现出java.util.concurrent.ThreadPoolExecutor中 核心线程和救急线程的执行逻辑

|                      | 代码                              | 逻辑                 |
| -------------------- | --------------------------------- | -------------------- |
| 核心线程             | taskQueue.take()                  | 创建出来后会一直存活 |
| 救急线程（外包线程） | taskQueue.poll(timeout, timeUnit) | 执行完会销毁         |



```
public void run() {
    // 执行任务，重用线程 taskQueue.take()是关键！！！
    // 1) 当 task 不为空，执行任务
    // 2) 当 task 执行完毕，再接着从任务队列获取任务并执行
    // while(task != null || (task = taskQueue.poll(timeout, timeUnit)) != null) {
    while (task != null || (task = taskQueue.take()) != null) {
        try {
            log.debug("正在执行...{}", task);
            task.run();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            task = null;
        }
    }
    //能走到这一步只会是执行 taskQueue.poll(timeout, timeUnit) 的线程
    synchronized (workers) {
        log.debug("worker 被移除{}", this);
        workers.remove(this);
    }
}
```



#### 代码

代码仓库 https://gitee.com/xiaoruiqing/juc/tree/master/src/main/java/ThreadPool/CustomThreadPool/Stage1

##### 1）自定义的一个阻塞队列

```

import lombok.extern.slf4j.Slf4j;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 自定义的一个阻塞队列
 *
 * @author xrq
 */
@Slf4j(topic = "c.MyBlockingQueue")
public class MyBlockingQueue<T> {

    /**
     * 1. 任务队列 ，队列先进先出 ，Deque双向链表
     */
    private Deque<T> queue = new ArrayDeque<>();

    /**
     * 2. 锁  1：锁的作用是防止一个任务被三个线程获取,
     *        2：还有就是队列中没有元素时，可以通过reentrantLock.newCondition() 等待。
     */
    private ReentrantLock lock = new ReentrantLock();

    /**
     * 3. 生产者条件变量：如果队列满了生产者线程去这个Condition个await(),等待队列不满时被消费者唤醒
     */
    private Condition fullWaitSet = lock.newCondition();

    /**
     * 4. 消费者条件变量：如果队列空了消费者线程去这个Condition里await(),等待队列不空时被生产者唤醒
     */
    private Condition emptyWaitSet = lock.newCondition();

    /**
     * 5. 容量
     */
    private int capacity;

    public MyBlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    /**
     * 阻塞获取
     *
     * @return
     */
    public T take() {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                try {
                    emptyWaitSet.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            T t = queue.removeFirst();
            fullWaitSet.signal();
            return t;
        } finally {
            lock.unlock();
        }
    }

    /**
     * 带超时的阻塞获取
     *
     * @param timeout 超时时间
     * @param unit    时间单位
     * @return
     */
    public T poll(long timeout, TimeUnit unit) {
        lock.lock();
        try {
            // 将 timeout 统一转换为 纳秒
            long nanos = unit.toNanos(timeout);
            while (queue.isEmpty()) {
                try {
                    // 等待时间过了队列还是空就结束
                    if (nanos <= 0) {
                        return null;
                    }
                    // 返回值是剩余等待时间
                    nanos = emptyWaitSet.awaitNanos(nanos);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            T t = queue.removeFirst();
            fullWaitSet.signal();
            return t;
        } finally {
            lock.unlock();
        }
    }


    /**
     * 阻塞添加
     *
     * @param task
     */
    public void put(T task) {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                try {
                    log.debug("等待加入任务队列 {} ...", task);
                    fullWaitSet.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("加入任务队列 {}", task);
            queue.addLast(task);
            emptyWaitSet.signal();
        } finally {
            lock.unlock();
        }
    }


    /**
     * 获取队列大小
     *
     * @return
     */
    public int size() {
        lock.lock();
        try {
            return queue.size();
        } finally {
            lock.unlock();
        }
    }

}
```

##### 2）自定义的工作线程

```
class MyWorker extends Thread {
    private Runnable task;

    public MyWorker(Runnable task) {
        this.task = task;
    }

    @Override
    public void run() {
        // 执行任务，重用线程 taskQueue.take()是关键！！！
        // 1) 当 task 不为空，执行任务
        // 2) 当 task 执行完毕，再接着从任务队列获取任务并执行
        // while(task != null || (task = taskQueue.poll(timeout, timeUnit)) != null) {
        while (task != null || (task = taskQueue.take()) != null) {
            try {
                log.debug("正在执行...{}", task);
                task.run();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                task = null;
            }
        }
        //能走到这一步只有带从队列获取超时时间的线程
        synchronized (workers) {
            log.debug("worker 被移除{}", this);
            workers.remove(this);
        }
    }
}
```



##### 3）自定义的线程池

```
import lombok.extern.slf4j.Slf4j;

import java.util.HashSet;
import java.util.concurrent.TimeUnit;

/**
 * @author xrq
 */
@Slf4j(topic = "c.MyThreadPool")
public class MyThreadPool {

    /**
     * 任务队列
     */
    private MyBlockingQueue<Runnable> taskQueue;

    /**
     * 线程集合
     */
    private HashSet<MyWorker> workers = new HashSet<>();

    /**
     * 核心线程数
     */
    private int coreSize;

    /**
     * 从队列获取任务时的超时时间
     */
    private long timeout;

    /**
     * 从队列获取任务时的超时时间 的 时间单位
     */
    private TimeUnit timeUnit;


    public MyThreadPool(int coreSize, long timeout, TimeUnit timeUnit, int queueCapacity) {
        this.coreSize = coreSize;
        this.timeout = timeout;
        this.timeUnit = timeUnit;
        this.taskQueue = new MyBlockingQueue<>(queueCapacity);
    }

    /**
     * 执行任务，生产任务提交线程池
     *
     * @param task
     */
    public void execute(Runnable task) {
        // 当线程集合没满时，创建worker对象并交给他执行
        // 当线程集合满了时，加入任务队列暂存
        synchronized (workers) {
            if (workers.size() < coreSize) {
                MyWorker worker = new MyWorker(task);
                log.debug("新增 worker{}, {}", worker, task);
                workers.add(worker);
                worker.start();
            } else {
                taskQueue.put(task);
            }
        }
    }

    //工作线程
    class MyWorker extends Thread {
       //..........
    }

}

```

##### 4）测试类

```
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

/**
 * 测试类
 */
@Slf4j(topic = "c.CustomThreadPool")
public class MyThreadPoolTest {


    public static void main(String[] args) {

        MyThreadPool threadPool = new MyThreadPool(2, 1000, TimeUnit.MILLISECONDS, 10);

        for (int i = 0; i < 5; i++) {
            int j = i;
            threadPool.execute(() -> {
                log.debug("任务被执行了" + j);
            });
        }
    }

}
```

1）使用 `while (task != null || (task = taskQueue.take()) != null) {` 代码测试，发现工作线程并没有执行完毕。符合核心线程复用的预期结果。

{{< admonition info >}}
14:22:05.022 c.MyThreadPool [main] - 新增 worker Thread[Thread-0,5,main], ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1888ff2c
14:22:05.028 c.MyThreadPool [main] - 新增 worker Thread[Thread-1,5,main], ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1b40d5f0
14:22:05.028 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1888ff2c
14:22:05.028 c.CustomThreadPool [Thread-0] - 任务被执行了0
14:22:05.028 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1b40d5f0
14:22:05.028 c.CustomThreadPool [Thread-1] - 任务被执行了1
14:22:05.029 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@3c5a99da
14:22:05.029 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@3c5a99da
14:22:05.029 c.CustomThreadPool [Thread-1] - 任务被执行了2
14:22:05.030 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@47f37ef1
14:22:05.030 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@47f37ef1
14:22:05.030 c.CustomThreadPool [Thread-0] - 任务被执行了3
14:22:05.030 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5a01ccaa
14:22:05.030 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5a01ccaa
14:22:05.030 c.CustomThreadPool [Thread-1] - 任务被执行了4
{{< /admonition >}}



2）使用 `while(task != null || (task = taskQueue.poll(timeout, timeUnit)) != null) {` 代码测试，发现工作线程最终执行完毕。符合救急线程预期结果。

{{< admonition info >}}
14:18:23.020 c.MyThreadPool [main] - 新增 worker Thread[Thread-0,5,main], ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1888ff2c
14:18:23.026 c.MyThreadPool [main] - 新增 worker Thread[Thread-1,5,main], ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1b40d5f0
14:18:23.026 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1888ff2c
14:18:23.026 c.CustomThreadPool [Thread-0] - 任务被执行了0
14:18:23.027 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1b40d5f0
14:18:23.027 c.CustomThreadPool [Thread-1] - 任务被执行了1
14:18:23.027 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@3c5a99da
14:18:23.028 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@3c5a99da
14:18:23.028 c.CustomThreadPool [Thread-1] - 任务被执行了2
14:18:23.028 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@47f37ef1
14:18:23.028 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5a01ccaa
14:18:23.028 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@47f37ef1
14:18:23.028 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5a01ccaa
14:18:23.028 c.CustomThreadPool [Thread-0] - 任务被执行了3
14:18:23.028 c.CustomThreadPool [Thread-1] - 任务被执行了4

Process finished with exit code 0
{{< /admonition >}}



3）当任务量大，发生的现象。taskQueue.put(task);这步会阻塞住，这样不好，主线程阻塞住了就不能提交接下来的任务了。当队列慢时，主线程应该能作出选择，这就是接下来要说的拒绝策略。

如2个工作线程，队列长度10，15个任务。

{{< admonition info >}}

15:00:34.296 c.MyThreadPool [main] - 新增 worker Thread[Thread-0,5,main], ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1888ff2c
15:00:34.301 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1888ff2c
15:00:34.301 c.MyThreadPool [main] - 新增 worker Thread[Thread-1,5,main], ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1b40d5f0
15:00:34.301 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1b40d5f0
15:00:34.301 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@3c5a99da
15:00:34.302 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@47f37ef1
15:00:34.302 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5a01ccaa
15:00:34.302 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@71c7db30
15:00:34.302 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@19bb089b
15:00:34.302 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@4563e9ab
15:00:34.302 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@11531931
15:00:34.302 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5e025e70
15:00:34.303 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1fbc7afb
15:00:34.303 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@45c8e616
15:00:34.303 c.MyBlockingQueue [main] - 等待加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@4cdbe50f ...
15:00:43.305 c.CustomThreadPool [Thread-0] - 任务被执行了0
15:00:43.305 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@3c5a99da
15:00:43.305 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@4cdbe50f
15:00:43.305 c.MyBlockingQueue [main] - 等待加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@7cf10a6f ...
15:00:43.306 c.CustomThreadPool [Thread-1] - 任务被执行了1
15:00:43.306 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@47f37ef1
15:00:43.306 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@7cf10a6f
15:00:43.306 c.MyBlockingQueue [main] - 等待加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@7e0babb1 ...
15:00:52.310 c.CustomThreadPool [Thread-0] - 任务被执行了2
15:00:52.310 c.CustomThreadPool [Thread-1] - 任务被执行了3
15:00:52.310 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@71c7db30
15:00:52.310 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5a01ccaa
15:00:52.310 c.MyBlockingQueue [main] - 加入任务队列 ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@7e0babb1
15:01:01.315 c.CustomThreadPool [Thread-1] - 任务被执行了5
15:01:01.315 c.CustomThreadPool [Thread-0] - 任务被执行了4
15:01:01.316 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@19bb089b
15:01:01.316 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@4563e9ab
15:01:10.319 c.CustomThreadPool [Thread-0] - 任务被执行了7
15:01:10.319 c.CustomThreadPool [Thread-1] - 任务被执行了6
15:01:10.319 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@11531931
15:01:10.319 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@5e025e70
15:01:19.324 c.CustomThreadPool [Thread-0] - 任务被执行了8
15:01:19.324 c.CustomThreadPool [Thread-1] - 任务被执行了9
15:01:19.324 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@45c8e616
15:01:19.324 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@1fbc7afb
15:01:28.328 c.CustomThreadPool [Thread-1] - 任务被执行了11
15:01:28.328 c.CustomThreadPool [Thread-0] - 任务被执行了10
15:01:28.328 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@4cdbe50f
15:01:28.329 c.MyThreadPool [Thread-0] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@7cf10a6f
15:01:37.330 c.CustomThreadPool [Thread-1] - 任务被执行了12
15:01:37.330 c.CustomThreadPool [Thread-0] - 任务被执行了13
15:01:37.331 c.MyThreadPool [Thread-1] - 正在执行...ThreadPool.CustomThreadPool.Stage1.MyThreadPoolTest$$Lambda$1/984849465@7e0babb1
15:01:46.332 c.CustomThreadPool [Thread-1] - 任务被执行了14

{{< /admonition >}}







### Stage2.给这个线程池配置拒绝策略

#### 拒绝策略是什么

如果阻塞队列满了，主线程应该使用配置的拒绝策略来执行具体的逻辑,而非只有没有超时时间的阻塞（等待）下去。

#### 几种拒绝策略

- 让调用者线程进行其中之一操作

	- 死等
	
	- 带超时等待
	
	- 放弃任务执行
	
	- 抛出异常，不继续执行了
	
	- 自己执行任务






#### 代码

代码仓库 https://gitee.com/xiaoruiqing/juc/tree/master/src/main/java/ThreadPool/CustomThreadPool/Stage2

##### 1）拒绝策略FunctionalInterface

拒绝策略应该获取到task对象和 queue才能执行拒绝策略，所以设计一个FunctionalInterface，参数是这两个

```
/**
 * 拒绝策略
 */
@FunctionalInterface
interface MyRejectPolicy<T> {
    void reject(MyBlockingQueue<T> queue, T task);
}
```









##### 2）主线程获取队列任务带超时时间的offer方法

 在之前的代码中如果队列满了，主线程会没有超时时间的阻塞（等待）下去。

所以先实现一个带超时时间的阻塞添加 offer方法

```
/**
 * 带超时时间的阻塞添加
 *
 * @param task
 * @param timeout
 * @param timeUnit
 * @return
 */
public boolean offer(T task, long timeout, TimeUnit timeUnit) {
    lock.lock();
    try {
        long nanos = timeUnit.toNanos(timeout);
        while (queue.size() == capacity) {
            try {
                if (nanos <= 0) {
                    return false;
                }
                log.debug("等待加入任务队列 {} ...", task);
                nanos = fullWaitSet.awaitNanos(nanos);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("加入任务队列 {}", task);
        queue.addLast(task);
        emptyWaitSet.signal();
        return true;
    } finally {
        lock.unlock();
    }
}
```

##### 3）不同的拒绝策略实现

```
  public static void main(String[] args) {

        //这里的timeout是 工作线程从队列获取任务的超时时间
        MyThreadPool threadPool = new MyThreadPool(1, 1000, TimeUnit.MILLISECONDS, 1,
                (queue, task) -> {
                    // 1. 死等
//            queue.put(task);
                    // 2) 带超时等待
                    // 这里的timeout是队列满了之后，主线程等待队列不满的超时时间
            queue.offer(task, 500, TimeUnit.MILLISECONDS);
                    // 3) 让调用者放弃任务执行
//            log.debug("放弃{}", task);
                    // 4) 让调用者抛出异常
//            throw new RuntimeException("任务执行失败 " + task);
                    // 5) 让调用者自己执行任务
//                    task.run();
                });

        for (int i = 0; i < 5; i++) {
            int j = i;
            threadPool.execute(() -> {
                try {
                    Thread.sleep(2000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("任务被执行了" + j);
            });
        }
    }
```



### 提问

1）为什么只用了一把锁(ReentrantLock)，生产者拿到锁提交任务到队列的时候，此时队列中还有很多任务的时候，队列中有数据但消费者却因为获取不到锁了就不能消费了吗？不是降低了性能？

如果改成生产一个锁l1，消费一个锁l2，那生产者和消费者不是共用一把锁，就无法在生产者的同步代码中调用l2的condition2.singal()方法，如果这么写，只会报IllegalMonitorStateException。这实际上还是对ReentrantLock锁的理解问题。比如获得对应的锁，才能执行condition的方法。



2）那如果改成 消费和生产方法分别用两个锁实现？





------

3）线程池中的核心线程重复利用 和救急线程用完后移除是怎么实现的？

```
//核心线程
while (task != null || (task = taskQueue.take()) != null) {

//救急线程
while(task != null || (task = taskQueue.poll(timeout, timeUnit)) != null) {

        //.........
        synchronized (workers) {
            log.debug("worker 被移除{}", this);
            workers.remove(this);
        }
}
```





## 三.java.util.concurrent.ThreadPoolExecutor

核心不是一定会重用， jdk中有个allowCoreThreadTimeOut == false，false的时候 核心线程不会被销毁， true的时候 一段时间后会销毁

**3) newFixedThreadPool**

```
public static ExecutorService newFixedThreadPool(int nThreads) {

return new ThreadPoolExecutor(nThreads, nThreads,

0L, TimeUnit.MILLISECONDS,

new LinkedBlockingQueue<Runnable>());

}
```

特点

核心线程数 == 最大线程数（没有救急线程被创建），因此也无需超时时间

阻塞队列是无界的，可以放任意数量的任务

> **评价** 适用于任务量已知，相对耗时的任务



**4) newCachedThreadPool**

```
public static ExecutorService newCachedThreadPool() {

return new ThreadPoolExecutor(0, Integer.MAX_VALUE,

60L, TimeUnit.SECONDS,

new SynchronousQueue<Runnable>());

}
```

特点

核心线程数是 0， 最大线程数是 Integer.MAX_VALUE，救急线程的空闲生存时间是 60s，意味着

全部都是救急线程（60s 后可以回收）

> **评价** 适合任务数比较密集，但每个任务执行时间较短

**5) newSingleThreadExecutor**

```
public static ExecutorService newSingleThreadExecutor() {

return new FinalizableDelegatedExecutorService

 (new ThreadPoolExecutor(1, 1,

0L, TimeUnit.MILLISECONDS,

new LinkedBlockingQueue<Runnable>()));

}
```

使用场景：

希望多个任务排队执行。线程数固定为 1，任务数多于 1 时，会放入无界队列排队。任务执行完毕，这唯一的线程

也不会被释放。

区别：

自己创建一个单线程串行执行任务，如果任务执行失败而终止那么没有任何补救措施，而线程池还会新建一

个线程，保证池的正常工作

Executors.newSingleThreadExecutor() 线程个数始终为1，不能修改

FinalizableDelegatedExecutorService 应用的是装饰器模式，只对外暴露了 ExecutorService 接口，因

此不能调用 ThreadPoolExecutor 中特有的方法

Executors.newFixedThreadPool(1) 初始时为1，以后还可以修改

对外暴露的是 ThreadPoolExecutor 对象，可以强转后调用 setCorePoolSize 等方法进行修改



## 四.其他问题

 1.创建多少线程池合适？

cpu密集型 

CPU核数+1

IO密集型

经验公式如下

线程数 = 核数 * 期望 CPU 利用率 * 总时间(CPU计算时间+等待时间) / CPU 计算时间

例如 4 核 CPU 计算时间是 50% ，其它等待时间是 50%，期望 cpu 被 100% 利用，套用公式

4 * 100% * 100% / 50% = 8

例如 4 核 CPU 计算时间是 10% ，其它等待时间是 90%，期望 cpu 被 100% 利用，套用公式

4 * 100% * 100% / 10% = 40



## 参考：

https://www.bilibili.com/video/BV16J411h7Rd?p=204&spm_id_from=pageDriver&vd_source=429ff5cbebe69e9917bdeda0424404db

https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html
