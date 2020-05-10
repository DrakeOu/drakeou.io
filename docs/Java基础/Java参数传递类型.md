## Java参数传递类型

**明确要点**，Java中只存在按值传递。

实参到形参传递的不同效果实际需要结合，变量的**类型**和**作用域**来看

> Java数据类型的划分

![java数据类型](..\static\java数据类型.png)

**注意**：数组在Java中是被视为对象的，内部细节应该是依赖于JVM去实现了，这里不做讨论。

> 局部变量中的按值传递

```java
public static void main(String[] args) {
    String str = "a";
    test(str, 10);
    System.out.println(str);
}
private static void test(String str, int time){
    if(time > 0){
        str += "a";
        test(str, time-1);
    }
}
```

这段代码将`str`递归传入`test`方法并进行拼接，最终结果输出`a`，`main`方法中的`str`并未被修改。

- (`String`可以换成任何其他基本类型，及其包装类，结果都是一样的。)

而栈中的内存情况如下：

![在栈中的局部变量](..\static\在栈中的局部变量.png)

可以看到，如果变量仅存在于方法中（是个**局部变量**），同时是基本类型（**包括其包装类**），那么方法调用时，只是将其值复制给了形参。

> 局部变量中的”引用传递“

如果方法中的变量是一个实例化的对象（强调调用了`new`方法创建的，因为`String`也是对象，但是并不在堆中分配内存空间）。也可以说，如果基本类型是某一类的属性，那么只有实例化后才存在，所以这种基本类型存在于堆中。



看看下面的代码：

```java
public static void main(String[] args) {
    Point point = new Point(1, 1);
    change(point);
    System.out.println(point.x+" "+point.y);
}
private static void change(Point point){
    point.x = 10;
    point.y = 10;
}

 static class Point{
    public int x;
    public int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

最后结果`point`会被成功修改，包含堆内存图如下

![实例化对象的传递](..\static\实例化对象的传递.png)

方法传递时，是将new出来对象的内存地址传递了（**是一份复制，不是原来的那个引用**）。这样看起来Java好像存在引用传递，但是如果让方法中Point指向新的对象，原来的对象并不会发生修改。

```java
public static void main(String[] args) {
        Point point = new Point(1, 1);
        change(point);
        System.out.println(point.x+" "+point.y);
    }
    private static void change(Point point){
        point = new Point(10, 10);
//        point.x = 10;
//        point.y = 10;
    }

     static class Point{
        public int x;
        public int y;

        public Point(int x, int y) {
            this.x = x;
            this.y = y;
        }
    }
```

输出结果是1，1  也就是传递的地址值是**一份复制**，对它的修改不影响原来的对象。

![实例化对象的传递(重新赋值)](..\static\实例化对象的传递(重新赋值).png)

> 稍微总结一下

根据开头的图，其实不用讨论变量是否是类的属性，因为需要区分是**值传递**还是**”引用“传递时**一定是在方法栈中。所以区分的最好方式是判断变量是否是一个引用变量即可（关注是否是`new`出来的对象），如果是那么一定是"引用"，否则仅在局部变量表中，就是值传递了。