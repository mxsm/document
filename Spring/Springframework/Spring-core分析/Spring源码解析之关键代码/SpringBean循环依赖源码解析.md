### 1. 什么是Spring Bean的循环依赖

两个或者两个以上的bean互相持有对方、最终形成 **闭环** 。如下图：

![](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/SpringBean%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8.png?raw=true)

A、B、C三个Bean形成了循环依赖。

Spring中循环依赖的场景有：

1. 构造函数循环依赖
2. Field属性循环依赖

### 2. 怎么检查循环依赖

检测循环依赖相对比较容易，Bean在创建的时候可以给该Bean打标，如果递归调用回来发现正在创建中的话，即说明了循环依赖了。

### 3. Spring怎么解决循环依赖

在分析Spring源码中如何解决循环依赖的过程中前，我们先了解一下Spring的Bean的创建过程。简化后主要分为三个主要的步骤：

![](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/bean%E5%88%9B%E5%BB%BA%E7%9A%84%E4%B8%89%E4%B8%AA%E4%B8%BB%E8%A6%81%E6%AD%A5%E9%AA%A4.png?raw=true)

1. **实例化**
2. **实例化好的Bean填充属性**
3. **初始化**

> Bean的实例化和初始化的区别？
>
> Java有以下几种方式创建类对象：
>
> - 利用new关键字
> - 利用反射Class.newInstance
> - 利用Constructor.newIntance(相比Class.newInstance多了有参和私有构造函数)
> - 利用Cloneable/Object.clone()
> - 利用反序列化
>
> 这些都是讲的实例化。而初始化是Class实例化后的具体对象才来进行初始化。
>
> **`简单的说：实例化针对的是Class,而初始化针对的是实例对象。`**

#### 3.1 Spring循环依赖解决的源码解析--5.2.X的源码版本

Spring主要是通过缓存不同状态下的Bean来解决循环依赖的问题，下面看一下 **`DefaultSingletonBeanRegistry`** 类中的三个缓存变量：

```java
	/** Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

三个缓存的作用简单说明：

| 缓存                  | 作用                                                         |
| --------------------- | ------------------------------------------------------------ |
| singletonObjects      | 用于存放完全初始化好的 bean，从该缓存中取出的 bean 可以直接使用 |
| singletonFactories    | 存放原始的 bean 对象（尚未填充属性），用于解决循环依赖       |
| earlySingletonObjects | 提前暴光的单例对象的Cache，用于解决循环依赖                  |

下面来看一下创建Bean实例的主要一个方法 **AbstractBeanFactory#doGetBean**：

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// 检查单例缓存是否有手动注册的单例
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			//删除日志打印代码
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			//判断是否在创建中
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 检查parentBeanFactory是否存在调用parentBeanFactory的getBean方法
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 确保当前bean依赖的bean的初始化
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							//抛错BeanCreationException
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							//抛错BeanCreationException
						}
					}
				}

				// 创建Bean--单例模式、原型模式、根据Scop来创建三种
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
							//关键方法
							return createBean(beanName, mbd, args);
							//删除了部分无关紧要的代码
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						//抛错BeanCreationException
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// 检查所需的类型是否与实际bean实例的类型匹配
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				//抛错BeanNotOfRequiredTypeException
			}
		}
		return (T) bean;
	}
```

下面我们用流程图来梳理一下这一段代码的逻辑：

![](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/doGetBean%E6%B5%81%E7%A8%8B.png?raw=true)

