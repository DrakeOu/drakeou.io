# JUC，辅助类

- ```CountDownLatch```、```CyclicBarrier```、```Semaphore```底层都是AQS实现的

CountDownLatch实现一个用于倒计时的线程同步工具，主要countDown()方法减少计数，减少到0 await()方法处的线程调用立即返回。一个等多个

CyclicBarrier实现一个加和计数的线程同步工具，初始化时传入一个栏栅数，和可选的回调方法（Runnable），当不同线程讲计数器加到指定值，就调用回调方法（如果有）。回调方法是await()在等待，有一个线程



CountDownLatch不能回调，等待的关键时间点是计数器归零。所以可以用于主线程中，等待其他必要的先序任务完成

CyclicBarrier是另外开启一个线程，在突破屏障时进行回调。不影响主要工作线程，相当于是监听之后的触发操作。

Semaphore控制同时运行存在的资源数量，比较适合在限流时使用。控制运行工作的线程数