---
title: "CompletableFuture"
date: 2023-12-13T08:28:26+08:00
draft: true
---

## CompletableFuture和Future区别

Future

- 只有阻塞才能获取结果，即get方法



CompletableFuture 

- 支持异步方法回调，
- 支持多任务编排
- 支持异常处理

##  创建异步任务



runAsync 开启不带返回结果的异步任务,入参是Runnable

supplyAsync 开启带返回结果的异步任务,入参是Supplier



##  异步任务回调

注意：thenApply ，thenAccpet，thenRun 如果是直接链式调用 则是串行化执行




需要获取上次CompletableFuture 的结果

- thenApply 添加异步任务回调 入参是Function
- thenAccpet添加异步任务回调 入参是Consumer

不需要获取上次CompletableFuture 的结果

- thenRun 添加异步任务回调 入参是Runnable





方法中名有没有 async 的区别

thenApply 和他上一个方法是在一个线程内执行

thenApplyAsync 和他上一个方法不一定在一个线程内执行

```
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    log.debug("supplyAsync1...");
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 1;
});
CompletableFuture<String> future2 = future1.thenApply(x ->
        {
            log.debug("supplyAsync2...:x="+ x);

            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return String.valueOf(x + 1);
        }
);
future2.thenAccept((x) -> {
            log.debug("supplyAsync3...:x="+x);

            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
);




```
```
15:42:01.445 c.testcompletableFuture [ForkJoinPool.commonPool-worker-1] - supplyAsync1...
15:42:06.460 c.testcompletableFuture [ForkJoinPool.commonPool-worker-1] - supplyAsync2...:x=1
15:42:11.474 c.testcompletableFuture [ForkJoinPool.commonPool-worker-1] - supplyAsync3...:x=2
```




##  异步任务编排


### 2个任务


#### 【thenCompose】2个互相依赖关系的 异步任务编排（a做完了，b才开始做）

thenCompose

thenCompose与thenApply类似   接受的参数也是个Function

thenApply  vs thenCompose？

可以看到返回值不一样，在回调函数thenApply方法接受的参数是CompletableFuture,比如引入第三方的一些api，那么使用thenCompose 就更合理

```
        CompletableFuture<CompletableFuture<Integer>> future3 =
                CompletableFuture.supplyAsync(() -> 1)
                        .thenApply(x -> CompletableFuture.supplyAsync(() -> x + 1));

        CompletableFuture<String> future2 =
                CompletableFuture.supplyAsync(() -> 1)
                        .thenCompose(x -> CompletableFuture.supplyAsync(() -> String.valueOf(x + 1)));
```



#### 【thenCombine】2个非互相非依赖的 异步任务编排（a和b同时开始做，a做完了，b也做完了之后执行Function）

```
/**
 * thenCombine
 */
public static void thenCombine() {


    CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
        log.debug("supplyAsync1... start");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("supplyAsync1... end ");
        return 1;
    });
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
        log.debug("supplyAsync2... start");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("supplyAsync2... end");
        return "2";
    });

    CompletableFuture<List> thenCombineFuture = future1.thenCombine(future2, (int1, string1) -> {
        log.debug("thenCombine...:int1={},string1={}", int1, string1);
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        List list = new ArrayList();
        list.add(int1);
        list.add(string1);
        log.debug("thenCombine...:list={}", list);
        return list;


    });


}
```

```
15:50:08.708 c.testcompletableFuture [ForkJoinPool.commonPool-worker-2] - supplyAsync2... start
15:50:08.708 c.testcompletableFuture [ForkJoinPool.commonPool-worker-1] - supplyAsync1... start
15:50:13.724 c.testcompletableFuture [ForkJoinPool.commonPool-worker-2] - supplyAsync2... end
15:50:13.724 c.testcompletableFuture [ForkJoinPool.commonPool-worker-1] - supplyAsync1... end 
15:50:13.724 c.testcompletableFuture [ForkJoinPool.commonPool-worker-1] - thenCombine...:int1=1,string1=2
15:50:18.725 c.testcompletableFuture [ForkJoinPool.commonPool-worker-1] - thenCombine...:list=[1, 2]
```





### 多个异步任务的编排

