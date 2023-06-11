+++

author = "旅店老板"
title = "SpringBoot原理(二):Starter"
date = "2023-05-22"
description = "关于SpringBoot中Starter的原理探索"
tags = [
	"SpringBoot",
]
categories = [
    "java",
"SpringBoot"
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "SpringBoot.png"
mermaid = true
+++
## 引言
SpringBoot内置了各种Starter的起步依赖，有了Starter我们不需要考虑项目需要什么库，版本会不会冲突等，大大减轻了我们的工作。  

比如我们经典的web应用在引入`spring-boot-starter-web`后,不需要再考虑`spring-web`和`spring-webmvc`及它们兼容的版本。  

## Maven的optional标签
关于RabbitMq在SpringBoot源码中有下面这样一个配置类，如下所示：
```javascript
import org.springframework.amqp.rabbit.core.RabbitTemplate;

@Configuration
@ConditionalOnClass(RabbitTemplate.class)
public class RabbitHealthContributorAutoConfiguration {
    
}
```
省略了大部分代码，剩余一个条件注解`@ConditionalOnClass`，如不了解条件注解请阅读[SpringBoot原理(一):自动配置](http://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E4%B8%80%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE/) ，该注解表示
RabbitTemplate这个类存在时自动注入`RabbitHealthContributorAutoConfiguration`实例。  

看到这里有些同学可能会有疑惑，既然该类可能不存在，直接使用`RabbitTemplate.class`，并使用`import`关键词导入的类路径不会爆红吗？打包项目时不会报错吗？  

像这样书写`@ConditionalOnClass(name = "org.springframework.amqp.rabbit.core.RabbitTemplate")`通过反射加载字符串判断类是否存在不是才正确吗？  

先说答案，两种方式都是正确的，为什么第一种方式类路径不爆红呢，我们通过ide点进`RabbitTemplate`发现SpringBoot是引入了对应依赖的，那为什么还要判断该类是否存在呢，关键在于引入的方式，引入的pom依赖如下：
```markdown
		<dependency>
			<groupId>org.springframework.amqp</groupId>
			<artifactId>spring-rabbit</artifactId>
			<optional>true</optional>
		</dependency>
```
该依赖在`spring-boot-actuator-autoconfigure`模块的pom中被引入，该依赖有一个`<optional>true</optional>`标签，该标签为`true`表示项目打包时不引入该依赖，
这就是可以使用且不暴红的原因。  

显然易见，倘若使用到该处代码会发生notFoundClassException，因此要使用rabbitmq需主动引入相关依赖，引入依赖后再在`application.yml`配置参数后就能使用`@Autowired`获取对应实例。  

通过上面一个例子我们说明了`<optional>true</optional>`标签的作用，`<exclusions></exclusions>`也有类似的功能。
***
## Starter构建原理
我们可以看到SringBoot源码中`spring-boot-build/spring-boot-project/spring-boot-starters`目录下有大量的Starter，以最常用的`spring-boot-starter-web`起步依赖来说明。  

`spring-boot-build/spring-boot-project/spring-boot-starters/spring-boot-starter-web`目录下没有代码，只有一个pom.xml,简略内容如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starters</artifactId>
		<version>${revision}</version>
	</parent>
	<artifactId>spring-boot-starter-web</artifactId>
	<name>Spring Boot Web Starter</name>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-json</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
			<exclusions>
				<exclusion>
					<groupId>org.apache.tomcat.embed</groupId>
					<artifactId>tomcat-embed-el</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
		</dependency>
	</dependencies>
</project>
```
可以看出，`spring-boot-starter-web`模块依赖了`spring-boot-starter`,`spring-boot-starter-tomcat`,`spring-web`和`spring-webmvc`等模块。  

`spring-boot-starter`模块是绝大部分`spring-boot-starter-xxx`模块依赖的基础模块。其中`spring-boot-starter`依赖了`spring-boot-autoconfigure`，因此
`spring-boot-starter-web`间接依赖了`spring-boot-autoconfigure`
## @ConfigurationProperties
经过上面的学习，我们明白了Starter引入依赖就自动配置的原因是**条件注解**+**`<optional>true</optional>`标签**。    

只引入一个Starter依赖是因为Starter本身已配置好对应依赖，例如`spring-boot-starter-web`。    

这些都是maven的功能，Starter还有一项重要的能力，在`application.yml`中配置对应的参数，实例能自动获取到值并配置，比如我们在`application.properties`配置文件中配置`server.port=8081`，
该值会自动绑定到类`ServerProperties`的属性`port`上 。这是怎么实现的呢？可以推测与`spring-boot-autoconfigure`有很大的相关性。  

细心的我们发现类`ServerProperties`上有注解`@ConfigurationProperties`,与之关联的注解有`@EnableConfigurationProperties`,接下来我们研究下两个注解的原理。  
***
### @ConfigurationProperties和@EnableConfigurationProperties
`ServerProperties`的源码如下：
```java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {

	private Integer port;

	private InetAddress address;
	//......
}
```
可以看到`ServerProperties`类上标注的`@ConfigurationProperties`注解，配置前缀为`server`，`ignoreUnknownFields`是否忽略未知值为`true`  

注解`@ConfigurationProperties`的源码为：
```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {

	@AliasFor("prefix")
	String value() default "";

	String prefix() default "";

	boolean ignoreInvalidFields() default false;
    
	boolean ignoreUnknownFields() default true;
}
```
该注解有四个字段，其中`value`为`prefix`的别名,`ignoreInvalidFields`表示忽略无效的配置，默认值为false。    

`@ConfigurationProperties`这个注解的作用就是将外部配置的配置值绑定到其注解的类的属性上，可以作用于配置类或配置类的方法上。  

可以发现这个注解没有任何处理逻辑，是一个标志性注解，代码调用入口不在这里。端口和地址是属于服务器配置，与之关联的代码为：
```java
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
//......
public class ServletWebServerFactoryAutoConfiguration {
//......
}
```
这是一个ServletWeb相关的配置类，对应的条件注解我们也能明白其作用，重点不在这里，该类上有一个`@EnableConfigurationProperties`注解，它的value值就是我们的服务器参数
配置类`ServerProperties.class`,这个注解的作用应该就是为`@ConfigurationProperties`注解绑定值提供支持。它是如何起作用的呢？  

`@EnableConfigurationProperties`的源码如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EnableConfigurationPropertiesRegistrar.class)
public @interface EnableConfigurationProperties {

	String VALIDATOR_BEAN_NAME = "configurationPropertiesValidator";

	Class<?>[] value() default {};
}
```
该注解上有一个重要注解`@Import(EnableConfigurationPropertiesRegistrar.class)`，对于这种代码是否有些熟悉呢，如果你详细了解
[SpringBoot原理(一):自动配置](http://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E4%B8%80%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE/)中`@SpringBootApplication`注解
那部分。  

`@Import`导入的类`EnableConfigurationPropertiesRegistrar`的源码为：
```java
class EnableConfigurationPropertiesRegistrar implements ImportBeanDefinitionRegistrar {
}
```
`ImportBeanDefinitionRegistrar`接口是SpringBoot对外提供的扩展接口，被导入时会调用该接口的方法，我们知道一个普通的bean被配置到容器有两个大的步骤，
第一步被`@Component`、`@bean`、`Import`注解的类和`META-INF/spring.factories`中配置的类会被包装成`BeanDefinition`实例存储到`beanFactory`中，
只有一些特殊的bean比如后置处理器会被提前实例化；第二步调用最后的`finishBeanFactoryInitialization`方法才是将`BeanDefinition`实例实例化为真正的bean  

`ImportBeanDefinitionRegistrar`接口会在第一步扫描到配置类时注册为`BeanDefinition`实例，同时也会获取该类上的`@Import`导入的`ImportBeanDefinitionRegistrar`实现类，并执行对应的方法，
可以在此时额外注册一些`BeanDefinition`实例(如果你对这部分感兴趣，请继续阅读SpringBoot原理系列文章)  

回到代码，这里在将`x`注册为`BeanDefinition`实例时，会执行实现类`EnableConfigurationPropertiesRegistrar`的
`registerBeanDefinitions`方法，对应源码为：
```java
	public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		registerInfrastructureBeans(registry);
		ConfigurationPropertiesBeanRegistrar beanRegistrar = new ConfigurationPropertiesBeanRegistrar(registry);
		getTypes(metadata).forEach(beanRegistrar::register);
	}
```

这里就不进行具体分析，该方法首先注册一个后置处理器`BeanPostProcessor`的实例`ConfigurationPropertiesBindingPostProcessor`，然后会扫描项目中的`@EnableConfigurationProperties`注解,例如`@EnableConfigurationProperties(ServerProperties.class)`等，
将它们注册成`BeanDefinition`实例  

在将`BeanDefinition`实例真正实例化时会执行这个后置处理器为注解中的class的bean实例绑定值，如`ServerProperties.class`实例  

真正绑定的逻辑在`ConfigurationPropertiesBindingPostProcessor`这里后置处理器中，它只重写了后置处理器的一个接口`postProcessBeforeInitialization`，初始化之前进行绑定，源码如下:

```java
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		bind(ConfigurationPropertiesBean.get(this.applicationContext, bean, beanName));
		return bean;
	}
```
核心就在这个bind方法，请自行分析  

## 小结
通过上面的介绍我们了解了Starter特性的原理，但我们可能对两种不同的后置处理器`BeanFactoryPostProcessor`和`BeanPostProcessor`在SpirngBoot启动过程中的调用时机，
不了解一个注解(`@Component`、`@bean`等等）标识的类何时注册成`BeanDefinition`实例，何时将`BeanDefinition`实例变为真正的bean。  

如果你对上诉问题感兴趣，请结合SpringBoot的源码和 [SpringBoot原理(一):自动配置](http://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E4%B8%80%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE/) 、
 [SpringBoot原理(三):启动流程分析](http://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E4%B8%89%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/) 、
 [SpringBoot原理(四):常用注解分析](http://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E5%9B%9B%E5%B8%B8%E7%94%A8%E6%B3%A8%E8%A7%A3%E5%88%86%E6%9E%90/) 进行调试学习。





