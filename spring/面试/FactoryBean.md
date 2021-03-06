# FactoryBean

在我们的开发工作中应该都见过见过或使用过FactoryBean这个类，也许你会看成了BeanFactory这个类。FactoryBean和BeanFactory虽然长得很像，但是他们的作用却是完全不像。这里可以想象以下，你会在什么样的场景下使用FactoryBean这个接口？FactoryBean是一个工厂Bean，可以生成某一个类型的Bean实例，它最大的一个作用是：<font color='red'> 可以让我们自定义Bean的创建过程。</font>BeanFactory是Spring容器中的一个基本类也是很重要的一个类，在BeanFactory中可以创建和管理Spring容器中的Bean，它对于Bean的创建有一个统一的流程。下面我们先看一下FactoryBean中有什么东西。

```java
public interface FactoryBean<T>{
    //返回的对象实例
    T getObject() throws Exception;
    //Bean的类型
    Class<?> getObjectType();
    //true
    boolean isSingleton();
}
```

从上面的代码中我们发现在FactoryBean中定义了一个Spring Bean的很重要的三个特性：是否单例、Bean类型、Bean实例，这也应该是我们关于Spring中的一个Bean最直观的感受。虽然没有返回BeanName的值，但是我们也知道BeanName的值。下面我们来写一个关于FactoryBean的小例子，看看我们是怎么使用FactoryBean的，然后再从源码的角度看看Spring是怎么解析FactoryBean中的Bean的。

```java
@Component
public class FactoryBeanLearn implements FactoryBean {
    @Override
    public Object getObject() throws Exception {
        //自定义bean构造过程
        return new SomeServiceImpl();
    }

    @Override
    public Class<?> getObjectType() {
        return com.melo.bean.SomeService.class;
    }
}

public interface SomeService {
    public void sayHello();
}
public class SomeServiceImpl implements SomeService {
    @Override
    public void sayHello() {
        System.out.println("Hello spring FactoryBean");
    }
}
public interface OtherService {
    public void sayHello();
}

@Service
public class OtherServiceImpl implements OtherService {
    @Override
    public void sayHello() {
        System.out.println("let us look look");
    }
}

public class Test {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
        for(String s : context.getBeanNamesForType(FactoryBean.class)){
            System.out.println(s);
        }
        for(String s : context.getBeanNamesForType(OtherService.class)){
            System.out.println(s);
        }
        for(String s : context.getBeanNamesForType(SomeService.class)){
            System.out.println(s);
        }
        SomeService service = context.getBean(com.melo.bean.SomeService.class);
        service.sayHello();
    }
}



&factoryBeanLearn
otherServiceImpl
factoryBeanLearn
Hello spring FactoryBean

```

从上面的代码中我们可以看到我们从Spring容器中获取了FactoryBeanService类型的Bean。那么这个获取Bean的过程Spring是怎么处理的呢？它是怎么从FactoryBean中获取我们自己创建的Bean实例的呢？我们先从getBean这个方法看起，因为在Spring的AbstractApplicationContext中有很多重载的getBean方法，这里我们调用的是根据Type(指Class类型)来获取的Bean信息。我们传入的type是FactoryBeanService类型。

## getBean

AbstractApplicationContext#getBean(java.lang.Class)

```java
@Override
    public <T> T getBean(Class<T> requiredType) throws BeansException {
        //检测BeanFactory的激活状态
        assertBeanFactoryActive();
        //getBeanFactory()获取到的是一个DefaultListableBeanFactory的实例
        //所以我们去DefaultListableBeanFactory中看一下getBean这个方法
        return getBeanFactory().getBean(requiredType);
    }
```

DefaultListableBeanFactory#getBean(java.lang.Class)

```java
@Override
    public <T> T getBean(Class<T> requiredType) throws BeansException {
        return getBean(requiredType, (Object[]) null);
    }
    @Override
    public <T> T getBean(Class<T> requiredType, Object... args) throws BeansException {
        //解析Bean
        NamedBeanHolder<T> namedBean = resolveNamedBean(requiredType, args);
        if (namedBean != null) {
            return namedBean.getBeanInstance();
        }
        //如果当前Spring容器中没有获取到相应的Bean信息，则从父容器中获取
        //SpringMVC是一个很典型的父子容器
        BeanFactory parent = getParentBeanFactory();
        if (parent != null) {
            //一个重复的调用过程，只不过BeanFactory的实例变了
            return parent.getBean(requiredType, args);
        }
        //如果都没有获取到，则抛出异常
        throw new NoSuchBeanDefinitionException(requiredType);
    }
```

在上面的代码中，我们重点关注的是resolveNamedBean这个方法：

