### 1. doGetBean方法源码解析

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		/**
		 * 通过name获取BeanName,这里不能使用name作为beanName：
		 * 1. name可能是别名，通过方法转换为具体的实例名称
		 * 2. name可能会以&开头，表明调用者想获取FactoryBean本身，而非FactoryBean创建bean
		 *    FactoryBean 的实现类和其他的 bean 存储方式是一致的，即 <beanName, bean>，
		 *    beanName 中是没有 & 这个字符的。所以我们需要将 name 的首字符 & 移除，这样才能从
		 *    缓存里取到 FactoryBean 实例。
		 *
		 */
		final String beanName = transformedBeanName(name);
		Object bean;

		// 从缓存中获取bean
		Object sharedInstance = getSingleton(beanName);

		/*
		 * 如果 sharedInstance = null，则说明缓存里没有对应的实例，表明这个实例还没创建。
		 *( BeanFactory 并不会在一开始就将所有的单例 bean 实例化好，而是在调用 getBean 获取bean 时再实例化，也就是懒加载)。
		 * getBean 方法有很多重载，比如 getBean(String name, Object... args)，我们在首次获取
		 * 某个 bean 时，可以传入用于初始化 bean 的参数数组（args），BeanFactory 会根据这些参数
		 * 去匹配合适的构造方法构造 bean 实例。当然，如果单例 bean 早已创建好，这里的 args 就没有
		 * 用了，BeanFactory 不会多次实例化单例 bean。
		 */
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}

			/*
			 * 如果 sharedInstance 是普通的单例 bean，下面的方法会直接返回。但如果
			 * sharedInstance 是 FactoryBean 类型的，则需调用 getObject 工厂方法获取真正的
			 * bean 实例。如果用户想获取 FactoryBean 本身，这里也不会做特别的处理，直接返回
			 * 即可。毕竟 FactoryBean 的实现类本身也是一种 bean，只不过具有一点特殊的功能而已。
			 */
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		/*
		 * 如果上面的条件不满足，则表明 sharedInstance 可能为空，此时 beanName 对应的 bean
		 * 实例可能还未创建。这里还存在另一种可能，如果当前容器有父容器，beanName 对应的 bean 实例
		 * 可能是在父容器中被创建了，所以在创建实例前，需要先去父容器里检查一下。
		 */
		else {
			// BeanFactory 不缓存 Prototype 类型的 bean，无法处理该类型 bean 的循环依赖问题
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 如果 sharedInstance = null，则到父容器中查找 bean 实例
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
				// 合并父 BeanDefinition 与子 BeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 检查是否有 dependsOn 依赖，如果有则先初始化所依赖的 bean
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {

						/*
						 * 检测是否存在 depends-on 循环依赖，若存在则抛异常。比如 A 依赖 B，
						 * B 又依赖 A，他们的配置如下：
						 *   <bean id="beanA" class="BeanA" depends-on="beanB">
						 *   <bean id="beanB" class="BeanB" depends-on="beanA">
						 *
						 * beanA 要求 beanB 在其之前被创建，但 beanB 又要求 beanA 先于它
						 * 创建。这个时候形成了循环，对于 depends-on 循环，Spring 会直接
						 * 抛出异常
						 */

						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 注册依赖记录
						registerDependentBean(dep, beanName);
						try {
							// 加载 depends-on 依赖
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 创建 bean 实例
				if (mbd.isSingleton()) {

					/*
					 * 这里并没有直接调用 createBean 方法创建 bean 实例，而是通过
					 * getSingleton(String, ObjectFactory) 方法获取 bean 实例。
					 * getSingleton(String, ObjectFactory) 方法会在内部调用
					 * ObjectFactory 的 getObject() 方法创建 bean，并会在创建完成后，
					 * 将 bean 放入缓存中。
					 */

					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					// 如果 bean 是 FactoryBean 类型，则调用工厂方法获取真正的 bean 实例。否则直接返回 bean 实例
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				// 创建 prototype 类型的 bean 实例
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
				// 创建其他类型的 bean 实例
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
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

从源码分析一下 **`doGetBean`** 的执行流程如下：

1. 转换beanName
2. 从缓存中获取实例
3. 如果实例不为空，且 args = null。调用 getObjectForBeanInstance 方法，并按 name 规则返回相应的 bean 实例
4. 若上面的条件不成立，则到父容器中查找 beanName 对有的 bean 实例，存在则直接返回
5. 若父容器中不存在，则进行下一步操作 -- 合并 BeanDefinition
6. 处理 depends-on 依赖
7. 创建并缓存 bean
8. 调用 getObjectForBeanInstance 方法，并按 name 规则返回相应的 bean 实例
9. 按需转换 bean 类型，并返回转换后的 bean 实例

### 2. 方法的源码解析

接下来逐步分析每一个方法

#### 2.1 transformedBeanName

**`transformedBeanName(name)`**  **`beanName`** 的转换，之前分析过由于 **`name`** 可能是 **`FactoryBean`** 或者普通的 Bean的别名所以需要转换。

```java
protected String transformedBeanName(String name) {
	//BeanFactoryUtils.transformedBeanName(name)处理 FactoryBean 类型
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}

//FactoryBean转换
public static String transformedBeanName(String name) {
    	//判空
		Assert.notNull(name, "'name' must not be null");
    	//判断是否已 & 开头是否为FactoryBean
		if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
			return name;
		}
		return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
			do {
				beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
			}
			while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
			return beanName;
		});
}

//将别名处理成BeanName
public String canonicalName(String name) {
		String canonicalName = name;
		// Handle aliasing...
		String resolvedName;
		do {
			resolvedName = this.aliasMap.get(canonicalName);
			if (resolvedName != null) {
				canonicalName = resolvedName;
			}
		}
		while (resolvedName != null);
		return canonicalName;
	}

```

从上面可以看出来 **transformedBeanName** 方法主要是用来处理beanName。

#### 2.2 getSingleton(beanName)

**`getSingleton(beanName)`** 主要从缓存中获取bean。

```java
public Object getSingleton(String beanName) {
		return getSingleton(beanName, true);
}

	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //从缓存Map中获取
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
                //从earlySingletonObjects获取bean
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

