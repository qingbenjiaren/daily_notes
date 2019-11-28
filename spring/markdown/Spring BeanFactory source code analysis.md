# Spring BeanFactory source code analysis

## Spring重要接口详解

### BeanFactory继承体系

#### 体系结构图

![](Spring BeanFactory source code analysis.assets/BeanFactoryPart1.png)

#### BeanFactory

```java
package org.springframework.beans.factory;
    public interface BeanFactory {
    //用来引用一个实例，或把它和工厂产生的Bean区分开
    //就是说，如果一个FactoryBean的名字为a，那么，&a会得到那个Factory
    String FACTORY_BEAN_PREFIX = "&";
    /*
    * 四个不同形式的getBean方法，获取实例
    */
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws
    BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;
    // 是否存在
    boolean containsBean(String name);
    // 是否为单实例
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    // 是否为原型（多实例）
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    // 名称、类型是否匹配
    boolean isTypeMatch(String name, Class<?> targetType)
    throws NoSuchBeanDefinitionException;
    // 获取类型
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    // 根据实例的名字获取实例的别名
    String[] getAliases(String name);
}
```

在BeanFactory里只对IOC容器的基本行为作了定义，根本不关心你的Bean是如何定义怎样加载的。正如我们只关心工厂里得到什么产品对象，至于工厂是怎么生成这些对象的，这个基本的接口不关心。

- **源码说明：**

  4个获取实例的方法。getBean的重载方法

  4个判断的方法。判断是否存在，是否为单例、原型，名称类型是否匹配

  1个获取类型的方法、1个获取别名的方法。根据名称获取类型、根据名称获取别名。一目了然。

- **总结：**

  这10个方法，很明显，这是一个典型的工厂模式的工厂接口



#### ListableBeanFactory

**可将Bean逐一列出的工厂**



```java
public interface ListableBeanFactory extends BeanFactory {
    // 对于给定的名字是否含有BeanDefinition
    boolean containsBeanDefinition(String beanName); 
    // 返回工厂的BeanDefinition总数
    int getBeanDefinitionCount();
    // 返回工厂中所有Bean的名字
    String[] getBeanDefinitionNames();
    // 返回对于指定类型Bean（包括子类）的所有名字
    String[] getBeanNamesForType(Class<?> type);
    /*
    * 返回指定类型的名字
    * includeNonSingletons为false表示只取单例Bean，true则不是
    * allowEagerInit为true表示立刻加载，false表示延迟加载。
    * 注意：FactoryBeans都是立刻加载的。
    */
    String[] getBeanNamesForType(Class<?> type, boolean
    includeNonSingletons,
    boolean allowEagerInit);
    // 根据类型（包括子类）返回指定Bean名和Bean的Map
    <T> Map<String, T> getBeansOfType(Class<T> type) throws BeansException;
    <T> Map<String, T> getBeansOfType(Class<T> type,
    boolean includeNonSingletons, boolean allowEagerInit)
    throws BeansException;
    // 根据注解类型，查找所有有这个注解的Bean名和Bean的Map
    Map<String, Object> getBeansWithAnnotation(
        Class<? extends Annotation> annotationType) throws
    BeansException;
    // 根据指定Bean名和注解类型查找指定的Bean
    <A extends Annotation> A findAnnotationOnBean(String beanName,
    Class<A> annotationType);
}
```

- **源码说明：**

  3个跟BeanDefinition有关的操作。包括BeanDefinition的总数、名字的集合、指定名字的集合

  ```
  这里指出，BeanDefinition是Spring中非常重要的一个类，每个BeanDefinition实例
  都包含一个类在Spring工厂中所有属性。
  ```

  2个getBeanNamesForType重载方法。根据指定类型（包括子类）

  2个getBeansOfType重载方法。根据类型（包括子类）返回指定bean名称和Bean的Map

  2个跟注解查找相关的方法。根据注解类型，查找Bean名和Bean的Map。以及根据指定Bean名和注解类型查找指定的Bean。

- **总结：**

  正如这个工厂接口的名字所示，这个工厂接口最大的特点就是可以列出工厂可以生产的所有实例。当然，工厂并没有直接提供返回所有实例的方法，也没这个必要。它可以返回指定类型的所有实例。而且你可以通过getBeanDefinitionNames()得到工厂所有bean的名字，然后根据这些名字得到所有的Bean。这个工厂接口扩展BeanFactory的功能，作为上文指出的BeanFactory耳机接口，有9个独有的方法，扩展了跟BeanDefinition的功能，提供了BeanDefinition、BeanName、注解有关的各种操作。**它可以根据条件返回Bean的信息集合，这就是它名字的由来--ListableBeanFactory**

