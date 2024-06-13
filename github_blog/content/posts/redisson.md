---
title: "Redisson"
date: 2022-12-30T10:09:27+08:00
draft: true
---

Redis分布式锁（悲观锁）

假设需求是商品下单，并且一人只能买一单。

mysql update行锁可以限制操作单张库存中表库存字段不超卖，但这只适用于单纯数据库中update操作的业务逻辑。

如果要实现一人一单的业务逻辑，即先要查询当前用户有没有下过单，没下过单可以减库存。用户下单的这个操作算是一个新增逻辑。

假设也不去考虑使用数据库联合列唯一索引的话。可以用分布式锁来完成这个需求。



## 一.基于redis实现分布式锁

最基本需要实现两点  1.获取锁，2.释放锁

一人一单逻辑使用锁的代码

```
    public Result seckillVoucherOld(Long voucherId) {
        // 1.查询优惠券
        SeckillVoucher voucher = seckillVoucherService.getById(voucherId);
        // 2.判断秒杀是否开始
        if (voucher.getBeginTime().isAfter(LocalDateTime.now())) {
            // 尚未开始
            return Result.fail("秒杀尚未开始！");
        }
        // 3.判断秒杀是否已经结束
        if (voucher.getEndTime().isBefore(LocalDateTime.now())) {
            // 尚未开始
            return Result.fail("秒杀已经结束！");
        }
        // 4.判断库存是否充足
        if (voucher.getStock() < 1) {
            // 库存不足
            return Result.fail("库存不足！");
        }

        Long userId = UserHolder.getUser().getId();
        // 创建锁对象
         SimpleRedisLockStage1 lock = new SimpleRedisLockStage1("order:" + userId, stringRedisTemplate);
//        RLock lock = redissonClient.getLock("lock:order:" + userId);
        // 获取锁
        boolean isLock = lock.tryLock(1200);
        // 判断是否获取锁成功
        if(!isLock){
            // 获取锁失败，返回错误或重试
            return Result.fail("不允许重复下单");
        }
        try {
            // 获取代理对象（事务）
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
        } finally {
            // 释放锁
            lock.unlock();
        }
    }
```

### 

### 第0阶段

| redis命令          | 问题                                                         |
| ------------------ | ------------------------------------------------------------ |
| SETNX lock1 thread | value用线程id即可，可以知道是哪个线程获得了锁。SETNX命令锁的是key<br />SETNX 和 EXPIRE分成了两部，并不是一个完整的原子性操作。可能会在执行 EXPIRE前服务宕机，锁一直不释放 |
| EXPIRE lock1 10    |                                                              |
| DEL lock1          |                                                              |

### 第一阶段

| redis命令                  | 问题                            |
| -------------------------- | ------------------------------- |
| SET lock1 thread1 EX 10 NX | 将 NX 和EX 整合进一个原子操作中 |
| DEL lock1                  |                                 |



#SimpleRedisLockStage1.class

```
import org.springframework.data.redis.core.StringRedisTemplate;

import java.util.concurrent.TimeUnit;

public class SimpleRedisLockStage1 implements ILock {

    private String name;
    private StringRedisTemplate stringRedisTemplate;

    public SimpleRedisLockStage1(String name, StringRedisTemplate stringRedisTemplate) {
        this.name = name;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    private static final String KEY_PREFIX = "lock:";

    @Override
    public boolean tryLock(long timeoutSec) {
        // 获取线程标示
        long threadId = Thread.currentThread().getId();
        // 获取锁
        Boolean success = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId+" ", timeoutSec, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(success);
    }

    @Override
    public void unlock() {
        stringRedisTemplate.delete(KEY_PREFIX + name);
    }

}
```

### 第二阶段

![](unlockByAnotherThread.png) 

为了解决有可能产生的 锁 被非当前线程释放的问题，

#SimpleRedisLockStage2.class

```
public class SimpleRedisLockStage2 implements ILock {

    private String name;
    private StringRedisTemplate stringRedisTemplate;

    public SimpleRedisLockStage2(String name, StringRedisTemplate stringRedisTemplate) {
        this.name = name;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    private static final String KEY_PREFIX = "lock:";
    private static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";

    @Override
    public boolean tryLock(long timeoutSec) {
        // 获取线程标示
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        // 获取锁
        Boolean success = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId+" ", timeoutSec, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(success);
    }

    @Override
    public void unlock() {
        // 获取线程标示
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        // 获取锁中的标示
        String id = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);
        // 判断标示是否一致
        if(threadId.equals(id)) {
            // 释放锁
            stringRedisTemplate.delete(KEY_PREFIX + name);
        }
    }
}
```

### 第三阶段

![](unlockBlock.png) 

