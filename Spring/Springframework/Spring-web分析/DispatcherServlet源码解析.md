### 1. DispatcherServlet是什么？
从本质上是说DispatcherServlet就是一个Servlet规范的实现。也就是一个Servlet！
### 2. DispatcherServlet在Spring中的继承关系

```java
public class DispatcherServlet extends FrameworkServlet {
    //省略代码
}

public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    //省略代码
}

public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {
    //省略代码    
}

```
从上面的关系可以看出来 *HttpServletBean* 继承了Servlet API的 *HttpServlet* ，然后 *FrameworkServlet* 继承了
*HttpServletBean*， 最后 *DispatcherServlet* 继承了 *FrameworkServlet* 。
### 3. 源码分析
由于DispatcherServlet是一个Servlet那么就从Servlet的方法入手。Servlet完成实例化以后首先调用的就是init方法。在GenericServlet类中init()方法实现为空，在HttpServletBean重写

```java
@Override
	public final void init() throws ServletException {

		// Set bean properties from init parameters.
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
				initBeanWrapper(bw);
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
				if (logger.isErrorEnabled()) {
					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
				}
				throw ex;
			}
		}

		// 该方法在HttpServletBean类中实现为空--主要由子类进行实现
		initServletBean();
	}
```
下面看一下initServletBean方法在FrameworkServlet中的实现：

```java
@Override
	protected final void initServletBean() throws ServletException {
		
		//省略日志打印

		try {
		    //初始化webApplicationContext
			this.webApplicationContext = initWebApplicationContext();
			//初始化initFrameworkServlet-- 默认实现为空
			initFrameworkServlet();
		}
		catch (ServletException | RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			throw ex;
		}

		//省略日志打印
	}
```
从上面可以看出来主要是初始化webApplicationContext，通过调用initWebApplicationContext()方法。

```java
protected WebApplicationContext initWebApplicationContext() {
        //获取ServletContext中的属性值为WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE对象的值
		//如果有ContextLoaderListner的情况设置的
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;
		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
				
					if (cwac.getParent() == null) {
						cwac.setParent(rootContext);
					}
					//刷新WebApplicationContext
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
		    //根据contextAttribute属性查找WebApplicationContext
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// 如果上面都没有创建一个
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
		
			synchronized (this.onRefreshMonitor) {
			   //触发onRefresh方法去做一些其他的事情
				onRefresh(wac);
			}
		}

		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
		}

		return wac;
	}
```
分析一下createWebApplicationContext(rootContext)方法，configureAndRefreshWebApplicationContext(cwac);在分析一下createWebApplicationContext中也有调用：

```java
protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
		//获取contextClass,如果没有设置contextClass默认是XmlWebApplicationContext
		Class<?> contextClass = getContextClass();
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			//不是继承关系抛错
		}
		
		//实例化数据
		ConfigurableWebApplicationContext wac =
				(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

		wac.setEnvironment(getEnvironment());
		wac.setParent(parent);
		//contextConfigLocation设置配置文件或者扫描包的路径,根据不同的Context来匹配
		String configLocation = getContextConfigLocation();
		if (configLocation != null) {
			wac.setConfigLocation(configLocation);
		}
		
		//配置刷新--调用了Spring core 的refresh方法
		configureAndRefreshWebApplicationContext(wac);

		return wac;
	}
```
创建完成Spring application context后开始FrameworkServlet#refresh方法这个方法是一个抽象的方法在子类中实现，也就是在DispatcherServlet中实现：

```java
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}
	//初始化一些请求过程中的策略
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```
对于方法initStrategies主要做了这9件事情。

- initMultipartResolver

  初始化MultipartResolver，用于处理文件上传服务，如果有文件上传，那么就会将当前的HttpServletRequest包装成DefaultMultipartHttpServletRequest，并且将每个上传的内容封装成CommonsMultipartFile对象。需要在dispatcherServlet-servlet.xml中配置文件上传解析器。

- initLocaleResolver

  用于处理应用的国际化问题，本地化解析策略。

- initThemeResolver

  用于定义一个主题

- initHandlerMapping

  用于定义请求映射关系

- initHandlerAdapters

  用于根据Handler的类型定义不同的处理规则

- initHandlerExceptionResolvers

  当Handler处理出错后，会通过此将错误日志记录在log文件中，默认实现类是SimpleMappingExceptionResolver

- initRequestToViewNameTranslators

  将指定的ViewName按照定义的RequestToViewNameTranslators替换成想要的格式

- initViewResolvers

  用于将View解析成页面

- initFlashMapManager

  用于生成FlashMap管理器