#### HierarachicalBeanFactory

**分层的Bean工厂**

```java
public interface HierarchicalBeanFactory extends BeanFactory {
    // 返回本Bean工厂的父工厂
    BeanFactory getParentBeanFactory();
    // 本地工厂是否包含这个Bean
    boolean containsLocalBean(String name);
}
```

- **参数说明：**

  第一个方法返回本Bean工厂的父工厂。这个方法实现了工厂的分层。

  第二个方法判断本地工厂是否包含这个Bean（忽略其他所有父工厂）。这也是分层思想体现

- **总结：**

  这个工厂接口非常简单，实现了Bean工厂的分层。这个工厂接口也是继承自BeanFactory，也是一个二级接口，相对于父接口，它只扩展了一个重要的功能--**工厂分层。**

#### AutowireCapableBeanFactory

**自动装配的Bean工厂**

```java
public interface AutowireCapableBeanFactory extends BeanFactory {
    // 这个常量表明工厂没有自动装配的Bean
    int AUTOWIRE_NO = 0;
    // 表明根据名称自动装配
    int AUTOWIRE_BY_NAME = 1;
    // 表明根据类型自动装配
    int AUTOWIRE_BY_TYPE = 2;
    // 表明根据构造方法快速装配
    int AUTOWIRE_CONSTRUCTOR = 3;
    //表明通过Bean的class的内部来自动装配（有没翻译错...）Spring3.0被弃用。
    @Deprecated
    int AUTOWIRE_AUTODETECT = 4;
    // 根据指定Class创建一个全新的Bean实例
    <T> T createBean(Class<T> beanClass) throws BeansException;
    // 给定对象，根据注释、后处理器等，进行自动装配
    void autowireBean(Object existingBean) throws BeansException;
    // 根据Bean名的BeanDefinition装配这个未加工的Object，执行回调和各种后处理器。
    Object configureBean(Object existingBean, String beanName) throws
    BeansException;
    // 分解Bean在工厂中定义的这个指定的依赖descriptor
    Object resolveDependency(DependencyDescriptor descriptor, String
    beanName) throws BeansException;
    // 根据给定的类型和指定的装配策略，创建一个新的Bean实例
    Object createBean(Class<?> beanClass, int autowireMode, boolean
    dependencyCheck) throws BeansException;
    // 与上面类似，不过稍有不同。
    Object autowire(Class<?> beanClass, int autowireMode, boolean
    dependencyCheck) throws BeansException;
    /*
    * 根据名称或类型自动装配
    */
    void autowireBeanProperties(Object existingBean, int autowireMode,
    boolean dependencyCheck)
    throws BeansException;
    /*
    * 也是自动装配
    */
    void applyBeanPropertyValues(Object existingBean, String beanName)
    throws BeansException;
    /*
    * 初始化一个Bean...
    */
    Object initializeBean(Object existingBean, String beanName) throws
    BeansException;
    /*
    * 初始化之前执行BeanPostProcessors
    */
        Object applyBeanPostProcessorsBeforeInitialization(Object existingBean,
    String beanName)
    throws BeansException;
    /*
    * 初始化之后执行BeanPostProcessors
    */
    Object applyBeanPostProcessorsAfterInitialization(Object existingBean,
    String beanName)
    throws BeansException;
    /*
    * 分解指定的依赖
    */
    Object resolveDependency(DependencyDescriptor descriptor, String
    beanName,
    Set<String> autowiredBeanNames, TypeConverter typeConverter)
    throws BeansException;
}
```

- **源码说明：**

  总共5个静态不可变量来致命装配策略，其中一个常量被Spring3.0废弃、一个常量标识没有自动装配，另外3个常量指明不同的装配策略--根据名称、根据类型、根据构造方法

  8个跟自动装配相关的方法，实在是繁杂，具体的意义我们研究类的时候再分辨

  2个执行BeanPostProcessors的方法

  2个分解指定依赖的方法

- **总结：**

  这个工厂接口继承自BeanFactory，它扩展了自动装配的功能，根据类定义BeanDefinition装配Bean、执行前、后处理等。

