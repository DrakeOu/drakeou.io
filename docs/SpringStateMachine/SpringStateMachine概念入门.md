

## Spring StateMachine概念入门

### 有限状态机的概念

有限状态机是一种用来进行对象行为建模的工具，主要作用是用来描述其在生命周期中的状态转换，以及响应导致状态转换的事件。

### 为什么需要状态机

因为对象是行为建模是必须且必要的，否则部分复杂系统的开发会沦落到面向数据库开发或者面向业务需求开发，对系统不够整体的抽象都会导致最终系统扩展性差，难以维护，重构代价大，想要继续开发只能往屎山上堆屎。而避免这种情况并非需要开始时就设想出全部的可能场景，需要的是一套可以方便扩展且易于维护的框架。

而状态机将整个系统的业务流程抽象成一个具有多个状态且可以相互转换的生命周期，那么业务场景就自然的和状态进行了绑定，同时由于状态的存在，对于场景的判断和事件的发生有了更加明确的原则。**事件绑定在了状态之间，而状态之间的转换被事件所描述**

### Spring StateMachine相关概念

#### 状态机的定义

1. `StateMachine`：状态机模型
2. `state`: 状态
3. `event`: 事件

上述的状态和事件一般都会定义成枚举类

#### 相关概念

- `Transition`: 节点，定义状态的核心
- `source`：节点当前状态
- `target`：节点的目标状态
- `event`：状态转换时的触发动作
- `guard`:  校验状态，用于决定是否执行后续action
- `action`: 当前节点的业务逻辑

#### 例子

```java
   builder.configureTransitions()  -- 配置节点
            .withExternal()   //表示source target两种状态不同
            .source(CREATE)  //当前节点状态
            .target(WYD_INITIAL_JUMP)  //目标节点状态，这里是设置了个中间状态
            .event(BizOrderStatusChangeEventEnum.EVT_CREATE)  //导致当前变化的动作/事件
            .action(orderCreateAction, errorHandlerAction) //执行当前状态变更导致的业务逻辑处理，以及出异常时的处理
       		.and()  // 使用and串联
            .withChoice() // 使用choice来做选择
            .source(WYD_INITIAL_JUMP) // 当前状态
            .first(WAIT_REAL_NAME_AUTH, needNameAuthGurad(), needNameAuthAction)  // 第一个分支
            .last(WAIT_BORROW, waitBorrowAction) // 第二个分支
```

几点说明：

- `withExternal/withInternal/withChoice`分别表示`source,target`不同，相同，以及一对多个可能性时。所以可以看出节点的配置是基于状态转移的。
- 而`source, target`则分别表示当前状态和目标状态
- `and`用于串联多个配置节点

### 写着看看

1. 首先按照规则，定义好状态枚举和事件枚举

   ```java
   /**
   * 状态枚举
   **/
   public enum States {
       DRAFT,
       PUBLISH_TODO,
       PUBLISH_DONE,
   }
   
   /**
   * 事件枚举
   **/
   public enum Events {
       ONLINE,
       PUBLISH,
       ROLLBACK
   }
   ```

2. 进行状态机的配置，将上述定义的状态及转换体现到状态机中

   ```java
   @Configuration
   @EnableStateMachine
   public class StateMachineConfig extends EnumStateMachineConfigurerAdapter<States, Events> {
   
       @Override
       public void configure(StateMachineStateConfigurer<States, Events> states) throws Exception {
           states.withStates().initial(States.DRAFT).states(EnumSet.allOf(States.class));
       }
   
       @Override
       public void configure(StateMachineTransitionConfigurer<States, Events> transitions) throws Exception {
           transitions.withExternal()
               .source(States.DRAFT).target(States.PUBLISH_TODO)
               .event(Events.ONLINE)
               .and()
               .withExternal()
               .source(States.PUBLISH_TODO).target(States.PUBLISH_DONE)
               .event(Events.PUBLISH)
               .and()
               .withExternal()
               .source(States.PUBLISH_DONE).target(States.DRAFT)
               .event(Events.ROLLBACK);
       }
   }
   ```

   **注意**：上面仅仅定义了抽象状态的转换，事件也只是抽象定义的，具体的业务方法在下面实现，并且`spring`会根据状态将其对应起来

3. 定义一个业务对象，实现状态转义方法，会映射到上面定义的状态中

   ```java
   @WithStateMachine
   @Data
   @Slf4j
   public class BizBean {
   
       /**
        * @see States
        */
       private String status = States.DRAFT.name();
   
       @OnTransition(target = "PUBLISH_TODO")
       public void online() {
           log.info("操作上线，待发布. target status:{}", States.PUBLISH_TODO.name());
           setStatus(States.PUBLISH_TODO.name());
       }
   
       @OnTransition(target = "PUBLISH_DONE")
       public void publish() {
           log.info("操作发布,发布完成. target status:{}", States.PUBLISH_DONE.name());
           setStatus(States.PUBLISH_DONE.name());
       }
   
       @OnTransition(target = "DRAFT")
       public void rollback() {
           log.info("操作回滚,回到草稿状态. target status:{}", States.DRAFT.name());
           setStatus(States.DRAFT.name());
       }
   
   }
   ```

4. 手动触发几个事件方法

   ```java
   public class StartupRunner implements CommandLineRunner {
   
       @Resource
       StateMachine<States, Events> stateMachine;
   
       @Override
       public void run(String... args) throws Exception {
           stateMachine.start();
           stateMachine.sendEvent(Events.ONLINE);
           stateMachine.sendEvent(Events.PUBLISH);
           stateMachine.sendEvent(Events.ROLLBACK);
       }
   }
   ```

真正的生产环境中应该会利用监听器做出响应式的触发，并不需要手动触发，实现了业务逻辑和状态转换的解耦。框架完成了建模以及抽象工作，提供了对应的工具方便使用，唯一需要做的就是结合框架在正确的地方写下自己的业务代码。