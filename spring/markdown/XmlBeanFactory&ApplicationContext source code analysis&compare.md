# XmlBeanFactory&ApplicationContext source code analysis&compare

## XmlBeanFactory(屌丝版inversion of control)

### entrance

```java
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);
}
```

### XmlBeanDefinitionReader

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			// 将资源文件转为InputStream的IO流
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				// 从InputStream中得到XML的解析源
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				// 这里是具体的读取过程
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			// 通过DOM4J加载解析XML文件，最终形成Document对象
			Document doc = doLoadDocument(inputSource, resource);
			// 通过对Document对象的操作，完成BeanDefinition的加载和注册工作
            return registerBeanDefinitions(doc, resource);
		}
		catch (Exception ex) {
			throw ex;
		}
	}
```

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		// 创建BeanDefinitionDocumentReader来解析Document对象，完成BeanDefinition解析
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 获得容器中已经注册的BeanDefinition数量
		int countBefore = getRegistry().getBeanDefinitionCount();
		//解析过程入口，BeanDefinitionDocumentReader只是个接口，具体的实现过程在DefaultBeanDefinitionDocumentReader完成
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 统计新的的BeanDefinition数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

### DefaultBeanDefinitionDocumentReader

```java
protected void doRegisterBeanDefinitions(Element root) {
		// 这里使用了委托模式，将具体的BeanDefinition解析工作交给了BeanDefinitionParserDelegate去完成
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		// 判断该根标签是否包含http://www.springframework.org/schema/beans默认命名空间
		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		// 在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性
		preProcessXml(root);
		// 委托给BeanDefinitionParserDelegate,从Document的根元素开始进行BeanDefinition的解析
		parseBeanDefinitions(root, this.delegate);
		// 在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
		postProcessXml(root);

		this.delegate = parent;
	}
```

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		// 加载的Document对象是否使用了Spring默认的XML命名空间（beans命名空间）
		if (delegate.isDefaultNamespace(root)) {
			// 获取Document对象根元素的所有子节点（bean标签、import标签、alias标签和其他自定义标签context、aop等）
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					// bean标签、import标签、alias标签，则使用默认解析规则
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {//像context标签、aop标签、tx标签，则使用用户自定义的解析规则解析元素节点
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			// 如果不是默认的命名空间，则使用用户自定义的解析规则解析元素节点
			delegate.parseCustomElement(root);
		}
	}
```



## ApplicationContext(高富帅版inversion of control)

### entrance

AbstractApplicationContext

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
            // STEP 1： 刷新预处理
			prepareRefresh();
			// Tell the subclass to refresh the internal bean factory.
            // STEP 2：
            // 		a） 创建IoC容器（DefaultListableBeanFactory）
            //		b） 加载解析XML文件（最终存储到Document对象中）
            //		c） 读取Document对象，并完成BeanDefinition的加载和注册工作
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// Prepare the bean factory for use in this context.
            // STEP 3： 对IoC容器进行一些预处理（设置一些公共属性）
			prepareBeanFactory(beanFactory);
			try {
				// Allows post-processing of the bean factory in context subclasses.
                // STEP 4： 
				postProcessBeanFactory(beanFactory);
				// Invoke factory processors registered as beans in the context.
				// STEP 5： 调用BeanFactoryPostProcessor后置处理器对BeanDefinition处理
                invokeBeanFactoryPostProcessors(beanFactory);
				// Register bean processors that intercept bean creation.
				// STEP 6： 注册BeanPostProcessor后置处理器
                registerBeanPostProcessors(beanFactory);
				// Initialize message source for this context.
				// STEP 7： 初始化一些消息源（比如处理国际化的i18n等消息源）
                initMessageSource();
				// Initialize event multicaster for this context.
				// STEP 8： 初始化应用事件广播器
                initApplicationEventMulticaster();
				// Initialize other special beans in specific context subclasses.
				// STEP 9： 初始化一些特殊的bean
                onRefresh();
				// Check for listener beans and register them.
				// STEP 10： 注册一些监听器
                registerListeners();
				// Instantiate all remaining (non-lazy-init) singletons.
				// STEP 11： 实例化剩余的单例bean（非懒加载方式）
                // 注意事项：Bean的IoC、DI和AOP都是发生在此步骤
                finishBeanFactoryInitialization(beanFactory);
				// Last step: publish corresponding event.
				// STEP 12： 完成刷新时，需要发布对应的事件
                finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
```

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		// 主要是通过该方法完成IoC容器的刷新
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```

### AbstractRefreshableApplicationContext

```java
protected final void refreshBeanFactory() throws BeansException {
		// 如果之前有IoC容器，则销毁
    	if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
       		// 创建IoC容器，也就是DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			// 设置工厂的属性：是否允许BeanDefinition覆盖和是否允许循环依赖
			customizeBeanFactory(beanFactory);
            // 调用载入BeanDefinition的方法，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
			loadBeanDefinitions(beanFactory);//钩子方法
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

### AbstractXmlApplicationContext

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		// 创建一个BeanDefinition阅读器，通过阅读XML文件，真正完成BeanDefinition的加载和注册
    //这里要注意入参，体现了接口分离原则的好处
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
	
		// 委托给BeanDefinition阅读器去加载BeanDefinition
		loadBeanDefinitions(beanDefinitionReader);
	}
```

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		// 获取资源的定位
		// 这里getConfigResources是一个空实现，真正实现是调用子类的获取资源定位的方法
		// 比如：ClassPathXmlApplicationContext中进行了实现
		// 		而FileSystemXmlApplicationContext没有使用该方法
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			// XML Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的资源
			reader.loadBeanDefinitions(configResources);
		}
		// 如果子类中获取的资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			// XML Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的资源
			reader.loadBeanDefinitions(configLocations);
		}
	}
```

### AbstractBeanDefinitionReader

```java
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int counter = 0;
		for (String location : locations) {
			counter += loadBeanDefinitions(location);
		}
		return counter;
	}
```

```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources)
			throws BeanDefinitionStoreException {
		// 获取在IoC容器初始化过程中设置的资源加载器
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				// 委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			} catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		} else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
```

### XmlBeanDefinitionReader

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		// 将读入的XML资源进行特殊编码处理
		return loadBeanDefinitions(new EncodedResource(resource));
	}
```

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			// 将资源文件转为InputStream的IO流
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				// 从InputStream中得到XML的解析源
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				// 这里是具体的读取过程
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			// 通过DOM4J加载解析XML文件，最终形成Document对象
			Document doc = doLoadDocument(inputSource, resource);
			// 通过对Document对象的操作，完成BeanDefinition的加载和注册工作
            return registerBeanDefinitions(doc, resource);
		}
		catch (Exception ex) {
			throw ex;
		}
	}
```

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		// 创建BeanDefinitionDocumentReader来解析Document对象，完成BeanDefinition解析
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 获得容器中已经注册的BeanDefinition数量
		int countBefore = getRegistry().getBeanDefinitionCount();
		//解析过程入口，BeanDefinitionDocumentReader只是个接口，具体的实现过程在DefaultBeanDefinitionDocumentReader完成
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 统计新的的BeanDefinition数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

