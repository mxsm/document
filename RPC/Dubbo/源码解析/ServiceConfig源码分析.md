服务导出的代码入口：

```java
public synchronized void export() {
        checkAndUpdateSubConfigs();

        if (!shouldExport()) {
            return;
        }

        if (shouldDelay()) {
            delayExportExecutor.schedule(this::doExport, delay, TimeUnit.MILLISECONDS);
        } else {
            doExport();
        }
    }
```

1. 首先检查和更新配置
2. 判断是否需要导出
3. 发布服务是否要进行延迟发布。延迟的单位毫秒。
4. 不需要延迟直接发布服务

从上面的服务可以看出不管是延迟发布服务和直接发布服务都是调用了 **`doExport`** 方法进行服务的发布。下面就来看一下在方法 **`doExport`** 中做了什么事情。

```java
 protected synchronized void doExport() {
        if (unexported) {
          //抛异常
        }
   			//判断是否已经发布了
        if (exported) {
            return;
        }
        exported = true;
				//判断path是否为空如果为空就赋值接口的名称
        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
   			//真正的做发布的服务的方法
        doExportUrls();
    }
```

从上面可以看出来 **`doExport`**  在方法前面部分主要是做一些校验性的工作。比如是否已经发布的校验等等。最后调用的是 **`doExportUrls`** 方法来发布。

```java
   private void doExportUrls() {
        //根据服务端接口实现等生成URL
        List<URL> registryURLs = loadRegistries(true);
     	  //根据不同的协议发布服务
        for (ProtocolConfig protocolConfig : protocols) {
            String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);
            //生成提供者模型
            ProviderModel providerModel = new ProviderModel(pathKey, ref, interfaceClass);
            //本地缓存提供者的模型
            ApplicationModel.initProviderModel(pathKey, providerModel);
            //根据协议发布服务
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```