#### ConfigurationBeanFactory

**复杂的配置Bean工厂**

```java
public interface ConfigurableBeanFactory extends HierarchicalBeanFactory,
SingletonBeanRegistry {
    String SCOPE_SINGLETON = "singleton"; // 单例
    String SCOPE_PROTOTYPE = "prototype"; // 原型
    /*
    * 搭配HierarchicalBeanFactory接口的getParentBeanFactory方法
    */
    void setParentBeanFactory(BeanFactory parentBeanFactory) throws
    IllegalStateException;
    /*
    * 设置、返回工厂的类加载器
    */
    void setBeanClassLoader(ClassLoader beanClassLoader);
       ClassLoader getBeanClassLoader();
    /*
    * 设置、返回一个临时的类加载器
    */
    void setTempClassLoader(ClassLoader tempClassLoader);
    ClassLoader getTempClassLoader();
    /*
    * 设置、是否缓存元数据，如果false，那么每次请求实例，都会从类加载器重新加载（热加
    载）
    */
    void setCacheBeanMetadata(boolean cacheBeanMetadata);
    boolean isCacheBeanMetadata();//是否缓存元数据
        /*
    * Bean表达式分解器
    */
    void setBeanExpressionResolver(BeanExpressionResolver resolver);
    BeanExpressionResolver getBeanExpressionResolver();
    /*
    * 设置、返回一个转换服务
    */
    void setConversionService(ConversionService conversionService);
    ConversionService getConversionService();
    /*
    * 设置属性编辑登记员...
    */
    void addPropertyEditorRegistrar(PropertyEditorRegistrar registrar);
    /*
    * 注册常用属性编辑器
    */
    void registerCustomEditor(Class<?> requiredType, Class<? extends
    PropertyEditor> propertyEditorClass);
    /*
    * 用工厂中注册的通用的编辑器初始化指定的属性编辑注册器
    */
    void copyRegisteredEditorsTo(PropertyEditorRegistry registry);

        /*
    * 设置、得到一个类型转换器
    */
    void setTypeConverter(TypeConverter typeConverter);
    TypeConverter getTypeConverter();
    /*
    * 增加一个嵌入式的StringValueResolver
    */
        void addEmbeddedValueResolver(StringValueResolver valueResolver);
    String resolveEmbeddedValue(String value);//分解指定的嵌入式的值
    void addBeanPostProcessor(BeanPostProcessor beanPostProcessor);//设置一
    个Bean后处理器
    int getBeanPostProcessorCount();//返回Bean后处理器的数量
    void registerScope(String scopeName, Scope scope);//注册范围
    String[] getRegisteredScopeNames();//返回注册的范围名
    Scope getRegisteredScope(String scopeName);//返回指定的范围
    AccessControlContext getAccessControlContext();//返回本工厂的一个安全访问上
    下文
    void copyConfigurationFrom(ConfigurableBeanFactory otherFactory);//从其
    他的工厂复制相关的所有配置
    /*
    * 给指定的Bean注册别名
    */
    void registerAlias(String beanName, String alias) throws
    BeanDefinitionStoreException;
    void resolveAliases(StringValueResolver valueResolver);//根据指定的
    StringValueResolver移除所有的别名
    /*
    * 返回指定Bean合并后的Bean定义
    */
    BeanDefinition getMergedBeanDefinition(String beanName) throws
    NoSuchBeanDefinitionException;
        boolean isFactoryBean(String name) throws
    NoSuchBeanDefinitionException;//判断指定Bean是否为一个工厂Bean
    void setCurrentlyInCreation(String beanName, boolean inCreation);//设置
    一个Bean是否正在创建
    boolean isCurrentlyInCreation(String beanName);//返回指定Bean是否已经成功
    创建
    void registerDependentBean(String beanName, String
    dependentBeanName);//注册一个依赖于指定bean的Bean
    String[] getDependentBeans(String beanName);//返回依赖于指定Bean的所欲Bean
    名
    String[] getDependenciesForBean(String beanName);//返回指定Bean依赖的所有
    Bean名
    void destroyBean(String beanName, Object beanInstance);//销毁指定的Bean
    void destroyScopedBean(String beanName);//销毁指定的范围Bean
        void destroySingletons(); //销毁所有的单例类
}
```

#### ConfigurableListableBeanFactory

