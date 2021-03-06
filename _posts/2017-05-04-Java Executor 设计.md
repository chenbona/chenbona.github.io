---
layout: post
title: Java Executor 设计
date: 2017-05-04 
tag: 多线程
---

　　写这个post主要是总结一下，从拿到一个问题到并发实现的思路和模式。


### Step1 找出适合并行化的子问题
　　举个例子，某个风险控制业务系统每天在夜深人静的时候，都会触发一次批量计算，需要按顺序先更新客户截止今天的实际贷款额度，再计算客户当前的贷款额度汇总。从整个流程来看，这个流程只能串行执行，因为第二步得等第一步完成才能开始执行。但是第一步和第二步本身都可以考虑并行化。

### Step2 分析子问题
　　考虑并行化计算客户的贷款额度汇总。考虑到计算客户的贷款额度这个任务，不同客户之前不会互相影响到计算结果，也不会消耗很长的计算时间，也就是说，要并行化的计算任务，刚好满足同构，短耗时的特点，正好适用线程池来实现。如果是异构的任务，看具体情况，可以考虑使用多个线程池。

　　分析了任务的特点之后，还需要确定并行化的执行策略。假设业务要求，必须等所有计算任务执行完之后，才能进入下一个处理步骤。那么执行策略应该是并行执行任务，等待直到所有计算任务完成，再返回。

### Step3 设计Executor
　　使用Java的Executor框架来实现。

　　首先需要设计线程池的大小。要想正确地设置线程池的大小，必须分析任务的特性，计算环境，资源预算。任务是计算密集型的还是IO密集型的，还是两者都有？是否需要连接数据库？部署的系统有多少个CPU？多大的内存？预算的最大CPU利用率是多少？预算的最大内存利用率是多少？

　　当任务需要某种通过资源池来管理的资源时，例如数据库连接，那么线程池的大小和资源池的大小将会互相影响。

　　这些方面的考虑，落到实现上来，就是对ThreadPoolExecutor的配置。

```
ThreadPoolExecutor的构造函数：
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {...}
```
#### 线程池的大小
　　线程池的基本大小（corePoolSize），最大大小（maximumPoolSize）,存活时间（keepAliveTime）共同负责线程的创建和销毁。基本大小就是没有任务执行时线程池的大小，最大大小表示可同时活动的线程数量的上限。如果某个线程的空闲时间超过了存活时间，那么这个线程将被标记为可回收的。如果当前线程池的大小超过了基本大小，这个线程将被终止。当有新的任务要处理的时候，如果当前线程池的大小超过了基本大小，但不超过最大大小，而且没有空闲线程，Executor将会创建新的线程。

#### 任务队列
　　利用任务队列来限制可并发执行的任务数量，防止系统资源被耗尽。
基本的任务队列排队方法有三种：无界队列，有界队列和同步移交。队列的选择与其他配置参数有关，例如线程池的大小。如果线程池较小而队列较大，那么有助于减少内存使用量，降低CPU使用率，减少线程上下文切换，但代价是有可能会限制吞吐量。对于非常大或者无界的线程池，可以假设任务基本不需要排队，可以通过使用SynchronousQueue来避免任务排队。

#### 队列饱和策略
　　当队列被填满之后，或者某个任务被提交到一个已经被关闭的Executor的时候，饱和策略开始发挥作用。

　　JDK提供了四种不同的RejectedExecutionHandler实现，分别包含不同的饱和策略：AbortPolicy（中止），CallerRunsPolicy（调用者运用），DiscardPolicy（抛弃），DisCardOldersPolicy（抛弃最旧的）。
中止策略是默认的饱和策略，该策略将抛出未检查的RejectedExecutionException.调用者运行策略实现了一种调度机制，将任务退回给Executor的调用者，在调用线程中执行该任务。这样做的好处是，主线程至少在一段时间内不能提交任务，从而使得线程池的工作线程有时间处理完正在处理的任务。

　　回到上面作为例子的计算问题，假设计算结果需要保存到数据库，数据库连接池有10个连接线程，部署环境是4核CPU，8G内存，并行计算的任务基本上不超过1000个，那么可以初步设计，让基本大小为10，最大大小为20，队列大小为1000。然后在这个基础上，通过一些分析或监控工具，获得系统资源使用情况和运行性能指标，进一步调优。

### Step4 实现执行策略和计算逻辑
　　利用Future.get()的阻塞特性，实现阻塞主线程直到所有任务执行完毕。

```
代码示例
public void waitForAllTasksOver(List<Future<String>> futureList) {
   for (Future<String> f : futureList) {
      String result = null;
      try {
         result = f.get();
      } catch (InterruptedException e1) {
         logger.error("Catch InterruptedException while fetching result caused by {}", e1.getMessage());
         f.cancel(true);
      } catch (ExecutionException e2) {
         logger.error("Catch ExecutionException while fetching result caused by {}", e1.getMessage());
         f.cancel(true);
      }
   }
}
```

### Step5 关闭Executor
　　Executor拓展了ExecutorService接口，添加了一些管理生命周期的方法。

　　ExecutorService的生命周期有三种状态：运行，关闭和终止。
　　- ExecutorService创建后处于初始状态。
　　- ExecutorService提供两个关闭方法：shutdown()和shutdownNow()。shutdown方法将执行平缓的关闭过程：不再接受新的任务，同时等待已经提交的任务执行完成。shutdownNow方法将执行粗暴的关闭过程：尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务，并返回后者。
　　- 等所有任务都完成后，ExecutorService将转入终止状态。

```
代码示例
public void shutdownExecutor(ExecutorService executor, Long awaitSeconds) {
   executor.shutdown();

   try {
      if (!executor.awaitTermination(awaitSeconds, TimeUnit.SECONDS)) {
         logger.warn("Timed out while waiting for executor to terminate");
      }
   } catch (InterruptedException e) {
      executor.shutdownNow();
   }
}
```

### To be continue: 性能与可伸缩性，正确性测试，性能测试










