生产者，消费者问题本质是不同线程都需求临界区中的资源。为保证线程安全，需要让线程同步操作。

在Java中，对这个问题的实现可以有两种方式：
1. ```synchronized```对代码块同步
**实现如下**：
- 注意将线程和任务进行解耦，单独定义资源类
```
class Data1{
    //资源类，对线程和任务进行分离
    private int number=1;

    public synchronized void increment() throws InterruptedException {
        if(number!=0){
            //判断，并等待
            wait();
        }
        //业务
        System.out.println(Thread.currentThread().getName()+"is running");
        number++;
        //唤醒
        notifyAll();

    }

    public synchronized void decrement() throws InterruptedException {
        if(number!=1){
            //判断，并等待
            wait();
        }
        //业务
        System.out.println(Thread.currentThread().getName()+"is running");
        number--;
        //唤醒
        notifyAll();
    }
}
```
```main```方法
```
public class ProducerConsumer {
    public static void main(String[] args) {
        Data2 data1 = new Data2();

        new Thread(()->{
            for (int i = 1; i<=10; i++){
                data1.increment();
            }
        }, "A").start();

        new Thread(()->{
            for (int i = 1; i<=10; i++){
                data1.decrement();
            }
        }, "B").start();
    }
}
```

2. 第二种方式使用JUC包中的```ReentrantLock```
####使用前将synchronized和ReentrantLock简单对比
-  synchronized锁是隐式的，ReentrantLock显式调用。
- syn代码块中的等待是通过```Object.wait()```实现
- ReentrantLock提供了等价的监视器对象```Condition```
- ```Condition.await() -> Object.wait()```; ```Condition.signalAll() -> Object.notifyAll()```

所以代码可以如下实现：

```main```方法不变
```
//使用Reentrantlock实现, condition.await()和signal()替换wait(),notifyAll()
class Data2{

    private int number = 1;
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void increment(){
        lock.lock();
        try {
            //判断等待
            while (number!=0){
                condition.await();
            }
            number++;
            System.out.println(Thread.currentThread().getName()+"is running, Num is "+number);
            condition.signalAll();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }

    }

    public void decrement(){
        lock.lock();
        try {
            //判断等待
            while (number!=1){
                condition.await();
            }
            number--;
            System.out.println(Thread.currentThread().getName()+"is running");
            condition.signalAll();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}
```

3. **Condition不只是覆盖原本的Object方法，还可以实现多个监听器的指定顺序唤醒**
- 只列出资源类的实现
- 可以看到每个线程将调用的方法均准备一个监视器用于监听
```
final class Data3{
    private int number = 1;
    private Lock lock = new ReentrantLock();
    private Condition condition1 = lock.newCondition();
    private Condition condition2 = lock.newCondition();
    private Condition condition3 = lock.newCondition();

    public void print1(){
        lock.lock();
        try {
            //判断等待
            while (number!=1){
                condition1.await();
            }
            number = 2;
            System.out.println(Thread.currentThread().getName()+"is running, Num is "+number);
            condition2.signal();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }

    }

    public void print2(){
        lock.lock();
        try {
            //判断等待
            while (number!=2){
                condition2.await();
            }
            number = 3;
            System.out.println(Thread.currentThread().getName()+"is running, Num is "+number);
            condition3.signal();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    public void print3(){
        lock.lock();
        try {
            //判断等待
            while (number!=3){
                condition3.await();
            }
            number = 1;
            System.out.println(Thread.currentThread().getName()+"is running, Num is "+number);
            condition1.signal();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

}
```
4. **细节分享**
####实现判断等待的代码一定要使用while循环判断，仅使用if会产生 ***虚假唤醒***
**虚假唤醒**： （不只一个生产者或消费者）考虑多个消费者线程同时在判断处等待， 生产者完成生产操作唤醒所有线程，但实际资源数小于等待线程数，且wait()被唤醒的方法默认获得了锁，继续向下执行消费的代码，那么会导致错误的结果。