因为判断锁是不是当前线程1自己的 和 释放锁 不是在一个原子性操作里，在释放锁前，如果jvm发生GC STW，导致锁过期，从而让线程2由获得了锁。但线程1阻塞结束后直接 进行释放锁操作，导致线程2执行到一半锁却被释放。导致其他线程获得锁与线程2并行执行。

但实际上线程1的事物已经提交了才会释放锁，所以在本次一人一单的场景中，线程2包括其他线程获取到锁之后，会进行校验，能够保持正确逻辑？但是只要出现并行 就有可能影响？

解决办法：使用lua 整合redis多条命令为一个原子性操作。即判断锁+释放锁 整合为一条命令

`localhost:0>EVAL "return redis.call ('set','name1','jack1')" 0`

![](lua.png) 

#SimpleRedisLockStage3.class

```
import cn.hutool.core.lang.UUID;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;

import java.util.Collections;
import java.util.concurrent.TimeUnit;



public class SimpleRedisLockStage3 implements ILock {

    private String name;
    private StringRedisTemplate stringRedisTemplate;

    public SimpleRedisLockStage3(String name, StringRedisTemplate stringRedisTemplate) {
        this.name = name;
        this.stringRedisTemplate = stringRedisTemplate;
    }

    private static final String KEY_PREFIX = "lock:";
    private static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";
    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }

    @Override
    public boolean tryLock(long timeoutSec) {
        // 获取线程标示
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        // 获取锁
        Boolean success = stringRedisTemplate.opsForValue()
                .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(success);
    }

    @Override
    public void unlock() {
        // 调用lua脚本
        stringRedisTemplate.execute(
                UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX + name),
                ID_PREFIX + Thread.currentThread().getId());
    }

}
```

## 二.RLock（Redisson分布式锁）

![](自定义锁的问题.png) 



Redisson wiki

https://github.com/redisson/redisson/wiki/8.-distributed-locks-and-synchronizers



### 1.获取锁释放锁原理

1.redisson Rlock是一个实现了可重入的分布式锁。可重入这个部分的实现可以参考jdk里的**ReentrantLock**，利用了redis hash结构的(HSET key field value)。fileld还跟以前一样可以设置成线程id，每次重入锁之后，value会加1。但redis的 hash类型 没有string类型的setnx的命令。所以redisson为了实现操作的原子性，利用了lua脚本，自定义了一组命令来实现互斥。

并且在持有锁时，如果leaseTime！=-1时，会有一个额外的看门狗线程检查线程scheduleExpirationRenewal() 过期时间续约方法

在加锁的时候，如果没有获取到锁，还会有等待锁并且订阅 信号量 的机制，

在解锁的时候，会发布 释放锁的信号量  消息



1.redisson 加锁

![](redissonLock.png)





```
 // KEYS[1] : hash的key参数
 // ARGV[1] : LeaseTime 锁释放时间
 // ARGV[2] ：hash的field参数，当前进程的 redisson client+ 线程id
 // 判断是不是自己获得了锁
 
 // 获取到了锁
    // key不存在，set field，1
"if (redis.call('exists', KEYS[1]) == 0) then " +
"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +  // hash incr 1
"redis.call('pexpire', KEYS[1], ARGV[1]); " +     // 设置过期时间
"return nil; " +
"end; " +
    // key存在且field也是自己，执行重入操作
"if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +  // 重入锁 +1
"redis.call('pexpire', KEYS[1], ARGV[1]); " +     // 重新设置过期时间
"return nil; " +
"end; " +
//  获取不到锁，返回锁的剩余时间
"return redis.call('pttl', KEYS[1]);";
```

```
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);

    return evalWriteAsync(getName(), LongCodec.INSTANCE, command,
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                    "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                    "return nil; " +
                    "end; " +
                    "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                    "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                    "return nil; " +
                    "end; " +
                    "return redis.call('pttl', KEYS[1]);",
            Collections.singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
```



2.redisson 释放锁

```
 // KEYS[1] : hash的key参数
 // ARGV[1] : 解锁时候 发送的消息，用于通知其他等待锁的进程
 // ARGV[2] : LeaseTime 锁释放时间
 // ARGV[3] ：redisson client+ 线程id
 // 判断锁是否还是自己持有
    // 如果不是自己，直接返回
"if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
"return nil;" +
"end; " +
    // 如果是自己 ，value incrby 1
"local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
        //  判断重入次数是否为0
        // >0 还不能释放锁，并且续期
"if (counter > 0) then " +
"redis.call('pexpire', KEYS[1], ARGV[2]); " +
"return 0; " +
"else " +
        // <=0 可以解锁，并发送消息通知
"redis.call('del', KEYS[1]); " +
"redis.call('publish', KEYS[2], ARGV[1]); " +
"return 1; " +
"end; " +
"return nil;",
```

![](redissonUnlock.png)