```java
private <T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType, Object... args) throws BeansException {
        Assert.notNull(requiredType, "Required type must not be null");
        //这个方法是根据传入的Class类型来获取BeanName，因为我们有一个接口有多个实现类的情况(多态)，
        //所以这里返回的是一个String数组。这个过程也比较复杂。
        //这里需要注意的是，我们调用getBean方法传入的type为com.zkn.spring.learn.service.SomeService类型，但是我们没有在Spring容器中注入SomeService类型的Bean
        //正常来说我们在这里是获取不到beanName呢。但是事实是不是这样呢？看下面我们对getBeanNamesForType的分析
        String[] candidateNames = getBeanNamesForType(requiredType);
        //如果有多个BeanName，则挑选合适的BeanName
        if (candidateNames.length > 1) {
            List<String> autowireCandidates = new ArrayList<String>(candidateNames.length);
            for (String beanName : candidateNames) {
                if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
                    autowireCandidates.add(beanName);
                }
            }
            if (!autowireCandidates.isEmpty()) {
                candidateNames = autowireCandidates.toArray(new String[autowireCandidates.size()]);
            }
        }
        //如果只有一个BeanName 我们调用getBean方法来获取Bean实例来放入到NamedBeanHolder中
        //这里获取bean是根据beanName，beanType和args来获取bean
        //这里是我们要分析的重点 在下一篇文章中有介绍
        if (candidateNames.length == 1) {
            String beanName = candidateNames[0];
            return new NamedBeanHolder<T>(beanName, getBean(beanName, requiredType, args));
        }
        //如果合适的BeanName还是有多个的话
        else if (candidateNames.length > 1) {
            Map<String, Object> candidates = new LinkedHashMap<String, Object>(candidateNames.length);
            for (String beanName : candidateNames) {
                //看看是不是已经创建多的单例Bean
                if (containsSingleton(beanName)) {
                    candidates.put(beanName, getBean(beanName, requiredType, args));
                }
                else {
                    //调用getType方法继续获取Bean实例
                    candidates.put(beanName, getType(beanName));
                }
            }
            //有多个Bean实例的话 则取带有Primary注解或者带有Primary信息的Bean
            String candidateName = determinePrimaryCandidate(candidates, requiredType);
            if (candidateName == null) {
                //如果没有Primary注解或者Primary相关的信息，则去优先级高的Bean实例
                candidateName = determineHighestPriorityCandidate(candidates, requiredType);
            }
            if (candidateName != null) {
                Object beanInstance = candidates.get(candidateName);
                //Class类型的话 继续调用getBean方法获取Bean实例
                if (beanInstance instanceof Class) {
                    beanInstance = getBean(candidateName, requiredType, args);
                }
                return new NamedBeanHolder<T>(candidateName, (T) beanInstance);
            }
            //都没有获取到 抛出异常
            throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
        }

        return null;
    }

```

在上面的代码中我们说我们传入的type是com.zkn.spring.learn.service.SomeService类型，但是在我们的Spring容器中却没有SomeService类型的Bean，那么我们是怎么从getBeanNamesForType获取到beanName的呢？ 
getBeanNamesForType的分析

```java
public String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
        if (!isConfigurationFrozen() || type == null || !allowEagerInit) {
            return doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, allowEagerInit);
        }
        //先从缓存中获取
        Map<Class<?>, String[]> cache =
                (includeNonSingletons ? this.allBeanNamesByType : this.singletonBeanNamesByType);
        String[] resolvedBeanNames = cache.get(type);
        if (resolvedBeanNames != null) {
            return resolvedBeanNames;
        }
        //调用doGetBeanNamesForType方法获取beanName
        resolvedBeanNames = doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, true);
        //所传入的类能不能被当前类加载加载
        if (ClassUtils.isCacheSafe(type, getBeanClassLoader())) {
            //放入到缓存中，解析一次以后从缓存中获取
            //这里对应到我们这里 key是FactoryBeanService Value是beanFactoryLearn
            cache.put(type, resolvedBeanNames);
        }
        return resolvedBeanNames;
    }
```

doGetBeanNamesForType的分析

