## Synchronized+Spring事务 == 线程不安全？？

某日进行多线程实践时，突发奇想将`@Transactional`注解的spring事务方法用`synchronized`进行了修饰。

那么理论上，事务方法指向时应当是线程安全的。然而在执行如下方法时，最终生成的订单数>商品的剩余数量。

```java
@Service
public class SecKillServiceImpl implements SecKillService {

    @Autowired
    SecKillMapper mapper;

    @Transactional
    @Override
    public synchronized void SecKillProduct(String productId) {
        //查询剩余库存
        Product product1 = mapper.queryProduceById(productId);
        if(product1.getStock()>0){
            product1.setStock(product1.getStock()-1);
            //更新库存
            mapper.SecKillProduct(product1);
            //创建对应订单
            mapper.CreateOrder(product1.getProductId(), UUID.randomUUID().toString().substring(0, 25));

        }
    }
}
```

如上方法被`synchronized`修饰，那么多个不同的线程进来不是应该排队串行执行mapper方法？

结论显然不是。

`mapper`和`controller`应该也没有问题

```xml
<mapper namespace="com.ed.concurrency.cdemo.mapper.SecKillMapper">
    <update id="SecKillProduct">
        update product_list
        set stock = #{stock}
        where id = #{productId}
    </update>

    <resultMap id="productMap" type="com.ed.concurrency.cdemo.POJO.Product">
        <result column="id" property="productId"/>
        <result column="stock" property="stock"/>
    </resultMap>
    <select id="queryProduceById" resultMap="productMap">
        select id, stock from unamed.product_list where id = #{id}
    </select>

    <insert id="CreateOrder">
        insert into unamed.order_list (produce_id, order_id) values (#{productId}, #{orderId})
    </insert>
</mapper>
```

Controller

```java
@RestController
public class SecKillController {

    @Autowired
    SecKillService service;

    @RequestMapping("/seckill/{id}")
    public String SecKill(@PathVariable("id") String productId){
        try{
            service.SecKillProduct(productId);
        }catch (Exception e){
            e.printStackTrace();
            return "抱歉商品已经售完";
        }
        return "商品秒杀成功";
    }

}
```

### Synchronized应该是安全的

> `@Transactional`里面的syn并不能给整个事务加锁

`synchronized`加在了这个方法上，但不是事务上。

为了说明这点需要先明白spring的事务是如何实现的。

Spring事务的底层是Spring AOP, 而Spring AOP的底层是动态代理实现。

回顾如下代码：

```java
public static void main(String[] args) {

    // 目标对象
    Object target ;

    Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), Main.class, new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            // 但凡带有@Transcational注解的方法都会被拦截
			
            
			try{
                // 1... 开启事务
                method.invoke(target);
                // 2... 提交事务
            }catch(Exception e){
                //异常处理
            }finally{
                //最终处理
            }
            return null;
        }
        
    });
}
```
不难发现，我们所添加的`synchronized`关键字实际作用在`method.invoke(target)`这行代码上。而整个动态代理过程却不是线程安全的。

- 同时，另外一个重要的点是：Spring的事务是执行成功后，才一次性写入到Mysql中（或其他数据库）
- 所以注意区分Spring中的事务和Mysql中的事务，Spring的事务是完全在Spring中控制的，与Mysql的事务没有任何关系

那么多线程下，不安全的情况就出现了：**如果被事务修饰的方法执行完了`synchronized`方法，但还未提交事务（也就是没有写入到数据库中），而此时其他的线程进入`synchronized`方法就获取到了实际上已经过时的数据。**

### 问题如何解决

知道了声明式事务修饰的方法是线程不安全的，那么在调用`service`出添加`synchronized`同步（也就是controller层）就可以解决这里线程不安全的问题。

```java
@RequestMapping("/seckill/{id}")
public String SecKill(@PathVariable("id") String productId){
    try{
        synchronized (SecKillService.class){
            service.SecKillProduct(productId);
        }
    }catch (Exception e){
        e.printStackTrace();
        return "抱歉商品已经售完";
    }
    return "商品秒杀成功";
}
```

当然，正常的并发控制应该不会这么解决。







奇怪的知识点增加了。



