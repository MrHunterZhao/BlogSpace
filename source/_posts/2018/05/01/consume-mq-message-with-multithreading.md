---
title: 聊一聊使用多线程处理MQ消息的正确姿势
date: 2018-05-01 03:27:18
categories: 多线程
tags:
    - 多线程
    - MQ
---

![openjdk](/images/post/2018/05/01/rabbitmq-logo.png)

> Q：有这样一个场景，MQ的生产者生产消息能力是消费者的数倍。
> 如果不能尽快消费完会导致队列中的消息随着时间的推移会越积越多，而且业务也无任何时效性可言，
> 那么问题来了，在不增加消费节点的前提下如何快速处理完消息以保证吞吐量？

面对以上问题，有人可能会信心满满地脱口而出：**用多线程**。OK，我只能说思路没错，那么如何落地呢？

<!-- more -->
你可能会说使用JDK的`ThreadPoolExecutor`或者`Executors`线程池来处理。具体如何去用？只用线程池就够了吗？有什么坑吗？
带着这些疑问，我们分别来深入分析一下看看是否能满足需求。


# ThreadPoolExecutor()
```java
    private void exec() {

        /*
         * 创建线程池
         * corePoolSize 10
         * maximumPoolSize 10
         * queue capacity 10
         */
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            10, 10, 60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(10));

        // 提交100个任务
        for (int i = 0; i < 100; i++) {
            threadPoolExecutor.submit(new WorkerThread());
        }

    }

    /**
     * 任务线程
     */
    private class WorkerThread implements Runnable {

        public void run() {
            System.out.println(new Date() + " " + Thread.currentThread().getName() + " is running...");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
```

执行以上程序会抛出错误，原因是因为submit的任务数已经超出了其queue的容量，导致触发了拒绝策略。
```
Exception in thread "main" java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.FutureTask@4d7e1886
```
> Q: 能不能把有界队列`ArrayBlockingQueue`的容量尽量设置大一些呢？或者干脆换成无界队列`LinkedBlockingQueue`？

1. 由于队列中的消息数量无法提前预测具体数量，所以无法使设置准确默认值。
2. 换成`LinkedBlockingQueue`虽然不会抛出以上错误，但这里会有一个问题：N多待处理任务临时放在JVM中，一方面占用大量内存，另一方面如果服务重启就会导致大量任务丢失。

> 既然无法从queue容量上去解决，那能否从线程池的拒绝策略着手？

JDK线程池有四种拒绝策略`AbortPolicy`、`CallerRunsPolicy`、 `DiscardOledestPolicy`、`DiscardPolicy`，默认使用AbortPolicy直接抛出异常。

其中`AbortPolicy`，`DiscardOledestPolicy`、`DiscardPolicy`均会直接丢弃任务肯定不符合预期，
`CallerRunsPolicy`是将该任务抛给主线程执行，这里也会有一个问题，主线程在执行任务时是无法向线程池中提交任务的，
假如主线程执行该任务需要3秒，在执行至第1秒的时候，线程池中已经有若干工作线程处于闲置状态，此时主线程需要执行完剩余的2秒才能继续向线程池工作线程分配任务，
使用该拒绝策略虽然不会导致消息丢失，但也不能达到资源最优利用，所以pass。

所以，问题解决方案基本浮出水面，
1. 主线程只负责分配任务，Worker线程只负责执行任务。
2. Worker线程执行完成后第一时间通知主线程，然后主线程及时分配任务。

这里，我们引入`Semaphore`作为令牌桶，以达到主线程和工作线程间通信的目的。

1. 初始化与Worker线程数量相同的令牌
2. 主线程向线程池提交任务时先尝试从令牌桶获取一个令牌，如果令牌桶为空则block。
3. 把令牌传入Worker线程，Worker线程执行完后调用`release()`归还令牌。

最终代码参见以下：
```java

    /* 初始化与Worker线程数量相同的令牌 */
    private Semaphore permits = new Semaphore(10);

    private void exec() throws InterruptedException {

        /*
         * 创建线程池
         * corePoolSize 10
         * maximumPoolSize 10
         * queue capacity 10
         */
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            10, 10, 60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(10));

        for (int i = 0; i < 100; i++) {
            // 获取令牌，如果获取不到则block，直到有worker线程归还
            permits.acquire();
            // 提交任务
            threadPoolExecutor.submit(new WorkerThread(permits));
        }
    }

    /**
     * 任务线程
     */
    private class WorkerThread implements Runnable {

        private Semaphore permits;

        public WorkerThread(Semaphore permits) {
            this.permits = permits;
        }

        public void run() {
            System.out.println(new Date() + " " + Thread.currentThread().getName() + " is running...");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            finally {
                if (permits != null) {
                    // 释放令牌
                    permits.release();
                }
            }

        }
    }

```


