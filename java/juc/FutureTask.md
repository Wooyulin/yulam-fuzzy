# 简介与作用

## future 

future表示一个任务的生命周期，是一个可取消的异步运算，futureTask就是一个future的实现。

## futureTask

futureTask为future提供基础实现，封装runnable 、callable作为一个执行任务提交线程执行。

# 源码

## future接口

```java
public interface Future<V> {
    
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    //阻塞
    V get() throws InterruptedException, ExecutionException;
    //超时阻塞
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

## 核心属性

```java
著作权归https://pdai.tech所有。
链接：https://www.pdai.tech/md/java/thread/java-thread-x-juc-executor-FutureTask.html

//内部持有的callable任务，运行完毕后置空
//
private Callable<V> callable;

//从get()中返回的结果或抛出的异常
private Object outcome; // non-volatile, protected by state reads/writes

//运行callable的线程
private volatile Thread runner;

//使用Treiber栈保存等待线程
private volatile WaitNode waiters;

//任务状态
private volatile int state;
//新建
private static final int NEW          = 0;
//运行结果未保存到outcome的状态，包括执行完、异常
private static final int COMPLETING   = 1;
//运行结果保存到outcome
private static final int NORMAL       = 2;
//异常保存到outcome
private static final int EXCEPTIONAL  = 3;
//调用cancel
private static final int CANCELLED    = 4;
//cancel并且中断
private static final int INTERRUPTING = 5;
//中断完成
private static final int INTERRUPTED  = 6;
```

## 构造函数

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

//实际runnable 的构造函数会调用适配器把两个参数封装成一个callable
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}

```

## run方法

任务启动方法

```java
public void run() {
        if (state != NEW ||
            //cas将当前线程设置为执行线程
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    //执行任务
                    result = c.call();
                    //执行结束后修改标志位
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    //执行结束后调用
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                //处理中断
                handlePossibleCancellationInterrupt(s);
        }
    }
```

```java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v; //结果输出
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();//执行完毕，唤醒等待线程
    }
}

//主要是唤醒等待结果的线程
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {//移除等待线程
            for (;;) {//自旋遍历等待线程
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);//唤醒等待线程
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    //任务完成后调用函数，自定义扩展
    done();

    callable = null;        // to reduce footprint
}

```

## get

```java


//获取执行结果
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

阻塞结果，如果未完成，则awaitDone进入等待队列



## 调试代码

```java
package cn.yulam.concurrent;

import java.util.concurrent.*;

public class FutureDemo {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
//        ExecutorService executorService = Executors.newCachedThreadPool();
//        Future<Object> future = executorService.submit((Callable<Object>) () -> {
//            Long start = System.currentTimeMillis();
//            while (true) {
//                Long current = System.currentTimeMillis();
//                if ((current - start) > 1000) {
//                    return 1;
//                }
//            }
//        });
//        Integer o = (Integer)future.get();
//        System.out.println(o);

//        Future<?> submit = executorService.submit(new Runnable() {
//            @Override
//            public void run() {
//                Long start = System.currentTimeMillis();
//                while (true) {
//                    Long current = System.currentTimeMillis();
//                    if ((current - start) > 50) {
//                        break;
//                    }
//                    System.out.println(System.currentTimeMillis());
//
//                }
//            }
//        });
//
//        Object o1 = submit.get();
        FutureTask<Integer> futureTask = new FutureTask<>(new MyCallable());
        Thread thread = new Thread(futureTask);
        thread.start();

        if(!futureTask.isDone()) {
            System.out.println("Task is not done");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        int result = 0;
        try {
            // 5. 调用get()方法获取任务结果,如果任务没有执行完成则阻塞等待
            result = futureTask.get();
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("result is " + result);
    }
}

/**
 * 实现 callable  有返回值
 * 需要包装再一个future 执行
 */
class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        Thread.sleep(10000);
        System.out.println("thread name is " + Thread.currentThread().getName());
        return 2;
    }
}

```



# futureTask 与compleableFuture

## 区别

- get()方法一样会阻塞线程，但是compleableFuture支持回调不用阻塞
- completionStage接口支持并行、串行、聚合关系执行，参数传递。future的阻塞获取结果，异步编程麻烦。
- 