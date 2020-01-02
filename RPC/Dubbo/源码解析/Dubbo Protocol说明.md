### 1.Protocol接口
代码中的注释解释该类是API和SPI的，并且是一个单例模式，线程安全的接口。

```java
@SPI("dubbo")
public interface Protocol {

    //获取协议的默认端口
    int getDefaultPort();

    //服务输出
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    //远程服务引用
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
   
    //协议销毁
    void destroy();

    //获取
    default List<ProtocolServer> getServers() {
        return Collections.emptyList();
    }

}
```
在Dubbo的Protocol接口实现类中有三个比较特殊的三个：
- **ProtocolFilterWrapper**
 
  [对于Filter可以看一下官方的说明](https://dubbo.apache.org/zh-cn/blog/first-dubbo-filter.html)
- **ProtocolListenerWrapper**

  官方对此并没有介绍
- **QosProtocolWrapper**
 
  [官方对于Qos的介绍](http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-qos.html)
 
首先这三个类在Dubbo中有一个统称叫做包装类。对于包装类在进行拓展加载的时候会进行特殊处理。[官方对Wrapper类的介绍](http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi-2.html)


