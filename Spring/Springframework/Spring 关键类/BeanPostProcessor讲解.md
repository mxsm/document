### 1. BeanPostProcessor是干什么的？

BeanPostProcessor接口作用是：如果我们需要在Spring容器完成Bean的实例化、配置和其他的初始化前后添加一些自己的逻辑处理，我们就可以定义一个或者多个BeanPostProcessor接口的实现，然后注册到容器中。(类似于拦截器和过滤器)

> 通俗的讲就是bean实例化后每个bean就会通过 **BeanPostProcessor** 实现的类的处理。

Spring Bean的实例化图解：

![图](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/bean%E5%AE%9E%E4%BE%8B%E5%8C%96%E8%BF%87%E7%A8%8B.png?raw=true)

在检查完 **Aware** 接口后，就开始调用 **BeanPostProcessor** 进行前置处理后后置处理。下面来看一下Spring中的几类继承：

- AOP相关的

  ![图](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/BeanPostProcessor-aop.png?raw=true)

- bean 和 context相关的

  ![图](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/BeanPostProcessor-core.png?raw=true)

- Spring Boot相关的实现

  ![图](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/BeanPostProcessor-springboot.png?raw=true)

  

BeanPostProcessor是在Bean实例化后，在自定义初始化方法前后执行。



### 2. BeanPostProcessor代码解析

```java
public interface BeanPostProcessor {

	//自定义初始化方法之前执行
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	//自定义初始化方法之后执行
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

代码演示：

```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

        System.out.println( " ----before----- " + beanName);

        return bean;
    }


    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {

        System.out.println( " ----after----- " + beanName);

        return bean;
    }
}
```
```java
public class TestBean {

    private String name;

    public void init(){
        System.out.println("TestBean---init()");
        this.name = "test";
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <bean id="testBean" class="com.github.mxsm.bean.TestBean" init-method="init"/>

    <bean class="com.github.mxsm.processor.MyBeanPostProcessor" id="myBeanPostProcessor"/>

</beans>
```

```java
public class ApplicationBoot{
    public static void main( String[] args ) {

        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("application.xml");
        TestBean testBean = applicationContext.getBean(TestBean.class);
        System.out.println(testBean.getName());

    }
}
```



![图示](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/BeanPostProcessor%E4%BB%A3%E7%A0%81%E6%BC%94%E7%A4%BA.png?raw=true)

通过代码可以看出来执行结果。

### 3. 看一下Spring自身的实现

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor {

	private final ConfigurableApplicationContext applicationContext;

	private final StringValueResolver embeddedValueResolver;


	/**
	 * Create a new ApplicationContextAwareProcessor for the given context.
	 */
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
	}


	@Override
	@Nullable
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		return bean;
	}

}
```

当前类主要用来处理继承了 **`Aware`** 接口类。用来处理设置对于的数据。



### 4. InstantiationAwareBeanPostProcessor接口介绍

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

	//实例化之前--bean对象还没生成
	@Nullable
	default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}


    //实例化之后--bean对象已经生成
	default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		return true;
	}


	@Nullable
	default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
			throws BeansException {

		return null;
	}
}
```

> 代码示例地址：https://github.com/mxsm/spring-sample/tree/master/spring-beanPostProcessor

https://cloud.tencent.com/developer/article/1409273