**BeanFacotry的集大成者**

```java
public interface ConfigurableListableBeanFactory
extends ListableBeanFactory, AutowireCapableBeanFactory,
ConfigurableBeanFactory {
    void ignoreDependencyType(Class<?> type);//忽略自动装配的依赖类型
    void ignoreDependencyInterface(Class<?> ifc);//忽略自动装配的接口
    /*
    * 注册一个可分解的依赖
    */
    void registerResolvableDependency(Class<?> dependencyType, Object
    autowiredValue);
    /*
    * 判断指定的Bean是否有资格作为自动装配的候选者
    */
    boolean isAutowireCandidate(String beanName, DependencyDescriptor
    descriptor) throws NoSuchBeanDefinitionException;
    // 返回注册的Bean定义
    BeanDefinition getBeanDefinition(String beanName) throws
    NoSuchBeanDefinitionException;
    // 暂时冻结所有的Bean配置
    void freezeConfiguration();
    // 判断本工厂配置是否被冻结
    boolean isConfigurationFrozen();
    // 使所有的非延迟加载的单例类都实例化。
    void preInstantiateSingletons() throws BeansException;
}
```

- **源码说明：**

  2个忽略自动装配的方法

  1个注册一个可分解的依赖的方法

  1个判断指定Bean是否有资格作为自动装配的候选者的方法

  1个根据指定bean名，返回注册的Bean定义的方法

  2个冻结所有的Bean配置相关的方法

  1个使所有非延迟加载的单例类都实例化的方法

- **总结：**

  工厂接口ConfigurableListableBeanFactory同时继承了3个接口，ListableBeanFactory、AutowireCapableBeanFactory和ConfigurableBeanFactory，扩展之后，加上自有的这8个方法，这个工厂接口总共有83个方法，实在是巨大到不行了。这个工厂接口自有方法总体上只是对父类接口功能的补充，包含BeanFactory体系目前所有方法，可以说是接口的集大成者。

#### BeanDedinitionRegistry

**额外的接口，这个接口基本用来操作定义在工厂内部的BeanDefinition的。**

```java
public interface BeanDefinitionRegistry extends AliasRegistry {
    // 给定bean名称，注册一个新的bean定义
    void registerBeanDefinition(String beanName, BeanDefinition
    beanDefinition) throws BeanDefinitionStoreException;
    /*
    * 根据指定Bean名移除对应的Bean定义
    */
    void removeBeanDefinition(String beanName) throws
    NoSuchBeanDefinitionException;
    /*
    * 根据指定bean名得到对应的Bean定义
    */
    BeanDefinition getBeanDefinition(String beanName) throws
    NoSuchBeanDefinitionException;
    /*
    * 查找，指定的Bean名是否包含Bean定义
    */
    boolean containsBeanDefinition(String beanName);
    String[] getBeanDefinitionNames();//返回本容器内所有注册的Bean定义名称
    int getBeanDefinitionCount();//返回本容器内注册的Bean定义数目
    boolean isBeanNameInUse(String beanName);//指定Bean名是否被注册过。
}
```

### BeanDefinition继承体系

#### 体系结构图

SpringIoc容器管理了我们定义的各种Bean对象及其相互的关系，Bean对象在Spring实现中是以Beandefinition来描述的，其继承体系如下

![](Spring BeanFactory source code analysis.assets/BeanDefinition.png)

### ApplictionContext体系

#### 体系结构图

![](Spring BeanFactory source code analysis.assets/ApplicationContext.png)





## 容器初始化流程源码分析

### 主流程源码分析

#### 找入口

- **java程序入口**

```java
BeanFactory bf = new XMLBeanFactory("spring.xml");
ApplicationContext ctx = new ClassPathXmlApplicationContext("spring.xml");
```

- **web程序入口**

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring.xml</param-value>
</context-param>
<listener>
    <listener-class>
    	org.springframework.web.context.ContextLoaderListener
    </listener-class>
