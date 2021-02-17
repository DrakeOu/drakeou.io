# AQS

## 我们为什么需要AQS

思考常用的`Lock`类，无论是独占锁，共享锁（或者以其他维度对锁进行划分），但锁的本质都是**利用一个对象来实现对一个公共资源同步状态的控制**。所以`AQS`就是这样一个模板类，包含一个`int`成员变量表示同步状态，和一个控制等待的FIFO队列，提供实现其他锁和同步需求的基础服务。

## AQS中实现自定义同步功能的核心方法

可以重写，应该重写，用于实现具体状态同步工具的`5个`方法：

1. `tryAcquire`
2. `tryRelease`
3. `tryAcquireShared`
4. `tryReleaseShared`
5. `isHeldExclusive`

主要的调用方法，不可变（被`AQS`设计好进行控制）：

1. `acquire`
2. `acquireInterruptibly`
3. `tryAcquireNanos`
4. `release`
5. `acquireShared`
6. `acquireSharedInterruptibly`
7. `tryAcquireSharedNanos`
8. `releaseShared`

上面方法看着很多，其实主要都可以抽象成**两类主要方法**

1. `acquire(Shared)`->`tryAcquire(Shared)` 获取同步变量的方法
2. `release(Shared)`->`tryRelease(Shared)` 释放同步变量的方法

而所说的同步变量，就是`AQS`中自带的`int`变量（用一个唯一的变量表示那把唯一的锁）。

> 那么这些主要方法应该怎么重写？

首先来看看`AQS`的大致运行流程：



可以发现`AQS`![AQS大致原理](..\static\AQS大致原理.png)的核心其实就是  **同步变量的获取** ，**同步变量的释放**以及**获取失败的排队等待**。关于**排队等待**可以事先说明，在队列中的排队是不需要去自己实现的，因为即使是不同的同步工具，排队都是：

- **获取同步资源失败**，才排队
- 在队列头，再次**尝试获取同步资源**

而**获取同步资源**这步操作与排队无关，仅和**同步需求**的实现有关，从队列中排完队，再次尝试即可，所以`AQS`对队列做了固定的实现，不需要自己控制。

所以说回**同步变量的获取和释放**，`tryAcquire(Shared)`方法也就等同于同步需求的实现，观察代码也可以发现，在**尝试**之后失败就进入排队，所以**尝试**==**逻辑上去获得锁**。

``` java
//ReentrantLock中的lock方法，调用了AQS的acquire()
public void lock() {
        sync.acquire(1);
    }
//AQS中的acquire()方法
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

//ReentrantLock中内部类FairSync(extends Sync)的实现
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
    // 忽略下面可重入的实现部分
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

## 基于AQS去设计锁

首先可以看到，当调用`ReentrantLock`的`lock()`时，真正执行的是`AQS`的`acquire()`，而`acquire()`中调用了`ReentrantLock`中实现`FairSync extends Sync`的`tryAcquire()`，失败之后执行了`AQS`的`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`。

用一张图来表示就是这样的。

![利用AQS设计锁](..\static\利用AQS设计锁.png)

我们替换了`AQS`留出的方法，然后整体的`AQS`去控制等待流程。而`Release`或者其他的几个方法都是类似的思路。

## 等待队列

这个FIFO队列是由`Node`节点构成的，`Node`包含如下属性：

- `int waitStatus` 表示等待状态
- `Node prev, next`表示前驱和后继节点
- `Node nextWaiter` 等待队列中的后继节点
- `Thread thread` 记录节点所属的线程信息

而这个队列有几个特殊点：

1. 队列头节点 不是等待节点，表示正在运行地，拥有同步状态的那个节点。
2. 每个节点入队是通过`CAS`+死循环 保证线程安全
3. 每个队列中的节点 都在 自旋 去检查**前驱节点是否是头节点**（`release`, `shared`, `nanos`都是如此）

所以用一个示意图表示这个FIFO队列如下：

![AQS中的同步队列](..\static\AQS中的同步队列.png)

## 可重入的实现

那么在了解了`AQS`是如何使用，以及`AQS`内部的队列是如何实现之后。考虑`Lock`要提供可重入性。

实际上在`AQS`源码中可以发现这么一段：

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    ...
}

//AbstractOwnableSynchronizer
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private static final long serialVersionUID = 3737899427754241961L;

    protected AbstractOwnableSynchronizer() { }

    private transient Thread exclusiveOwnerThread;

    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

可以发现就是这个`AbstractOwnableSynchronizer`提供了检查线程是否是独占线程的功能（记录和锁对象绑定的线程）

那么再看`ReentrantLock`后半段检查可重入就明白了：

```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
    // 检查 当前线程 是否是 锁的独占线程
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

## 其他

其实`AQS`中还有另外一个重要的内部实现类，`Condition`用来实现监视器的功能，这个就下篇再聊吧。