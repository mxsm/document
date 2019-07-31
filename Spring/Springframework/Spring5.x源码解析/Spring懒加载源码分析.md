### 1 什么是懒加载
Spring中有一个概念叫做懒加载，那么什么是懒加载？**就是在需要用到的时候加载类**。
### 2 Spring是如何处理懒加载。
对于Spring中的一个bean如何判断是否为懒加载Bean。分为两种情况：  
- **Bean的定义由XML完成**

  ```xml
  <bean id="addressBean" class="com.fh.spring.Address" lazy-init="true" />
  ```

- **Bean的定义由注解完成**

  @Lazy注解，对于类和类的变量值使用了@Lazy注解说明这个是懒加载

### 3 分析@Lazy的实现源码
@Lazy注解主要使用在两个地方：
1. 类上面--配合@Component注解使用
2. 类的私有变量上面--配合@Autowired注解使用

#### 3.1 分析@Lazy在类上

```java

@Component
@Lazy
public class ServiceTest {
 //.......省略其他的代码   
}
```
如上示例代码，@Lazy的位置在ServiceTest上面。在Spring扫描对应的包下面的类，解析未BeanDefinition的时候，对应的BeanDefinition#isLazyInit方法返回为true，说明这个BeanDefinition实现的是懒加载。  
那么我看一下在DefaultListableBeanFactory#preInstantiateSingletons方法

```java
public void preInstantiateSingletons() throws BeansException {
    
    //省略了部分代码
    for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			//bd.isLazyInit() 如果是true就执行下面的代码
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			    //  省略Spring源码中的代码
			}
		}
    
}
```