</listener>
```

不管上面那种方式，最终都会调AbstractApplicationContext的refresh方法，而这个方法才是我们的入口

#### 流程分析

**AbstractApplicationContext的refresh方法**

```java
public void refresh() throws BeansException, IllegalStateException {
synchronized (this.startupShutdownMonitor) {
    // Prepare this context for refreshing.
    // STEP 1： 刷新预处理
    prepareRefresh();
    // Tell the subclass to refresh the internal bean factory.
    // STEP 2：
    // a） 创建IoC容器（DefaultListableBeanFactory）
    // b） 加载解析XML文件（最终存储到Document对象中）
    // c） 读取Document对象，并完成BeanDefinition的加载和注册工作
    ConfigurableListableBeanFactory beanFactory =
    obtainFreshBeanFactory();
    // Prepare the bean factory for use in this context.
    // STEP 3： 对IoC容器进行一些预处理（设置一些公共属性）
    prepareBeanFactory(beanFactory);
    try {
        // Allows post-processing of the bean factory in context
        subclasses.
        // STEP 4：
        postProcessBeanFactory(beanFactory);
        // Invoke factory processors registered as beans in the
        context.
        // STEP 5： 调用BeanFactoryPostProcessor后置处理器对
        BeanDefinition处理
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
        // Initialize other special beans in specific context
        subclasses.
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
            logger.warn("Exception encountered during context
            initialization - " +
            "cancelling refresh attempt: " + ex);
    	}
        // Destroy already created singletons to avoid dangling
        resources.
        destroyBeans();
        // Reset 'active' flag.
        cancelRefresh(ex);
        // Propagate exception to caller.
        throw ex;
    }
    finally {
            // Reset common introspection caches in Spring's core,since we
            // might not ever need metadata for singleton beansanymore...
            resetCommonCaches();
        }
    }
}
```

### 创建BeanFactory流程源码分析

#### 找入口

**AbstractApplicationContext的refresh方法**

```java
// Tell the subclass to refresh the internal bean factory.
// STEP 2：
// a） 创建IoC容器（DefaultListableBeanFactory）
// b） 加载解析XML文件（最终存储到Document对象中）
// c） 读取Document对象，并完成BeanDefinition的加载和注册工作
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

#### 流程解析

- **进入AbstractApplication的obtainFreshBeanFactory方法**

  用于创建一个Ioc容器，这个Ioc容器**就是DefaultListableBeanFactory对象**

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 主要是通过该方法完成IoC容器的刷新
    refreshBeanFactory();
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " +
        beanFactory);
    }
    return beanFactory;
}
```

- **进入AbstractRefreshableApplicationContext的refreshBeanFactory方法**

  销毁以前的容器

  创建新的IOC容器

  加载BeanDefinition对象注册到Ioc容器中

```java
protected final void refreshBeanFactory() throws BeansException {
    //如果之前有Ioc容器，则销毁
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
            //创建IOC容器，也就是DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
            //加载BeanDefinition对象，并注册到IOC容器中（重点）
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

- **进入AbstractRefreshableApplicationContext的createBeanFactory方法**

```java
protected DefaultListableBeanFactory createBeanFactory() {
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}
```

### 加载BeanDefinition流程分析

#### 找入口

AbstractRefreshableApplicationContext类的refreshBeanFactory方法中的第13行代码：

```java
protected final void refreshBeanFactory() throws BeansException {
    //如果之前有Ioc容器，则销毁
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
            //创建IOC容器，也就是DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
            //加载BeanDefinition对象，并注册到IOC容器中（重点）
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

**BeanDefinition流程**

```
|--AbstractRefreshableApplicationContext#refreshBeanFactory
	|--AbstractXmlApplicationContext#loadBeanDefinitions
		|--AbstractBeanDefinitionReader#loadBeanDefinitions
			|--XmlBeanDefinitionReader#loadBeanDefinitions
				|--XmlBeanDefinitionReader#doLoadBeanDefinitions
					|--XmlBeanDefinitionReader#registerBeanDefinitions
						|--DefaultBeanDefinitionDocumentReader#registerBeanDefinitions
							|--#doRegisterBeanDefinitions
								|--#parseBeanDefinitions
									|--BeanDefinitionParserDelegate#parseCustomElement
```

#### **流程相关类的说明**

- **AbstractRefreshableApplicationContext**

  主要用来对BeanFactory提供refresh功能。包括BeanFacotry的创建和BeanDefinition的定义、解析、注册操作

- **AbstractXmlApplicationContext**

  主要提供对于XML资源的加载功能。包括从Resource资源对象和资源路径中加载XML文件

- **AbstractBeanDefinitionReader**

  主要提供对于BeanDefinition对象的读取功能。具体读取工作交给子类实现

- **XmlBeanDefinitionReader**

  主要通过DOM4J对于XML资源的读取、解析功能，并提供对于BeanDefinition的注册功能。

- **DefaultBeanDefinitionDocumentReader**

- **BeanDefinitionParserDelegate**

#### 流程解析

- **进入AbstractXmlApplicationContext的loadBeanDefinitions方法**

  创建一个XmlBeanDefinitionReader，通过阅读XML文件，真正完成BeanDefinition的加载和注册

  配置XmlBeanDefinitionReader并进行初始化

  委托给XmlBeanDefinitionReader去加载BeanDefinition

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    //Create a new XmlBeanDefinitionReader for the given BeanFactory
    //作用：通过阅读XML文件，真正完成BeanDefinition的加载和注册
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    //Configure the bean definition reader with this context's resource loading evironment
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    //Allow a subclass to provide custom initialization of the reader,
    //then proceed with actually loading the bean definition
        this.initBeanDefinitionReader(beanDefinitionReader);
    //委托给BeanDefinition阅读器去加载BeanDefinition
        this.loadBeanDefinitions(beanDefinitionReader);
    }
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    //获取资源的定位
    //这里getConfigResources是一个空实现，真正实现是调用子类的获取资源定位的方法
    //比如：ClassPathXmlApplicationContext中进行了实现
    //而FileSystemXmlApplicationContext没有使用该方法
        Resource[] configResources = this.getConfigResources();
        if (configResources != null) {
            //XML Bean读取器调用其父类AbstractBeanDefinitionReader读取定位资源
            reader.loadBeanDefinitions(configResources);
        }
		//如果子类中获取的资源为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置资源
        String[] configLocations = this.getConfigLocations();
        if (configLocations != null) {
            //XML Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的资源
            reader.loadBeanDefinitions(configLocations);
        }

    }
```

- **loadBeanDefinitions方法经过一路的兜兜转转，最终来到了XmlBeanDefinitionReader的doLoadBeanDefinitions方法**

  一个是对XML文件进行DOM解析

  一个是完成BeanDefinition对象的加载与注册

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) throws BeanDefinitionStoreException {
        try {
            //通过DOM4J加载解析XML文件，最终形成Document对象
            Document doc = this.doLoadDocument(inputSource, resource);
            //通过对Document对象的操作，完成BeanDefinition的加载和注册工作
            int count = this.registerBeanDefinitions(doc, resource);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Loaded " + count + " bean definitions from " + resource);
            }

            return count;
        }
    	//省略一些catch语句
        catch (Throwable ex) {
            ......
        }
    }
