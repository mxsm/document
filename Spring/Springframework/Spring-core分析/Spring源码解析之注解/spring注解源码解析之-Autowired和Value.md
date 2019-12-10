### 1. @Autowired和@Value注解

首先来看一下这两个注解的源码

```java
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Autowired {
	boolean required() default true;
}

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Value {
	String value();
}

```

从上面的源码可以看出来两个注解都可以用于 **`方法、参数、变量、注解类型`** 。**`@Autowired`** 还能用于构造函数。通过这个类使用的地方可以看出来，主要用于类的内部。所以主要是通过 **`BeanPostProcessor`** 来处理这两个注解。

### 2. 注解处理类AutowiredAnnotationBeanPostProcessor源码解析

首先看 ***`AutowiredAnnotationBeanPostProcessor`*** 是如何注入到上下文中的。通过代码发现主要是有 **`AnnotationConfigUtils#registerAnnotationConfigProcessors`** 注入到上下文中的。看一下源码

```java
	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
		//省略代码

        //注入AutowiredAnnotationBeanPostProcessor类的定义
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
        }
		//省略代码

		return beanDefs;
	}
```

下面来看一下 **`AutowiredAnnotationBeanPostProcessor`** 源码：

```java
public class AutowiredAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
		implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {
    
    //省略代码
}
```

**`AutowiredAnnotationBeanPostProcessor`** 继承了 **`InstantiationAwareBeanPostProcessorAdapter`** 类（这个类是BeanPostProcessor的实现）。然后还实现了其他的三个接口。

首先来看一下这个类的构造函数：

```java
	public AutowiredAnnotationBeanPostProcessor() {
		this.autowiredAnnotationTypes.add(Autowired.class);
		this.autowiredAnnotationTypes.add(Value.class);
		try {
			this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
			logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```

从这个构造函数可以看出来主要处理  **@Autowired、@Value、javax.inject.Inject** 。 第三个是javax。这就是对 JSR-330 标准的支持。