## 源码入口





## 自动装配原理

### **简介** 

不知道大家第一次搭SpringBoot环境的时候，有没有觉得非常简单。无须各种的配置文件，无须各种繁杂的pom坐标，一个main方法，就能run起来了。与其他框架整合也贼方便，使用EnableXXXXX注解就可以搞起来了！所以今天来讲讲SpringBoot是如何实现自动配置的

### **关键注解** 

**自动配置流程图** 

https://www.processon.com/view/link/5fc0abf67d9c082f447ce49b 

**从启动类开始入手**

**@SpringBootApplication**

标注在某个类上说明这个类是SpringBoot的主配置类，SpringBoot需要运行这个类的main方法来启动SpringBoot应用

```java
@Target(ElementType.TYPE)
// 当注解标注的类编译以什么方式保留
//	RetentionPolicy.RUNTIME: 会被jvm加载
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
// 标注在某个类上，表示这是一个Spring Boot的配置类
//	内部使用: @Configuration
//		内部使用: @Component
@SpringBootConfiguration
// 开启自动配置功能，以前我们需要配置的东西，Spring Boot帮我们自动配置(自动去加载'自动配置类')；
@EnableAutoConfiguration
// 相当于在spring.xml 配置中<context:comonent-scan>，但是并没有指定basepackage
// 如果没有指定，spring底层会自动扫描当前配置类所有在的包
@ComponentScan(excludeFilters = { 
    // springboot对外提供的扩展类， 可以供我们去按照我们的方式进行排除
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    // 排除所有配置类并且是自动配置类中里面的其中一个
	@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ...
}
```

**@EnableAutoConfiguration**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
// 将当前配置类所在包保存在BasePackages的Bean中。供Spring内部使用
//	内部使用：@Import(AutoConfigurationPackages.Registrar.class)
//		注册了一个保存当前配置类所在包的一个Bean
@AutoConfigurationPackage
// 关键点
// AutoConfigurationImportSelector 实现了 DeferredImportSelector
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
...
}
```

spring内部在解析@import的时候会调用selectImports # getAutoConfigurationEntry方法进行扫描具有META-INF/spring.factories文件的jar包

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 从META‐INF/spring.factories中获得候选的自动配置类
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 排重
	configurations = removeDuplicates(configurations);
    // 根据EnableAutoConfiguration注解中属性，获取不需要自动装配的类名单，一般不用
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
    // 通过读取spring.factories 中的OnBeanCondition\OnClassCondition\OnWebApplicationCondition进行过滤
	configurations = getConfigurationClassFilter().filter(configurations);
    // 实例化实现了AutoConfigurationImportListener的bean.
    // 循环调用onAutoConfigurationImportEvent方法
    // 分别把候选的配置名单，和排除的配置名单传进去做扩展
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
```

**spring.factories文件**

存在两个包中：spring-boot、spring-boot-autoconfigure

spring.factories文件是Key=Value形式，多个Value时使用‘,’隔开，该文件中定义了关于初始化，监听器等信息，而真正使自动配置生效的key是org.springframework.boot.autoconfigure.EnableAutoConfiguration

```properties
## spring-boot-configure
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
...
```

每一个这样的 xxxAutoConfiguration类都是容器中的一个组件，都加入到容器中；用他们来做自动配置

[springboot所有自动配置类列表](https://docs.spring.io/spring-boot/docs/current/reference/html/auto-configuration-classes.html#appendix.auto-configuration-classes)

后续： @EnableAutoConfiguration注解通过@SpringBootApplication被间接的标记在了Spring Boot的启动类上。在 SpringApplication.run(...)的内部就会执行selectImports()方法，找到所有JavaConfig自动配置类的全限定名对应的class，然后将所有自动配置类加载到Spring容器中

### HttpEncodingAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(ServerProperties.class)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnClass(CharacterEncodingFilter.class)
@ConditionalOnProperty(prefix = "server.servlet.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration {

	private final Encoding properties;

	public HttpEncodingAutoConfiguration(ServerProperties properties) {
		this.properties = properties.getServlet().getEncoding();
	}

	@Bean
	@ConditionalOnMissingBean
	public CharacterEncodingFilter characterEncodingFilter() {
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Encoding.Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Encoding.Type.RESPONSE));
		return filter;
	}

	@Bean
	public LocaleCharsetMappingsCustomizer localeCharsetMappingsCustomizer() {
		return new LocaleCharsetMappingsCustomizer(this.properties);
	}

	static class LocaleCharsetMappingsCustomizer
			implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>, Ordered {

		private final Encoding properties;

		LocaleCharsetMappingsCustomizer(Encoding properties) {
			this.properties = properties;
		}

		@Override
		public void customize(ConfigurableServletWebServerFactory factory) {
			if (this.properties.getMapping() != null) {
				factory.setLocaleCharsetMappings(this.properties.getMapping());
			}
		}

		@Override
		public int getOrder() {
			return 0;
		}

	}

}

```

**注解说明：**

- @Configuration

  标记了@Configuration，Spring底层会给配置创建cglib动态代理。 作用：就是防止每次调用本类的@Bean标记的方法而重新创建对象，@Bean是默认单例的

- @EnableConfigurationProperties

  - ServerProperties.class是标注了@ConfigurationProperties的类，最终会将配置文件中的配置项注入到ServerProperties.class的属性

**我们怎么知道哪些自动配置类生效?**

我们可以通过设置配置文件中：启用 debug=true属性；来让控制台打印自动配置报告，这样我们就可以很方便的知道哪些自动配置类生效

```sh
 Positive matches:‐‐‐**表示自动配置类启用的** 
 ‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐ 
 ...省略... 
 
 Negative matches:‐‐‐**没有匹配成功的自动配置类** 
 ‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐‐ 
 ...省略...
```





## @Import原理



## @EnableConfigurationProperties

将用@ConfigurationProperties标记了的类放入spring容器，且会将配置注入到该类的属性



## @Conditional原理

必须是@Conditional指定的条件成立，才给容器中添加组件，配置配里面的所有内容才生效；

| 扩展注解作用                    | （判断是否满足当前指定条件）                     |
| ------------------------------- | ------------------------------------------------ |
| @ConditionalOnJava              | 系统的java版本是否符合要求                       |
| @ConditionalOnBean              | 容器中存在指定Bean；                             |
| @ConditionalOnMissingBean       | 容器中不存在指定Bean；                           |
| @ConditionalOnExpression        | 满足SpEL表达式指定                               |
| @ConditionalOnClass             | 系统中有指定的类                                 |
| @ConditionalOnMissingClass      | 系统中没有指定的类                               |
| @ConditionalOnSingleCandidate   | 容器中只有一个指定的Bean，或者这个Bean是首选Bean |
| @ConditionalOnProperty          | 系统中指定的属性是否有指定的值                   |
| @ConditionalOnResource          | 类路径下是否存在指定资源文件                     |
| @ConditionalOnWebApplication    | 当前是web环境                                    |
| @ConditionalOnNotWebApplication | 当前不是web环境                                  |
| @ConditionalOnJndi              | JNDI存在指定项                                   |