```

- **此处我们暂不处理DOM4J加载解析XML的流程，我们重点分析BeanDefinition的加载注册流程**

- **进入XmlBeanDefinitionReader的registerBeanDefinitions方法**

  创建DefaultBeanDefinitionDocumentReader用来解析Document对象

  获得容器中已注册的BeanDefinition数量

  委托给DefaultDefinitionDocumentReader来完成BeanDefinition的加载注册工作

  统计新注册的BeanDefinition数量

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    //创建DefaultBeanDefinitionDocumentReader用来解析Document对象
        BeanDefinitionDocumentReader documentReader = this.createBeanDefinitionDocumentReader();
    //获得容器中注册Bean数量
        int countBefore = this.getRegistry().getBeanDefinitionCount();
    //解析过程入口，BeanDefinitionDocumentReader只是个接口
    //具体的实现过程在DefaultBeanDefinitionDocumentReader完成
        documentReader.registerBeanDefinitions(doc, this.createReaderContext(resource));
    //统计注册的Bean数量
        return this.getRegistry().getBeanDefinitionCount() - countBefore;
    }
```

- **进入DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法：**

  获得Document的跟元素标签

  真正实现BeanDefinition解析和注册工作

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
        this.readerContext = readerContext;
    //实现BeanDefinition解析和注册工作
        this.doRegisterBeanDefinitions(doc.getDocumentElement());
    }
