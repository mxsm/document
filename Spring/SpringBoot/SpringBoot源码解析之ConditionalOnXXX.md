### 1. SpringBoot中的ConditionalOnXXX类型的注解

```mermaid
graph LR
B(ConditionalOnXXX)
    B --> ConditionalOnBean(ConditionalOnBean)
    B -->ConditionalOnClass(ConditionalOnClass)
    B -->ConditionalOnCloudPlatform(ConditionalOnCloudPlatform)
    B -->ConditionalOnExpression(ConditionalOnExpression)
    B -->ConditionalOnJava(ConditionalOnJava)
    B -->ConditionalOnJndi(ConditionalOnJndi)
    B -->ConditionalOnMissingBean(ConditionalOnMissingBean)
    B -->ConditionalOnMissingClass(ConditionalOnMissingClass)
    B -->ConditionalOnNotWebApplication(ConditionalOnNotWebApplication)
    B -->ConditionalOnProperty(ConditionalOnProperty)
    B -->ConditionalOnResource(ConditionalOnResource)
    B -->ConditionalOnSingleCandidate(ConditionalOnSingleCandidate)
    B -->ConditionalOnWebApplication(ConditionalOnWebApplication)
```

![](https://github.com/mxsm/document/blob/master/image/Spring/SpringBoot/ConditionalOnXXX.png?raw=true)

通过上面的可以看出来主要是有这些条件化加载类的注解。然后加上三个顺序的注解

```mermaid
graph LR
B(顺序注解)
    B --> ConditionalOnBean(AutoConfigureBefore)
    B -->ConditionalOnClass(AutoConfigureOrder)
    B -->ConditionalOnCloudPlatform(AutoConfigureAfter)
```

![](https://github.com/mxsm/document/blob/master/image/Spring/SpringBoot/SpringBoot%E9%A1%BA%E5%BA%8F%E6%B3%A8%E8%A7%A3.png?raw=true)

通过这些注解来实现SpringBoot的自动注入和取消xml的配置。下面会选取 ***`ConditionalOnClass`*** 来进行源码分析。这些注解如何进行协同工作来实现条件化注入的。

### 2. SpringBoot源码分析之ConditionalOnClass

