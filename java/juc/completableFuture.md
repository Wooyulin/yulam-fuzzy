#  `CompletableFuture` 的作用

# 创建`completableFuture`

分为两类型： 

1. runAsync() runnable 接口的无返回值
2. supplyAsync() supplier接口有返回值

```java
// 使用默认线程池
static CompletableFuture<Void> 
  runAsync(Runnable runnable)
static <U> CompletableFuture<U> 
  supplyAsync(Supplier<U> supplier)
// 可以指定线程池  
static CompletableFuture<Void> 
  runAsync(Runnable runnable, Executor executor)
static <U> CompletableFuture<U> 
  supplyAsync(Supplier<U> supplier, Executor executor)  

```

默认线程池，使用的同一个forkjoinpool ，共享的线程池可能会导致阻塞，所以建议都是根据业务创建自定义的线程池

# 如何理解CompletionStage

这一步骤就是任务的执行，可以分为几种类型

## 串行

![image-20220518193903195](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220518193903195.png)

### 方法

#### thenApply  

接收一个参数，并且有一个返回

#### thenAccept

接受参数，但是不返回

#### thenRun

两个都没有

#### thenCompose

用来连接两个CompletableFuture，是生成一个新的CompletableFuture。

```java
// 接收一个参数，并且有一个返回
CompletionStage<R> thenApply(fn);
CompletionStage<R> thenApplyAsync(fn);
//接受参数，但是不返回
CompletionStage<Void> thenAccept(consumer);
CompletionStage<Void> thenAcceptAsync(consumer);
//两个都没有
CompletionStage<Void> thenRun(action);
CompletionStage<Void> thenRunAsync(action);
// 接受一个参数，用来连接两个CompletableFuture，是生成一个新的CompletableFuture。
CompletionStage<R> thenCompose(fn);
CompletionStage<R> thenComposeAsync(fn);


```

测试代码

```java
    public static void testThen() {
        /**
         * thenApply
         * 接受一个参数，一个返回值
         */
        CompletableFuture<String> result = CompletableFuture.supplyAsync(() -> "nihao")
                .thenApply(s -> s + " zyf")
//                .thenApply(String::toUpperCase);
                .thenApply(String::toUpperCase);
        System.out.println("thenApply: " + result.join());

        /**
         * thenAccept
         * 无返回值
         */
        CompletableFuture<Void> accept = CompletableFuture.supplyAsync(() -> "no then accept")
                .thenAccept(System.out::println);
        System.out.println("thenAccept:" + accept.join());

        /**
         * thenRun
         * 无入参，无返回 then run
         *
         */
        CompletableFuture<Void> thenRun = CompletableFuture.supplyAsync(() -> "then run")
                .thenRun(() -> System.out.println("无入参，无返回 then run "));
        System.out.println("thenRun: " + thenRun.join());
        /**
         * thenCompose
         * 场景是一个 String 的caf 转换为一个Integer的caf
         */
        CompletableFuture<Integer> length = CompletableFuture.supplyAsync(() -> "nihao")
//                .thenApply(String::toUpperCase);
                .thenCompose(s ->
                        CompletableFuture.supplyAsync(s::length)
                );
        System.out.println("thenCompose: " + length.join());
    }

//

```

输出：

```java
thenApply: NIHAO ZYF
no then accept
thenAccept:null
无入参，无返回 then run 
thenRun: null
thenCompose: 5
```

### 问题

可以看到每个方法都有两个版本，例如thenApply、thenApplyAsync

如果使用`thenApplyAsync`，那么执行的线程是从`ForkJoinPool.commonPool()`中获取不同的线程进行执行，如果使用`thenApply`，如果`supplyAsync`方法执行速度特别快，那么`thenApply`任务就是主线程进行执行，如果执行特别慢的话就是和`supplyAsync`执行线程一样。接下来我们通过例子来看一下，使用`sleep`方法来反应`supplyAsync`执行速度的快慢。

https://www.jianshu.com/p/f744eface31a



## 并行

![image-20220518193853648](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220518193853648.png)

### 方法

#### thenCombine

有返回值

#### thenAcceptBoth

接受一个消费函数，返回空的

#### runAfterBoth



```java
CompletionStage<R> thenCombine(other, fn);
CompletionStage<R> thenCombineAsync(other, fn);
CompletionStage<Void> thenAcceptBoth(other, consumer);
CompletionStage<Void> thenAcceptBothAsync(other, consumer);
CompletionStage<Void> runAfterBoth(other, action);
CompletionStage<Void> runAfterBothAsync(other, action);
```





## 汇聚

![image-20220518193831477](https://yulam-1308258423.cos.ap-guangzhou.myqcloud.com/note/image-20220518193831477.png)

### 方法



```java
CompletionStage applyToEither(other, fn);
CompletionStage applyToEitherAsync(other, fn);
CompletionStage acceptEither(other, consumer);
CompletionStage acceptEitherAsync(other, consumer);
CompletionStage runAfterEither(other, action);
CompletionStage runAfterEitherAsync(other, action);
```





# 异常处理

```java
CompletionStage exceptionally(fn); //catch
CompletionStage<R> whenComplete(consumer); //finally w
CompletionStage<R> whenCompleteAsync(consumer);
CompletionStage<R> handle(fn);//finally  有返回结果
CompletionStage<R> handleAsync(fn);
```