```

- **进入DefaultBeanDefinitionDocumentReader的doRegisterBeanDefinitions方法：**

  这里使用了委托模式，将具体的BeanDefinition解析工作交给了BeanDefinitionParserDelegate去完成

  在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性

  委托给BeanDefinitionParserDelegate，从Document的跟元素开始进行BeanDefinition的解析

  在解析Bean定义后，进行自定义解析，增加解析过程的可扩展性

```java
protected void doRegisterBeanDefinitions(Element root) {
    //这里使用了委托模式，将具体的BeanDefinition解析工作交给了BeanDefinitionParserDelegate去完成
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = this.createDelegate(this.getReaderContext(), root, parent);
        if (this.delegate.isDefaultNamespace(root)) {
            String profileSpec = root.getAttribute("profile");
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, ",; ");
                if (!this.getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (this.logger.isDebugEnabled()) {
                        this.logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec + "] not matching: " + this.getReaderContext().getResource());
                    }

                    return;
                }
            }
        }
		//在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性
        this.preProcessXml(root);
    	//委托给BeanDefinitionParserDelegate，从Document的跟元素开始进行
        this.parseBeanDefinitions(root, this.delegate);
    	//在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
        this.postProcessXml(root);
        this.delegate = parent;
    }
```

- **进入DefaultBeanDefinitionDocumentReader的parseBeanDefinitions方法**

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

- **此时开始正式的解析xml文件并加载BeanDefinition**

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		// 解析<import>标签
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		// 解析<alias>标签
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		// 解析<bean>标签
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		// 解析内置<beans>标签
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			// 递归调用
			doRegisterBeanDefinitions(ele);
		}
	}
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		// 获取命名空间URI（就是获取beans标签的xmlns:aop或者xmlns:context属性的值）
		// http://www.springframework.org/schema/aop
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
		// 根据不同的命名空间URI,去匹配不同的NamespaceHandler(一个命名空间对应一个NamespaceHandler)
		// 此处会调用DefaultNamespaceHandlerResolver类的resolve方法
		// 两步操作：查找NamespaceHandler 、调用NamespaceHandler的init方法进行初始化（针对不同自定义标签注册相应的BeanDefinitionParser）
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		// 调用匹配到的NamespaceHandler的解析方法
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```



### Bean实例化流程分析

#### 找入口

**AbstractApplicationContext的refresh方法**

```java
//Instantiate all remaining(non-lazy-init) singletons
//STEP 11 实例化剩余的单例bean（非懒加载凡是）
//注意事项：Bean的IOC/DI/AOP都是发生在此步骤
this.finishBeanFactoryInitialization(beanFactory);
```

#### 解析流程

**进入AbstractApplicationContext的finishBeanFactoryInitialization方法**

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		// 实例化单例Bean
		beanFactory.preInstantiateSingletons();
	}
```

**进入DefaultListableBeanFactory的preInstantiateSingletons方法**

```java
public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		// 触发所有非懒加载方式的单例bean的创建
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			// 如果bean不是抽象的，而且是单例的，同时还不是懒加载的，则进行下面的操作
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				// 如果bean是一个FactoryBean，则走下面的方法
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else { // 普通bean走下面的方法
					getBean(beanName);
				}
			}
		}
```

**进入上层类AbstractBeanFactory的doGetBean方法**

doGetBean方法很复杂，主流程为先到缓存取bean，如果已经存在直接返回，不存在则需要创建，这里我们着重看try-catch里面的创建bean的代码

```java
try {
				// 获取要实例化的bean的BeanDefinition对象
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				// 检查该BeanDefinition对象对应的Bean是否是抽象的
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				// 如果是单例的Bean，请下面的代码
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							// 创建单例Bean的主要方法
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
```

**重点看单例bean的创建，进入AbstractAutowireCapableBeanFactory的createBean方法**

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		// 处理方法覆盖
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			// 获取BeanPostProcessor代理对象
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			// 完成Bean实例的创建（实例化、填充属性、初始化）
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
```

**进入doCreateBean方法，主要流程分3步实例化、属性填充、初始化**

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		// bean初始化第一步：默认调用无参构造实例化Bean
		// 构造参数依赖注入，就是发生在这一步
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		// 实例化后的Bean对象
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		// 解决循环依赖的关键步骤
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		// 如果需要提前暴露单例Bean，则将该Bean放入三级缓存中
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 将刚创建的bean放入三级缓存中singleFactories(key是beanName，value是FactoryBean)
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			// bean初始化第二步：填充属性（DI依赖注入发生在此步骤）
			// 调用反射和内省去进行属性设置
			// 属性值需要进行类型转换
			populateBean(beanName, mbd, instanceWrapper);
			// bean初始化第三步：调用初始化方法，完成bean的初始化操作（AOP发生在此步骤）
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

