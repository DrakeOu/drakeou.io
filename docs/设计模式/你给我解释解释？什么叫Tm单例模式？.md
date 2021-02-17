## 你给我解释解释？什么叫Tm单例模式？

> 饿汉式

顾名思义：在类加载时就进行单例对象的创建，十分简单。

代码如下：

```java
public class Hungry {
    // 饿汉式单例

    private Hungry(){
		// 模拟类中属性
        byte[] data1 = new byte[1024];
        byte[] data2 = new byte[1024];
        byte[] data3 = new byte[1024];
    }
    private static Hungry hungry = new Hungry();

    public static Hungry getInstance(){
        return hungry;
    }
}

```

存在的问题是显而易见的：如果类始终不需要使用，单例对象还是会一直占用内存资源



> 懒汉式

仅在需求单例对象时才进行创建，把对象的创建时机延后了。

```java
public class LazyMan {
    // 懒汉式
    private LazyMan(){
        System.out.println(Thread.currentThread().getName()+" is running constructor");
    }

    private static LazyMan lazyMan = null;

    private static LazyMan getInstance(){
        if (lazyMan == null){
            lazyMan = new LazyMan();
        }
        return lazyMan;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(LazyMan::getInstance).start();
        }
    }

}
```

但是这种方式很容易看出在多线程中是不安全的。

考虑多个线程同时调用`getInstance()`方法并成功进入了`if`的条件中，就会出现多个对象。

​		Thread-0 is running constructor
​		Thread-1 is running constructor



> DCL懒汉式

使用双重判断，并加锁的方法。

样例：

```java
public class DCLLazy {
    //双重检测

    private DCLLazy(){
        System.out.println(Thread.currentThread().getName()+" is running constructor");
    }

    private static DCLLazy dclLazy = null;

    private static DCLLazy getInstance(){
        if (dclLazy == null){
            synchronized (DCLLazy.class){
                if (dclLazy == null){
                    dclLazy = new DCLLazy();
                }
            }
        }
        return dclLazy;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                getInstance();
            }).start();
        }
    }
}
```

- 第一次判断是**判断当前是否有对象**，这里并不会有线程安全问题。（如果有，多个线程都返回对象，如果没有多个线程进入内部尝试创建对象）
- 加锁确保只会有一个线程进入去创建对象
- 第二次判断，因为除了成功创建对象的那个线程，其他线程都在阻塞等待。而等待到了锁也没有创建的必要了，为了防止多次创建这里再进行一次判断

> 静态内部类

```java
public class SLazy {
    //静态内部类

    private SLazy(){
        System.out.println(Thread.currentThread().getName()+" is running constructor");
    }

    private static class SLazyHolder{
        private static final SLazy instance = new SLazy();
    }

    public static SLazy getInstance(){
        return SLazyHolder.instance;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                getInstance();
            }).start();
        }
    }
}
```

DCL和静态内部类的特点是类似的，保证了线程安全，会受指令重排影响

其实这样的DCL和静态内部类也还是有问题的，问题在于**指令重排序**。

真正安全的单例方式应该对单例对象加上`volatile`关键字。（这是`volatile`**禁止指令重排**的重要应用，务必记住）

`new`方法并不是原子操作。大致有以下三步

1. 申请内存空间
2. 调用构造器方法
3. 将对象指向内存

而实际上因为指令重排的原因，步骤2，3顺序可能颠倒。导致对象指向了一个没有成功构造的内存。而其他线程会认为已经构造，导致调用没有构造的对象。

> 反射破解

使用如下代码就可以跨过`getInstance()`方法直接构造

```java
public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    Class clazz = DCLLazy.getInstance().getClass();
    Constructor declaredConstructor = clazz.getDeclaredConstructor(null);
    declaredConstructor.setAccessible(true);

    DCLLazy instance1 = (DCLLazy) declaredConstructor.newInstance();
    DCLLazy instance2 = (DCLLazy) declaredConstructor.newInstance();
    System.out.println(instance1);
    System.out.println(instance2);
}
```

```java
main is running constructor
main is running constructor
main is running constructor
com.baidu.ai.aip.auth.single.DCLLazy@74a14482
com.baidu.ai.aip.auth.single.DCLLazy@1540e19d


```

可以看到对象被构建了三次

---

那么对构造函数进行如下升级

```java
private DCLLazy(){
    synchronized (DCLLazy.class){
        if(dclLazy != null){
            throw new RuntimeException("请不要使用反射进行创建");
        }
    }
    System.out.println(Thread.currentThread().getName()+" is running constructor");
}
```

再次尝试

```java
main is running constructor
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.baidu.ai.aip.auth.single.DCLLazy.main(DCLLazy.java:37)
Caused by: java.lang.RuntimeException: 请不要使用反射进行创建
	at com.baidu.ai.aip.auth.single.DCLLazy.<init>(DCLLazy.java:13)
	... 5 more

```

可以看到这种方式已经行不通了。

但如果不通过单例对象获取字节码，直接通过字节码得到构造方法进行构造，这种方式仍然是会被破解的。

```java
public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    // Class clazz = DCLLazy.getInstance().getClass();
    Constructor declaredConstructor = DCLLazy.class.getDeclaredConstructor(null);
    declaredConstructor.setAccessible(true);

    DCLLazy instance1 = (DCLLazy) declaredConstructor.newInstance();
    DCLLazy instance2 = (DCLLazy) declaredConstructor.newInstance();
    System.out.println(instance1);
    System.out.println(instance2);

}	 
```

可以看到成功创建了两个对象

```java
main is running constructor
main is running constructor
com.baidu.ai.aip.auth.single.DCLLazy@74a14482
com.baidu.ai.aip.auth.single.DCLLazy@1540e19d
```

总结：无论是增加判断，锁，还是私有属性进行检查，都没办法阻止反射通过获取私有构造器，并直接构造。



> 枚举类型

枚举类型相比之前的方式最大的优点在于不会被反射破解。

```java
public enum EDemo {

    INSTANCE;
    
    public EDemo getInstance(){
        return INSTANCE;
    }
}
```