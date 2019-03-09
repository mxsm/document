### 1. 什么是EventLoop?

运行任务来处理在连接的生命周期内发生的事件是任何网络框架的基本功能。与之相应的编程上的构造通常被称为事件循环。这就是 **`Netty`** 中的 **`EventLoop`**

### 2. EventLoop接口

**`EventLoop`** 是协同设计的一部分采用了两个基本的API:

- **并发**

  **`io.netty.util.concurrent`** 包构建在 JDK 的 **`java.util.concurrent`** 包上，用来提供线程执行器

- **网络编程**

  **`io.netty.channel`** 包中的类，为了与 **`Channel`** 的事件进行交互，扩展了接口/类

![图解](https://github.com/mxsm/document/blob/master/image/netty/EventLoop%E7%9A%84%E7%B1%BB%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84%E5%9B%BE.jpg?raw=true)

```java
 //netty 4.1.17版本
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup {
    @Override
    EventLoopGroup parent();
}
```

在这个模型中，一个 **`EventLoop`** 将由一个永远都不会改变的 **`Thread`** 驱动，同时任务 (**`Runnable`** 或者 **`Callable`**)可以直接提交给 **`EventLoop`** 实现，以立即执行或者调度执行。 根据配置和可用核心的不同，可能会创建多个 EventLoop 实例用以优化资源的使用，并且单个 **`EventLoop`** 可能会被指派用于服务多个 **`Channel`** 。

- **`EventLoop`**  和 **`Channel`**  **存在1对多的关系**

需要注意的是， **`Netty`** 的 **`EventLoop`** 在继承了 **`ScheduledExecutorService`** 的同时，只定 义了一个方法，**`parent()`** 。这个方法，如下面的代码片断所示，用于返回到当前 **`EventLoop`** 实 现的实例所属的 **`EventLoopGroup`** 的引用 

```
注意：从上面的代码来看继承了OrderedEventExecutor顺序时间执行器。那么为什么要顺序执行事件或任务呢？
事件/任务的执行顺序 事件和任务是以先进先出(FIFO)的顺序执行的。这样可以通过保证字 节内容总是按正确的顺序被理，
消除潜在的数据损坏的可能性。
```