```
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                    "end; " +
                    "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                    "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                    "else " +
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; " +
                    "end; " +
                    "return nil;",
            Arrays.asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
}
```

3.看门狗代码





如果tryLock方法自己设置了leaseTime， 那就没有看门狗了（锁续期功能）。

```
//设置看门狗 ，可以看到会有一个EXPIRATION_RENEWAL_MAP map来维护每个看门狗

private void renewExpiration() {
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (ee == null) {
        return;
    }
    
    Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
            if (ent == null) {
                return;
            }
            Long threadId = ent.getFirstThreadId();
            if (threadId == null) {
                return;
            }
            
            RFuture<Boolean> future = renewExpirationAsync(threadId);
            future.onComplete((res, e) -> {
                if (e != null) {
                    log.error("Can't update lock " + getName() + " expiration", e);
                    return;
                }
                
                if (res) {
                    // reschedule itself
                    renewExpiration();
                }
            });
        }
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
    
    ee.setTimeout(task);
}





 //重置锁 有效期！
 protected RFuture<Boolean> renewExpirationAsync(long threadId) {
     return evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
             "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                     "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                     "return 1; " +
                     "end; " +
                     "return 0;",
             Collections.singletonList(getName()),
             internalLockLeaseTime, getLockName(threadId));
 }







//取消看门狗

 @Override
    public RFuture<Void> unlockAsync(long threadId) {
        RPromise<Void> result = new RedissonPromise<Void>();
        RFuture<Boolean> future = unlockInnerAsync(threadId);

        future.onComplete((opStatus, e) -> {
            cancelExpirationRenewal(threadId);

            if (e != null) {
                result.tryFailure(e);
                return;
            }

            if (opStatus == null) {
                IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                        + id + " thread-id: " + threadId);
                result.tryFailure(cause);
                return;
            }

            result.trySuccess(null);
        });

        return result;
    }
    
    
        void cancelExpirationRenewal(Long threadId) {
        ExpirationEntry task = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (task == null) {
            return;
        }
        
        if (threadId != null) {
            task.removeThreadId(threadId);
        }

        if (threadId == null || task.hasNoThreads()) {
            Timeout timeout = task.getTimeout();
            if (timeout != null) {
                timeout.cancel();
            }
            EXPIRATION_RENEWAL_MAP.remove(getEntryName());
        }
    }
```





### 2.api使用

redisson有多个lock和trylock方法.

总结：

1. lock() 和 trylock()的区别，lock()如果获取不到锁会一直阻塞,类似synchronized 。trylock()如果获取不到锁可以选择放弃。
2. 方法参数：没有设置过释放时间的会自动续期。设置过释放时间的不会续期，过了设置的释放时间直接释放锁。也就是releaseTime如果是-1的，就没有看门狗。

lock()

```
//lock()如果线程一直没有执行完，会在剩 2/3的 持有锁时间时，重新续上完整的持有锁时间，直到线程结束运行
lock.lock(); 默认持有锁时间30s，到ttl 20s时，会再续到30s，会一直续下去，直到线程结束运行
//lock(90L, TimeUnit.SECONDS) 不会续期，超过了设定的释放时间，锁就会释放
lock.lock(90L, TimeUnit.SECONDS);  设置持有锁时间90s，90s过了程序没执行完，锁也会被释放
```

tryLock()

```
//tryLock()如果锁可用，则获取锁并立即返回值 true。 如果锁不可用，则此方法将立即返回值 false。
//获得锁后，默认持有锁时间30s，到ttl 20s时，会再续到30s，会一直续下去，直到线程结束运行
lock.tryLock();


//tryLock()如果在设置时间内获取到了锁，返回值 true。 如果锁不可用，会等待设定的时间，超过时间返回false
//获得锁后，默认持有锁时间30s，到ttl 20s时，会再续到30s，会一直续下去，直到线程结束运行
lock.tryLock(100L, TimeUnit.SECONDS);

//tryLock()如果锁可用，如果在设置时间内获取到了锁，返回值 true。 如果锁不可用，会等待设定的时间，超过时间返回false
//不会续期，超过了设定的释放时间，锁就会释放
boolean b2 = lock.tryLock(10,90L, TimeUnit.SECONDS);
```




```
//**但是 这种情况 先lock()后lock.lock(90L, TimeUnit.SECONDS); .lock(90L, TimeUnit.SECONDS)可以续签
lock.lock();
System.out.println("lock.lock() 获取到了锁");
Thread.sleep(100*1000);
lock.lock(90L, TimeUnit.SECONDS);
System.out.println("lock.lock(90L, TimeUnit.SECONDS) 获取到了锁");
Thread.sleep(300*1000);
```





### 3.流程

## 三.MultiLock（联锁）

![](红锁.png) 
