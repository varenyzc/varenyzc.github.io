---
title: 线程池中execute和submit的区别
date:  2020-11-01 18:22:39 +0800
category: develop 
tags: Android
excerpt: 工作中bug引出的思考
---

在前阵子的工作中，收到一个bug，大概是在app运行过程中出现了异常，可是查看了app日志、系统日志并没有异常日志打印。最后发现了context.getExternalCacheDir方法返回值为null，原以为bug可以回给系统组的人去修了，在leader的指导下，发现了app中的问题：线程池启动线程的方法为submit，而没有对submit的返回值做处理，在运行过程中也没有catch NPE，任务一直在线程池中错误循环，所以导致了问题，最终将所有线程池的submit方法替换为execute方法，当有未catch的execption时，会直接crash，在系统中打印日志，方便后续的bug处理。

下面说说execute和submit的区别。

首先从源码开始
**submit:**
```java
public interface ExecutorService extends Executor {
　　...
　　<T> Future<T> submit(Callable<T> task);

　　<T> Future<T> submit(Runnable task, T result);

　　Future<?> submit(Runnable task);
　　...
}
```
通过源码，submit来源于ExecutorService接口，可以看出，在ExecutorService接口中，一共有以上三个sumbit()方法，入参可以为Callable<T>，也可以为Runnable，而且方法有返回值Future<T>；
*(补充说明：Callable<T>与Runnable类似，也是创建线程的一种方式，实现其call()方法即可，方法可以有返回值，而且方法上可以抛出异常)*

**execute:**
execute方法来源于Executor接口，上述的ExecutorService接口继承于此，Executor是最上层的，其中只包含一个execute()方法：
```java
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

**总结：**
**（1）可以接受的任务类型不同**
execute只能接受Runnable类型的任务
submit不管是Runnable还是Callable类型的任务都可以接受，但是Runnable返回值均为void，所以使用Future的get()获得的还是null

**（2）submit()有返回值，而execute()没有**
例如，有个task，希望该task执行完后告诉我它的执行结果，是成功还是失败，然后继续下面的操作，这时需要用submit

**（3）submit()可以进行Exception处理**
例如，如果task里会抛出checked或者unchecked exception，而你又希望外面的调用者能够感知这些exception并做出及时的处理，那么就需要用到submit，通过对Future.get()进行抛出异常的捕获，然后对其进行处理。




