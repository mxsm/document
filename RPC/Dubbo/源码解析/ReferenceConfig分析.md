### ReferenceConfig代码分析

首先看一下 **`ReferenceConfig`** 继承关系：

![图](https://github.com/mxsm/document/blob/master/image/RPC/Dubbo/ReferenceConfig.png?raw=true)

看一下代码调用：

```java
import org.apache.dubbo.rpc.config.ApplicationConfig;
import org.apache.dubbo.rpc.config.RegistryConfig;
import org.apache.dubbo.rpc.config.ConsumerConfig;
import org.apache.dubbo.rpc.config.ReferenceConfig;
import com.xxx.XxxService;
 
// 当前应用配置
ApplicationConfig application = new ApplicationConfig();
application.setName("yyy");
 
// 连接注册中心配置
RegistryConfig registry = new RegistryConfig();
registry.setAddress("10.20.130.230:9090");
registry.setUsername("aaa");
registry.setPassword("bbb");
 
// 注意：ReferenceConfig为重对象，内部封装了与注册中心的连接，以及与服务提供方的连接
 
// 引用远程服务
ReferenceConfig<XxxService> reference = new ReferenceConfig<XxxService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
reference.setApplication(application);
reference.setRegistry(registry); // 多个注册中心可以用setRegistries()
reference.setInterface(XxxService.class);
reference.setVersion("1.0.0");
 
// 和本地bean一样使用xxxService
XxxService xxxService = reference.get(); // 注意：此代理对象内部封装了所有通讯细节，对象较重，请缓存复用
```

直接从 **`reference.get()`** 开始分析

```java
    public synchronized T get() {
        //检查配置
        checkAndUpdateSubConfigs();

        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        if (ref == null) {
            //不存在进行初始化
            init();
        }
        return ref;
    }
```

如要存在两个方法：

- **`checkAndUpdateSubConfigs()`**

  检查配置

- **`init()`**

  ```java
      private void init() {
          //判断是否已经初始化
          if (initialized) {
              return;
          }
          initialized = true;
          //本地检查
          checkStubAndLocal(interfaceClass);
          //检查Mock
          checkMock(interfaceClass);
          Map<String, String> map = new HashMap<String, String>();
  
          map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
          //追加运行时参数
          appendRuntimeParameters(map);
          //检测是否为泛化接口
          if (!isGeneric()) {
              String revision = Version.getVersion(interfaceClass, version);
              if (revision != null && revision.length() > 0) {
                  map.put("revision", revision);
              }
  
              String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
              if (methods.length == 0) {
                  logger.warn("No method found in service interface " + interfaceClass.getName());
                  map.put("methods", Constants.ANY_VALUE);
              } else {
                  map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
              }
          }
          map.put(Constants.INTERFACE_KEY, interfaceName);
          appendParameters(map, application);
          appendParameters(map, module);
          appendParameters(map, consumer, Constants.DEFAULT_KEY);
          appendParameters(map, this);
          Map<String, Object> attributes = null;
          if (CollectionUtils.isNotEmpty(methods)) {
              attributes = new HashMap<String, Object>();
              for (MethodConfig methodConfig : methods) {
                  appendParameters(map, methodConfig, methodConfig.getName());
                  String retryKey = methodConfig.getName() + ".retry";
                  if (map.containsKey(retryKey)) {
                      String retryValue = map.remove(retryKey);
                      if ("false".equals(retryValue)) {
                          map.put(methodConfig.getName() + ".retries", "0");
                      }
                  }
                  attributes.put(methodConfig.getName(), convertMethodConfig2AyncInfo(methodConfig));
              }
          }
  
          String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
          if (StringUtils.isEmpty(hostToRegistry)) {
              hostToRegistry = NetUtils.getLocalHost();
          }
          map.put(Constants.REGISTER_IP_KEY, hostToRegistry);
          //创建代理类
          ref = createProxy(map);
  
          ApplicationModel.initConsumerModel(getUniqueServiceName(), buildConsumerModel(attributes));
      }
  
  ```

  初始化的步骤：

  1. **判断是否已经初始化**

  2. **各种参数的检查**

  3. **创建动态代理类**

     创建代理类方式有两种：

     - **`JavassistProxyFactory`**

       ```java
       public class JavassistProxyFactory extends AbstractProxyFactory {
       
           @Override
           @SuppressWarnings("unchecked")
           public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
               return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
           }
       
           @Override
           public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
               // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
               final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
               return new AbstractProxyInvoker<T>(proxy, type, url) {
                   @Override
                   protected Object doInvoke(T proxy, String methodName,
                                             Class<?>[] parameterTypes,
                                             Object[] arguments) throws Throwable {
                       return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
                   }
               };
           }
       
       }
       ```

       

     - **`JdkProxyFactory`**

       ```java
       public class JdkProxyFactory extends AbstractProxyFactory {
       
           //JDK获取代理类
           @Override
           public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
               return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
           }
       
           @Override
           public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
               return new AbstractProxyInvoker<T>(proxy, type, url) {
                   @Override
                   protected Object doInvoke(T proxy, String methodName,
                                             Class<?>[] parameterTypes,
                                             Object[] arguments) throws Throwable {
                       Method method = proxy.getClass().getMethod(methodName, parameterTypes);
                       return method.invoke(proxy, arguments);
                   }
               };
           }
       
       }
       ```

       

