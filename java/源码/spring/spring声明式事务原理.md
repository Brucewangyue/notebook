## @Transactional

SpringBoot大行其道的今天，基于XML配置的Spring Framework的使用方式注定已成为过去式。 

注解驱动应用，面向元数据编程已然成受到越来越多开发者的偏好了，毕竟它的便捷程度、优势都是XML方式不可比拟的。 

对SpringBoot有多了解，其实就是看你对Spring Framework有多熟悉~ 比如SpringBoot大量的模块装配的设计模式，其实它属于Spring Framework提供的能力

1、开启注解驱动

```java

```

```
spring-tx.jar META-INF spring.handlers

org.springframework.transaction.config.TxNamespaceHandler
```

```
org.springframework.beans.factory.xml.AbstractBeanDefinitionParser # parse

org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser # parseInternal
```

```java
protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
	BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
	String parentName = getParentName(element);
	if (parentName != null) {
		builder.getRawBeanDefinition().setParentName(parentName);
	}
    // org.springframework.transaction.config.TxAdviceBeanDefinitionParser 返回 TransactionInterceptor.class;
	Class<?> beanClass = getBeanClass(element);
	if (beanClass != null) {
		builder.getRawBeanDefinition().setBeanClass(beanClass);
	}
	else {
		String beanClassName = getBeanClassName(element);
		if (beanClassName != null) {
			builder.getRawBeanDefinition().setBeanClassName(beanClassName);
		}
	}
	builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
	BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
	if (containingBd != null) {
		// Inner bean definition must receive same scope as containing bean.
		builder.setScope(containingBd.getScope());
	}
	if (parserContext.isDefaultLazyInit()) {
		// Default-lazy-init applies to custom bean definitions as well.
		builder.setLazyInit(true);
	}
    // TxAdviceBeanDefinitionParser # doParse
	doParse(element, parserContext, builder);
	return builder.getBeanDefinition();
}
```

```java
@Override
protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
	builder.addPropertyReference("transactionManager", TxNamespaceHandler.getTransactionManagerName(element));
    // 解析tx:Attributes标签
	List<Element> txAttributes = DomUtils.getChildElementsByTagName(element, ATTRIBUTES_ELEMENT);
	if (txAttributes.size() > 1) {
		parserContext.getReaderContext().error(
				"Element <attributes> is allowed at most once inside element <advice>", element);
	}
	else if (txAttributes.size() == 1) {
		// Using attributes source.
		Element attributeSourceElement = txAttributes.get(0);
        // 解析tx:Attributes标签内的method标签
		RootBeanDefinition attributeSourceDefinition = parseAttributeSource(attributeSourceElement, parserContext);
		builder.addPropertyValue("transactionAttributeSource", attributeSourceDefinition);
	}
	else {
		// Assume annotations source.
		builder.addPropertyValue("transactionAttributeSource",
				new RootBeanDefinition("org.springframework.transaction.annotation.AnnotationTransactionAttributeSource"));
	}
}
```

解析tx:Attributes标签

解析 tx:method标签

创建NameMatchTransactionAttributeSource Bean定义，放入 TransactionInterceptor



此时端点到refresh最后，查看生成好的实例对象

<img src="assets/image-20220515233414419.png" alt="image-20220515233414419" style="zoom:80%;" />



在invokeBeanFactoryPostProcessors中 处理配置文件中的占位符

![image-20220515234016085](assets/image-20220515234016085.png)

![image-20220515234042821](assets/image-20220515234042821.png)

通过以下BFPP处理

![image-20220515234331842](assets/image-20220515234331842.png)



Spring事务是基于AOP的，所以会首先通过 AspectJAwareAdvisorAutoProxyCreator 创建Advisor Advice等





<img src="assets/image-20220516000637672.png" alt="image-20220516000637672" style="zoom:80%;" />



创建Advisor前先创建advice

<img src="assets/image-20220516000921601.png" alt="image-20220516000921601" style="zoom:80%;" />

具体参考AOP源码



在遇到第一个需要被创建动态代理的类时，开始实例化 TransactionInterceptor

AspectJProxyUtils # isAspectJAdvice(advisor)  #  advisor.getAdvice()
