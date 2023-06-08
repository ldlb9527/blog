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