```java
private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
        List<String> result = new ArrayList<String>();
        //循环所有的beanName 这个是在Spring容器启动解析Bean的时候放入到这个List中的
        for (String beanName : this.beanDefinitionNames) {
            //不是别名
            if (!isAlias(beanName)) {
                try {
                    //根据beanName获取RootBeanDefinition
                    RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                    //RootBeanDefinition中的Bean不是抽象类、非延迟初始化 
                    if (!mbd.isAbstract() && (allowEagerInit ||
                            ((mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading())) &&
                                    !requiresEagerInitForType(mbd.getFactoryBeanName()))) {
                        //是不是FactoryBean的子类
                        boolean isFactoryBean = isFactoryBean(beanName, mbd);
                        BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
                        //这里我们其他的几个变量的意思都差不多  我们需要重点关注的是isTypeMatch这个方法 
                        //如果isTypeMatch这个方法返回true的话，我们会把这个beanName即factoryBeanLearn 放入到result中返回
                        boolean matchFound =
                                (allowEagerInit || !isFactoryBean ||
                                        (dbd != null && !mbd.isLazyInit()) || containsSingleton(beanName)) &&
                                (includeNonSingletons ||
                                        (dbd != null ? mbd.isSingleton() : isSingleton(beanName))) &&
                                isTypeMatch(beanName, type);
                        if (!matchFound && isFactoryBean) {
                            //如果不匹配，还是FactoryBean的子类 这里会把beanName变为 &beanName
                            beanName = FACTORY_BEAN_PREFIX + beanName;
                            //判断类型匹配不匹配
                            matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
                        }
                        if (matchFound) {
                            result.add(beanName);
                        }
                    }
                }
            }
        }

        //这里的Bean是Spring容器创建的特殊的几种类型的Bean 像Environment
        for (String beanName : this.manualSingletonNames) {
            try {
                // In case of FactoryBean, match object created by FactoryBean.
                if (isFactoryBean(beanName)) {
                    if ((includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type)) {
                        result.add(beanName);
                        // Match found for this bean: do not match FactoryBean itself anymore.
                        continue;
                    }
                    // In case of FactoryBean, try to match FactoryBean itself next.
                    beanName = FACTORY_BEAN_PREFIX + beanName;
                }
                // Match raw bean instance (might be raw FactoryBean).
                if (isTypeMatch(beanName, type)) {
                    result.add(beanName);
                }
            }
        }

        return StringUtils.toStringArray(result);
    }
```

在上面的代码中，我们可以知道的是我们的FactoryBeanLearn是一个FactoryBean类型的类。所以在上面的代码中会调用isTypeMatch这个方法来判断FactoryBeanLearn是不是和我们传入的类型相匹配。这里是值：FactoryBeanService类。我们去isTypeMatch方法中去看一下它是怎么进行类型匹配判断的：由于isTypeMatch方法很长，这里我们只看和我们这次分析相关的一部分代码

```java
 public boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException {
        //转换beanName   这里我们可以知道我们的beanName为factoryBeanLearn 因为上面是循环了Spring容器中的所有的Bean
        String beanName = transformedBeanName(name);
        //因为我们这里是用的AbstractApplicationContext的子类来从Spring容器中获取Bean
        //获取beanName为factoryBeanLearn的Bean实例 这里是可以获取到Bean实例的
        //这里有一个问题：使用AbstractApplicationContext的子类从Spring容器中获取Bean和
        //使用BeanFactory的子类从容器中获取Bean有什么区别？这个可以思考一下
        Object beanInstance = getSingleton(beanName, false);
        if (beanInstance != null) {
            //factoryBeanLearn是FactoryBean的一个实现类
            if (beanInstance instanceof FactoryBean) {
                //这里判断beanName是不是以&开头  这里明显不是 这里可以想一下什么情况下会有&开头的Bean
                if (!BeanFactoryUtils.isFactoryDereference(name)) {
                    //这里就是从factoryBeanLearn中获type类型 我们在下面会分析一下这个类
                    Class<?> type = getTypeForFactoryBean((FactoryBean<?>) beanInstance);
                    //从factoryBeanLearn中获取到的type类型和我们传入的类型是不是同一种类型 是的话直接返回
                    return (type != null && typeToMatch.isAssignableFrom(type));
                }
                else {
                    return typeToMatch.isInstance(beanInstance);
                }
            }
```

getTypeForFactoryBean

```java
protected Class<?> getTypeForFactoryBean(final FactoryBean<?> factoryBean) {
        try {
            if (System.getSecurityManager() != null) {
                return AccessController.doPrivileged(new PrivilegedAction<Class<?>>() {
                    @Override
                    public Class<?> run() {
                        return factoryBean.getObjectType();
                    }
                }, getAccessControlContext());
            }
            else {
                //看到这里是不是很熟悉了 调用FactoryBean实例的getObjectType()方法
                return factoryBean.getObjectType();
            }
        }
    }
```



我们在调用factoryBeanLearn的getObjectType方法的时候，获取到的值为：com.zkn.spring.learn.service.FactoryBeanService和我们传入的type是一样的类型。所以这里返回true，根据我们上面说的如果isTypeMatch返回true的话，我们返回的beanName为factoryBeanLearn。 

