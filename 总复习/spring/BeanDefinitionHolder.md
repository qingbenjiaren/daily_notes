# BeanDefinitionHodler

BeanDefinitionHodler顾名思义就是BeanDefinition的持有者，同时持有包括BeanDefinition的名称和别名。

先看下BeanDefinitionHolder的源码

```java
/**
 * Holder for a BeanDefinition with name and aliases.
 * Can be registered as a placeholder for an inner bean.
 *
 * <p>Can also be used for programmatic registration of inner bean
 * definitions. If you don't care about BeanNameAware and the like,
 * registering RootBeanDefinition or ChildBeanDefinition is good enough.
 *
 * @author Juergen Hoeller
 * @since 1.0.2
 * @see org.springframework.beans.factory.BeanNameAware
 * @see org.springframework.beans.factory.support.RootBeanDefinition
 * @see org.springframework.beans.factory.support.ChildBeanDefinition
 */
public class BeanDefinitionHolder implements BeanMetadataElement {

    //持有BeanDefinition
	private final BeanDefinition beanDefinition;
	//BeanDefinition的名称
	private final String beanName;
	//BeanDefinition的别名
	@Nullable
	private final String[] aliases;


	/**
	 * Create a new BeanDefinitionHolder.
	 * @param beanDefinition the BeanDefinition to wrap
	 * @param beanName the name of the bean, as specified for the bean definition
	 */
    //根据bean的名称和beandefinition初始化BeanDefinitionHolder
	public BeanDefinitionHolder(BeanDefinition beanDefinition, String beanName) {
        //内部时继是调用的另外一个构造方法
		this(beanDefinition, beanName, null);
	}

	/**
	 * Create a new BeanDefinitionHolder.
	 * @param beanDefinition the BeanDefinition to wrap
	 * @param beanName the name of the bean, as specified for the bean definition
	 * @param aliases alias names for the bean, or {@code null} if none
	 */
	public BeanDefinitionHolder(BeanDefinition beanDefinition, String beanName, @Nullable String[] aliases) {
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");
		Assert.notNull(beanName, "Bean name must not be null");
		this.beanDefinition = beanDefinition;
		this.beanName = beanName;
		this.aliases = aliases;
	}

	/**
	 * Copy constructor: Create a new BeanDefinitionHolder with the
	 * same contents as the given BeanDefinitionHolder instance.
	 * <p>Note: The wrapped BeanDefinition reference is taken as-is;
	 * it is {@code not} deeply copied.
	 * @param beanDefinitionHolder the BeanDefinitionHolder to copy
	 */
    //根据BeanDefinitionHolder拷贝一个新的Holder，注意，不是深拷贝
	public BeanDefinitionHolder(BeanDefinitionHolder beanDefinitionHolder) {
		Assert.notNull(beanDefinitionHolder, "BeanDefinitionHolder must not be null");
		this.beanDefinition = beanDefinitionHolder.getBeanDefinition();
		this.beanName = beanDefinitionHolder.getBeanName();
		this.aliases = beanDefinitionHolder.getAliases();
	}


	/**
	 * Return the wrapped BeanDefinition.
	 */
	public BeanDefinition getBeanDefinition() {
		return this.beanDefinition;
	}

	/**
	 * Return the primary name of the bean, as specified for the bean definition.
	 */
	public String getBeanName() {
		return this.beanName;
	}

	/**
	 * Return the alias names for the bean, as specified directly for the bean definition.
	 * @return the array of alias names, or {@code null} if none
	 */
	@Nullable
	public String[] getAliases() {
		return this.aliases;
	}

	/**
	 * Expose the bean definition's source object.
	 * @see BeanDefinition#getSource()
	 */
	@Override
	@Nullable
	public Object getSource() {
		return this.beanDefinition.getSource();
	}

	/**
	 * Determine whether the given candidate name matches the bean name
	 * or the aliases stored in this bean definition.
	 */
	public boolean matchesName(@Nullable String candidateName) {
		return (candidateName != null && (candidateName.equals(this.beanName) ||
				candidateName.equals(BeanFactoryUtils.transformedBeanName(this.beanName)) ||
				ObjectUtils.containsElement(this.aliases, candidateName)));
	}


	/**
	 * Return a friendly, short description for the bean, stating name and aliases.
	 * @see #getBeanName()
	 * @see #getAliases()
	 */
	public String getShortDescription() {
		StringBuilder sb = new StringBuilder();
		sb.append("Bean definition with name '").append(this.beanName).append("'");
		if (this.aliases != null) {
			sb.append(" and aliases [").append(StringUtils.arrayToCommaDelimitedString(this.aliases)).append("]");
		}
		return sb.toString();
	}

	/**
	 * Return a long description for the bean, including name and aliases
	 * as well as a description of the contained {@link BeanDefinition}.
	 * @see #getShortDescription()
	 * @see #getBeanDefinition()
	 */
	public String getLongDescription() {
		StringBuilder sb = new StringBuilder(getShortDescription());
		sb.append(": ").append(this.beanDefinition);
		return sb.toString();
	}

	/**
	 * This implementation returns the long description. Can be overridden
	 * to return the short description or any kind of custom description instead.
	 * @see #getLongDescription()
	 * @see #getShortDescription()
	 */
	@Override
	public String toString() {
		return getLongDescription();
	}


	@Override
	public boolean equals(Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof BeanDefinitionHolder)) {
			return false;
		}
		BeanDefinitionHolder otherHolder = (BeanDefinitionHolder) other;
		return this.beanDefinition.equals(otherHolder.beanDefinition) &&
				this.beanName.equals(otherHolder.beanName) &&
				ObjectUtils.nullSafeEquals(this.aliases, otherHolder.aliases);
	}

	@Override
	public int hashCode() {
		int hashCode = this.beanDefinition.hashCode();
		hashCode = 29 * hashCode + this.beanName.hashCode();
		hashCode = 29 * hashCode + ObjectUtils.nullSafeHashCode(this.aliases);
		return hashCode;
	}

}
```

继续看看BeanDefinitionHolder在spring中的应用，DefaultBeanDefinitionDocumentReader中在解析默认的bena标签的时候：

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		// 解析<bean>标签，获取BeanDefinition
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			// 如果需要，则装饰BeanDefinition对象
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				// 注册最终的BeanDefinition到BeanDefinitionRegistry（DefaultListableBeanFactory）
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

我们在开发中也可以借鉴BeanDefinitionHolder 这种方式，更好的封装我们的数据。