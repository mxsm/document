### 1. Spring如何加载自定义的xml Element

下面来通过代码的Debug来看Spring是如何加载自定义的xml Element

> 代码：<https://github.com/mxsm/spring-sample/tree/master/namespace-handler>

### 2. NamespaceHandler的继承关系

首先看一下 **`NamespaceHandler`** 的继承关系

![图](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/NamespaceHandler.png?raw=true)

从上图可以看出来几个比较熟悉的：

- AopNamespaceHandler

  **aop** 的 **Element** 处理

- TxNamespaceHandler

  事务的处理节点

在继承过程中抽象类 **`NamespaceHandlerSupport`** 实现了 **`NamespaceHandler`** 。自定义也是主要通过 **`NamespaceHandlerSupport`**  实现这个抽象类。

### 3. 如何加载使用NamespaceHandler的实现类

在 **Spring** 中定义了一个 **`NamespaceHandlerResolver`**  接口用来解析 **`NamespaceHandler`**  

```java
@FunctionalInterface
public interface NamespaceHandlerResolver {

	/**
	 * Resolve the namespace URI and return the located {@link NamespaceHandler}
	 * implementation.
	 * @param namespaceUri the relevant namespace URI
	 * @return the located {@link NamespaceHandler} (may be {@code null})
	 */
	@Nullable
	NamespaceHandler resolve(String namespaceUri);

}
```

这个接口就一个人方法，方法的参数传入的是命名空间的Uri。在 Spring中实现了这个接口的只有一个类 **`DefaultNamespaceHandlerResolver`** 。下面一下这类的代码实现(主要关注一下resolve方法)：

```java
public class DefaultNamespaceHandlerResolver implements NamespaceHandlerResolver {

	/**
	 * 空间URI和处理器对应关系存放的文件(自定义同样会被加载)
	 */
	public static final String DEFAULT_HANDLER_MAPPINGS_LOCATION = "META-INF/spring.handlers";


	/** Logger available to subclasses. */
	protected final Log logger = LogFactory.getLog(getClass());

	/** ClassLoader to use for NamespaceHandler classes. */
	@Nullable
	private final ClassLoader classLoader;

	/** Resource location to search for. */
	private final String handlerMappingsLocation;

	/** Stores the mappings from namespace URI to NamespaceHandler class name / instance. */
	@Nullable
	private volatile Map<String, Object> handlerMappings;


	/**
	 * Create a new {@code DefaultNamespaceHandlerResolver} using the
	 * default mapping file location.
	 * <p>This constructor will result in the thread context ClassLoader being used
	 * to load resources.
	 * @see #DEFAULT_HANDLER_MAPPINGS_LOCATION
	 */
	public DefaultNamespaceHandlerResolver() {
		this(null, DEFAULT_HANDLER_MAPPINGS_LOCATION);
	}

	/**
	 * Create a new {@code DefaultNamespaceHandlerResolver} using the
	 * default mapping file location.
	 * @param classLoader the {@link ClassLoader} instance used to load mapping resources
	 * (may be {@code null}, in which case the thread context ClassLoader will be used)
	 * @see #DEFAULT_HANDLER_MAPPINGS_LOCATION
	 */
	public DefaultNamespaceHandlerResolver(@Nullable ClassLoader classLoader) {
		this(classLoader, DEFAULT_HANDLER_MAPPINGS_LOCATION);
	}


	public DefaultNamespaceHandlerResolver(@Nullable ClassLoader classLoader, String handlerMappingsLocation) {
		Assert.notNull(handlerMappingsLocation, "Handler mappings location must not be null");
		this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
		this.handlerMappingsLocation = handlerMappingsLocation;
	}


	/**
	 * 解析传入的命名空间的URI
	 */
	@Override
	@Nullable
	public NamespaceHandler resolve(String namespaceUri) {
		Map<String, Object> handlerMappings = getHandlerMappings();
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			String className = (String) handlerOrClassName;
			try {
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
                //调用init方法--所以在实现 NamespaceHandlerSupport只需要
                //实现init方法原因就在这里
				namespaceHandler.init();
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
						"] for namespace [" + namespaceUri + "]", ex);
			}
			catch (LinkageError err) {
				throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
						className + "] for namespace [" + namespaceUri + "]", err);
			}
		}
	}

	/**
	 * 获取META-INF/spring.handlers里面的对应的命名空间URI和空间处理器的
	 * 对应的关系
	 */
	private Map<String, Object> getHandlerMappings() {
		Map<String, Object> handlerMappings = this.handlerMappings;
		if (handlerMappings == null) {
			synchronized (this) {
				handlerMappings = this.handlerMappings;
				if (handlerMappings == null) {
					if (logger.isTraceEnabled()) {
						logger.trace("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
					}
					try {
						Properties mappings =
								PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
						if (logger.isTraceEnabled()) {
							logger.trace("Loaded NamespaceHandler mappings: " + mappings);
						}
						handlerMappings = new ConcurrentHashMap<>(mappings.size());
						CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
						this.handlerMappings = handlerMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
					}
				}
			}
		}
		return handlerMappings;
	}


	@Override
	public String toString() {
		return "NamespaceHandlerResolver using mappings " + getHandlerMappings();
	}

}
```

通过断点的方式来看一下整个调用方法的调用如下图：

![图](https://github.com/mxsm/document/blob/master/image/Spring/Springframework/Spring%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8B%93%E5%B1%95xml%E8%B0%83%E7%94%A8%E9%93%BE.png?raw=true)

 下面来分一下整个调用链的过程，上面断点打在  **`DefaultNamespaceHandlerResolver`** 的 **`resolve`**  方法的下面这段代码处

```java
handlerMappings.put(namespaceUri, namespaceHandler);
```

> 细心的话可能你会发现debug的过程中，如果你xml中包含 bean这个节点你会发现并不会走到你的断点这个地方来这个是为什么呢？(答案会在下面的分析过程中给出来)
>
> Debug的代码在上面已经给出来了。

............有点晚了明天接着分析  睡觉了