上面的分析总结起来是：我们调用getBean(Class requiredType)方法根据类型来获取容器中的bean的时候，对应我们的例子就是：根据类型com.zkn.spring.learn.service.FactoryBeanService来从Spring容器中获取Bean(首先明确的一点是在Spring容器中没有FactoryBeanService类型的BeanDefinition。但是却有一个Bean和FactoryBeanService这个类型有一些关系)。Spring在根据type去获取Bean的时候，会先获取到beanName。获取beanName的过程是：先循环Spring容器中的所有的beanName，然后根据beanName获取对应的BeanDefinition，如果当前bean是FactoryBean的类型，则会从Spring容器中根据beanName获取对应的Bean实例，接着调用获取到的Bean实例的getObjectType方法获取到Class类型，判断此Class类型和我们传入的Class是否是同一类型。如果是则返回测beanName，对应到我们这里就是：根据factoryBeanLearn获取到FactoryBeanLearn实例，调用FactoryBeanLearn的getObjectType方法获取到返回值FactoryBeanService.class。和我们传入的类型一致，所以这里获取的beanName为factoryBeanLearn。换句话说这里我们把factoryBeanLearn这个beanName映射为了：FactoryBeanService类型。**即SomeService类型对应的beanName为factoryBeanLearn**这是很重要的一点。



# Aware接口

Aware这个单词翻译过来就是知道，感知的意思。在spring中，它的常见的子接口，比如BeanNameAware、BeanFactoryAware、ApplicationContextAware接口。假设我们的类继承了BeanNameAware这个接口，对应这个接口有一个方法setBeanName的方法，spring在依赖注入的初始化阶段会调用生成对象的这个方法，把beanName传为入参传进来。一般我们在会自己写的类里面定义一个属性来接收这个beanName，然后这个beanName我们就可以在开发中使用了。

给实现这个接口的类，注入一个属性，根据策略模式，Aware的类型，选择注入一个属性，谁实现了Aware谁就有了属性的引用

个人理解，因为我们创建的bean继承BeanFactory接口，所以在我们编写这个bean的实现的时候，就要实现setBeanFactory方法。然后再spring为我们创建该bean的时候，会判断我们是不是实现Aware接口，若是，则会为我们注入相应的属性，我们bean创建好了之后就自然用了了spring为我们创建的属性，例如：beanFactory



# 循环依赖问题

什么是循环依赖：

循环依赖是针对成员变量----单例才可以解决setter方法循环依赖，多例是无法解决循环依赖。----依靠缓存解决循环依赖，多例不存缓存

构造方法循环依赖--------无法解决，只能将构造依赖改为setter方法依赖

只能将构造依赖改为setter方法依赖

循环调用是针对方法---无法解决的

getBean流程
		先去缓存中获取对象的bean
			先去一级缓存查找-----------value：完全初始化完毕的Bean实例
			没有再去二级缓存查找----------value：只完成bean实例化的bean的实例
			没有再去三级缓存查找----------value：ObjectFactory。通过该缓存将只完成bean实例化的bean的实例提前暴露（不只是将bean的引用放入缓存中，还需要对该而初始化完成的bean进行beanPostProcessor----动态代理---AOP）
		如果bean没有再去创建
		并且将创建之后的bean放入缓存中

疑问一：为什么要有三级缓存：
	主要是通过二级和三级缓存来解决循环依赖的问题，其中二级缓存和三级缓存是互斥的
疑问二：
	三级缓存和二级缓存的区别？
	二级缓存只要存储beanName和提前暴露的bean实例的映射关系即可
	三级缓存不仅需要对提前暴露的bean进行返回，还要对该bean做后置处理（beanPostProcessor）。
	三级缓存将暴露的bean处理完之后，将暴露的bean转移到二级缓存，同时删除三级缓存的数据。
	三级缓存才是解决循环依赖的根本。
	

	A对象依赖B对象、B对象依赖A对象
	
		1、A对象实例化
			在1和2之间需要做一个处理，会将刚创建的bean通过beanFactory进行吧暴露，并且将ObjectFactory放入三级缓存
		2、A对象属性填充
			A对象给B对象属性进行赋值，需要获取B对象的值getBean()
				B对象实例化
				B对象的属性填充
					B对象给A对象属性进行赋值，需要获取A对象的值getBean()
					直接从缓存中取出对应的A对象？
				B对象的初始化
		3、A对象初始化


​		

		引用是可以传递的，提前暴露的Bean和完成初始化成功的Bean其实是一个对象

​	