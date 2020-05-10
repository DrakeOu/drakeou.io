**一句话攻略**：3个方法，7种属性，4种策略 （来自b站up：狂神说Java）
## 使用Executors获得线程池, `3个方法`（不推荐使用，也是阿里开发手册不建议的方式）

1. ```Executors.newSingleThreadExecutor()```
        获取只有一个线程构成的线程池
```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
2. ```Executors.newFixedThreadPool(int nThread)```
获取指定数量的线程池
```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
3. ```Executors.newCachedThreadPool()```
获取一个理论最大2^31-1的线程池，按需求可以一直增加线程。这也是不推荐使用的原因，计算机并不能承载理论大小的线程数量，会导致OOM。
```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
---
简单观察可知以上方法均简单封装了```new ThreadPoolExecutor()```，所以线程池的核心仍是`ThreadPoolExecutor`。上述方法不被推荐是因为Executors进行创建提供的参数太少，没有进行控制的参数太多，不利于使用也不利于优化。

下面以```ThreadPoolExecutor```为例分析线程池的运行原理
-
7个属性指```ThreadPoolExecutor```共有`7`个主要属性
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
从完整的构造方法来看
- ```@param corePoolSize``` 线程池的**核心**线程数
- ```@param maximumPoolSize``` 线程池的**最大**线程数
- ```@param keepAlive```  **非核心**线程**不工作状态**的最大存活时间
- ```@param unit ```  keepAlive的**时间单位**
- ```@param workQueue```  线程池中任务排队的阻塞队列
- ```@param handler```  拒绝任务时的具体策略
- ```@param threadFactory```  使用的线程构造工厂（一般不改变）

对于理解线程池**工作原理**而言最重要的是 **核心线程数**，**最大线程数**，和不工作状态的**存活时间**。
~~此处应有图~~
简述运行过程：
1. 通过```.execute(task)```方法添加任务到线程池，```task```是一个实现```run```方法的```Runnable```对象
2. 根据当前线程池的运行状态：
      1. 当前线程数不足核心线程数： 创建核心线程并将任务添加到Worker
      2. 当核心线程已满，而队列未满时。将任务添加到队列进行等待
      3. 当核心线程和等待队列都满时，如果线程数小于最大线程数，开启临时线程处理
      4. 临时线程在最大存活时间内都没有运行，就销毁线程
      5. 已达到最大线程数，队列已满，任务无法添加只能执行拒绝策略

## 4种策略
其实所指的是拒绝策略。
![RejectedExecutionHandler的实现类](https://upload-images.jianshu.io/upload_images/20009308-6a00cf191e5b9d02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

初始化传入的拒绝策略是`RejectedExecutionHandler`，上述是全部实现类。
代码样例：
```
public static void main(String[] args) {
        ExecutorService service = new ThreadPoolExecutor(
                2, //核心线程数
                5, //最大线程数
                1, //超时时长
                TimeUnit.SECONDS,  //时长单位
                new ArrayBlockingQueue<>(3),  //定长阻塞队列
                new ThreadPoolExecutor.CallerRunsPolicy());  //拒绝策略

        for (int i = 1; i <= 10 ; i++) {
            try {
                service.execute(()->{
                    System.out.println(Thread.currentThread().getName()+"-------OK");
                });
            }catch (Exception e){
                e.printStackTrace();
            }

        }
        service.shutdown();
    }
```
可以看到允许的策略分别是：
1. `AbortPolicy`默认策略，当队列满时抛出`java.util.concurrent.RejectedExecutionException`
2. `CallerRunsPolicy`谁调用谁执行（原本主线程将任务试图将任务添加到线程池执行，该策略拒绝后，由主线程自己执行任务）
3. `DiscardPolicy`抛弃当前任务
4. `DiscardOldestPolicy`当前任务和队列中加入最早的任务竞争

## 调优相关
- CPU密集型
  需要让CPU能充分运行，所以设置**最大线程数**>=CPU核数
- IO密集型
  IO操作不占用CPU，但长时间占用线程资源。导致CPU空闲但没有线程可以用于执行任务。所以**最大线程数**可以设置为IO线程数的**两倍**