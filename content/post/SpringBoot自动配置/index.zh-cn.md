+++

author = "旅店老板"
title = "SpringBoot一之自动配置"
date = "2023-05-22"
description = "了解SpringBoot自动配置原理及其源码学习"
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
# 开始之前
* 需要SpringBoot的demo项目，用于验证自动配置
* 需要熟悉java注解的相关的知识
* SpringBoot 2.2.13RELEASE的源码调试环境(该文章所使用的源码都来自该版本)  
下载地址为：[https://github.com/spring-projects/spring-boot/tree/v2.2.13.RELEASE](https://github.com/spring-projects/spring-boot/tree/v2.2.13.RELEASE)
***
# SpringBoot源码目录结构说明
* **spring-boot-build**:项目根目录
* **spring-boot-build/Spring-boot-tests**：这个模块SpringBoot的测试模块，跟部署测试和集成测试有关。
* **spring-boot-build/spring-boot-project**：SpringBoot框架的源码目录，因此我们重点分析该目录
  * **spring-boot**: 该模块是SpringBoot项目的核心，包含启动类`SpringApplication`,外部传参支持`java -jar test.jar --server.port=80`,以及各种启动的初始化逻辑
  * **spring-boot-parent**: 这个模块是SpringBoot的父项目，被其他模块依赖，该模块下没有代码
  * **spring-boot-actuator**：
# Spring Boot自动配置原理
## 条件注解
SpringBoot的自动配置是需要满足一定条件才会进行自动配置的，这与注解`ConditionalOnXXX`密切相关。下面先以一个具体的例子进行说明，再探究原理。
### ConditionalOnAuto
首先开发一个Spring Boot的demo项目，
我们自定义一个注解`ConditionalOnAuto`,当该注解的method字段为`auto`时自动配置bean
* `ConditionalOnAuto`注解代码如下：
```java
package com.cx.config;

public @interface ConditionalOnAuto {
    String method() default "";
    String value() default "";
}
```
该注解有两个属性：`method`和`value`
***
* 定义类`AutoCondition`实现`Condition`接口，实现其`matches()`方法,代码如下：
```java
package com.cx.config;

import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
import org.springframework.util.MultiValueMap;

public class AutoCondition implements Condition {
    @Override
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata annotatedTypeMetadata) {

        MultiValueMap<String, Object> map = annotatedTypeMetadata.getAllAnnotationAttributes(ConditionalOnAuto.class.getName());
        Object o = map.getFirst("method");
        if ("auto".equals(o)) {
            return true;
        }
        return false;
    }
}

```
这个map的key为注解的字段,value为注解的值，类型是LinkedList

这里我们定义的字段为字符串，使用`getFirst`获取第一个值即可，当method值为`auto`返回true，否则返回false

`matches`方法返回true时才会创建对应的bean
***
* 给`ConditionalOnAuto`注解添加`Conditional`注解,绑定Condition接口的实现类，`ConditionalOnAuto`注解修改后代码如下：
```java
package com.cx.config;

import org.springframework.context.annotation.Conditional;

//Conditional注解的value需填写Condition接口实现类的class
@Conditional(AutoCondition.class)
public @interface ConditionalOnAuto {
    String method() default "";
    String value() default "";
}
```
此时注解`@ConditionalOnAuto`是注解`@Conditional`的派生注解，与`@Conditional(AutoCondition.class)`是等价的
***
* 下面测试自定义注解是否生效，定义资源如下：

首先创建任意类`A`,代码如下：
```java
package com.cx.config;

public class A {
}
```
创建配置类`AConfig`,添加`@ConditionalOnAuto`注解，代码如下：
```java
package com.cx.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AConfig {

    @Bean
    @ConditionalOnAuto(value = "aaaaa", method = "auto")
    public A a() {
        return new A();
    }
}
```
测试时我们会修改`@ConditionalOnAuto`注解的`method`的值

创建测试接口`TestController`，代码如下：
```java
package com.cx;

import com.cx.config.A;
import io.prometheus.client.Counter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController {

    @Autowired(required = false)
    private A a;

    @RequestMapping("test1")
    public void test1() {
        System.out.println("a的地址为："+a);
    }

}
```
当注解`@ConditionalOnAuto`的`method`的值为`auto`时，访问接口控制台打印为`a的地址为：com.cx.config.A@619f2afc`

当注解`@ConditionalOnAuto`的`method`的值不为`auto`时，访问接口控制台打印为`a的地址为：null`，符合我们的预期。
***
* 我们来梳理下上述测试代码的逻辑：
    1. `AutoCondition`实现了`Condition`接口的`matches`方法，获取注解`ConditionalOnAuto`的对应值判断是否创建bean(返回true创建)
    2. 根据注解的定义形式，`@ConditionalOnAuto`与`@Conditional(AutoCondition.class)`是等价的
    3. 最后我们定义了一个配置类`AConfig`,对应方法添加注解`@ConditionalOnAuto(value = "aaaaa", method = "auto")`
    4. 所以`@ConditionalOnAuto`注解真正起作用的是Condition接口的具体实现类`AutoCondition`的`matches`方法
    5. Springboot在启动时执行了`Condition`接口的实现类`AutoCondition`的`matches`方法

>上面提到，`@ConditionalOnAuto`与`@Conditional(AutoCondition.class)`是等价的。既然等价，当然可以直接替换。
>
>但是，上面实现的`matches`方法依赖`@ConditionalOnAuto`的字段的值，无法直接进行替换，倘若`matches`方法是从yaml中获取值，两个注解就可以正常替换了
>
>根据上述代码，我们能模糊地感受到为什么springboot引入中间件后只需要yaml中进行简单配置就能直接用`@AutoWire`进行使用了
***
### SpringBootCondition
上一节我们自定义注解并最终实现了`Condition`接口，Spring Boot有没有内置Condition接口实现类呢？有的，`SpringBootCondition`类

`SpringBootCondition`类位于`spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition/SpringBootCondition.java`

有很多`OnXXXCondition`类继承了`SpringBootCondition`类，它们对应的注解为`@ConditionalOnXXX`
* `SpringBootCondition`的`matches`方法源码如下：
```java
package org.springframework.boot.autoconfigure.condition;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.core.type.ClassMetadata;
import org.springframework.core.type.MethodMetadata;
import org.springframework.util.ClassUtils;
import org.springframework.util.StringUtils;

public abstract class SpringBootCondition implements Condition {

	private final Log logger = LogFactory.getLog(getClass());

	@Override
	public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		String classOrMethodName = getClassOrMethodName(metadata);
		try {
			ConditionOutcome outcome = getMatchOutcome(context, metadata);
			logOutcome(classOrMethodName, outcome);
			recordEvaluation(context, classOrMethodName, outcome);
			return outcome.isMatch();
		}
		catch (NoClassDefFoundError ex) {
			throw new IllegalStateException("Could not evaluate condition on " + classOrMethodName + " due to "
					+ ex.getMessage() + " not found. Make sure your own configuration does not rely on "
					+ "that class. This can also happen if you are "
					+ "@ComponentScanning a springframework package (e.g. if you "
					+ "put a @ComponentScan in the default package by mistake)", ex);
		}
		catch (RuntimeException ex) {
			throw new IllegalStateException("Error processing condition on " + getName(metadata), ex);
		}
	}
    //......
}
```
* `SpringBootCondition`是一个抽象类，`matches`的实现代码较少，有五个方法：
    1. `getClassOrMethodName`获取类名和方法名，逻辑为如果是`ClassMetadata`类型则返回类名，否则返回类名+方法名，具体实现如下：
  ```java
  private static String getClassOrMethodName(AnnotatedTypeMetadata metadata) {
        if (metadata instanceof ClassMetadata) {
            ClassMetadata classMetadata = (ClassMetadata) metadata;
            return classMetadata.getClassName();
        }
        MethodMetadata methodMetadata = (MethodMetadata) metadata;
        return methodMetadata.getDeclaringClassName() + "#" + methodMetadata.getMethodName();
    }

  ```
    2. `getMatchOutcome`是`SpringBootCondition`的抽象方法，具体实现在子类`OnXXXCondition`中，具体逻辑是判断自身配置类的条件注解`@ConditionalOnXXX`是否满足条件，然后记录到ConditionOutcome中
    3. `logOutcome(classOrMethodName, outcome);`作用是打印Condition是否满足条件的日志，如匹配会打印`matched`,不匹配会打印`did not match`
    4. `recordEvaluation(context, classOrMethodName, outcome);`是否匹配的评估信息记录到`ConditionEvaluationReport`中
    5. `outcome.isMatch()`直接调用了outcome的match字段，满足条件则返回true，否则返回false，子类实现`getMatchOutcome`时会给该字段赋值
    6. 即`SpringBootCondition`封装了一个模板方法`getMatchOutcome(context, metadata)`供子类`OnXXXCondition`实现具体的判断逻辑，因此`SpringBootCondition`的`matches`方法的作用除了调用`getMatchOutcome(context, metadata)`外，
       就是打印和记录是否满足匹配条件的评估信息
***
### OnResourceCondition
`OnResourceCondition`继承`SpringBootCondition`实现了其`getMatchOutcome`方法，对应的注解为`@ConditionalOnResource`

它们的源代码都位于`spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition/`下
* 注解`@ConditionalOnResource`的源代码如下：
```java
package org.springframework.boot.autoconfigure.condition;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.context.annotation.Conditional;

@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnResourceCondition.class)
public @interface ConditionalOnResource {

	/**
	 * The resources that must be present.
	 * @return the resource paths that must be present.
	 */
	String[] resources() default {};

}
```
注解中有个`resources`,根据前面的经验，该字段要满足一定条件才会创建对应的bean
***
* `OnResourceCondition`的源码实现了`getMatchOutcome`,代码如下：
```java
package org.springframework.boot.autoconfigure.condition;

import java.util.ArrayList;
import java.util.List;

import org.springframework.boot.autoconfigure.condition.ConditionMessage.Style;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.type.AnnotatedTypeMetadata;
import org.springframework.util.Assert;
import org.springframework.util.MultiValueMap;

/**
 * {@link Condition} that checks for specific resources.
 *
 * @author Dave Syer
 * @see ConditionalOnResource
 */
@Order(Ordered.HIGHEST_PRECEDENCE + 20)
class OnResourceCondition extends SpringBootCondition {

	@Override
	public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
		//attributes这个map获取了@ConditionalOnResource的值，key为resource  value是List<Object>,实际类型是LinkedList
		MultiValueMap<String, Object> attributes = metadata
				.getAllAnnotationAttributes(ConditionalOnResource.class.getName(), true);
		ResourceLoader loader = context.getResourceLoader();
		
		List<String> locations = new ArrayList<>();
		//collectValues将List<Object>中每个元素转为字符串，存入locations中
		collectValues(locations, attributes.get("resources"));
		Assert.isTrue(!locations.isEmpty(),
				"@ConditionalOnResource annotations must specify at least one resource location");
		
		//遍历locations,依次加载对应路径的资源，不存在则加入missing集合
		List<String> missing = new ArrayList<>();
		for (String location : locations) {
			String resource = context.getEnvironment().resolvePlaceholders(location);
			if (!loader.getResource(resource).exists()) {
				missing.add(location);
			}
		}
		//如果missing集合不为空，创建ConditionOutcome对象，mathch设置为false,Message记录不存在资源的路径，并返回
		if (!missing.isEmpty()) {
			return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnResource.class)
					.didNotFind("resource", "resources").items(Style.QUOTE, missing));
		}
		//执行到这里表示missing集合为空，创建ConditionOutcome对象，mathch设置为true
		return ConditionOutcome.match(ConditionMessage.forCondition(ConditionalOnResource.class)
				.found("location", "locations").items(locations));
	}

	private void collectValues(List<String> names, List<Object> values) {
		for (Object value : values) {
			for (Object item : (Object[]) value) {
				names.add((String) item);
			}
		}
	}

}
```
`OnResourceCondition`的代码逻辑与我们自定义实现大同小异，实现逻辑也比较简单

`OnResourceCondition`的判断逻辑是拿到`@ConditionalOnResource`注解指定的资源路径后，然后用ResourceLoader根据指定路径去加载看资源存不存在

注解中的资源路径是一个List，所有路径都存在是返回true
* 源码中的其他条件注解举例：
```java
package org.springframework.boot.autoconfigure.data.mongo;

import com.mongodb.reactivestreams.client.MongoClient;
//......
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ MongoClient.class, ReactiveMongoTemplate.class })
@ConditionalOnBean(MongoClient.class)
@EnableConfigurationProperties(MongoProperties.class)
@Import(MongoDataConfiguration.class)
@AutoConfigureAfter(MongoReactiveAutoConfiguration.class)
public class MongoReactiveDataAutoConfiguration {
    
	@Bean
	@ConditionalOnMissingBean(ReactiveMongoDatabaseFactory.class)
	public SimpleReactiveMongoDatabaseFactory reactiveMongoDatabaseFactory(MongoProperties properties,
			MongoClient mongo) {
		String database = properties.getMongoClientDatabase();
		return new SimpleReactiveMongoDatabaseFactory(mongo, database);
	}

	@Bean
	@ConditionalOnMissingBean(ReactiveMongoOperations.class)
	public ReactiveMongoTemplate reactiveMongoTemplate(ReactiveMongoDatabaseFactory reactiveMongoDatabaseFactory,
			MongoConverter converter) {
		return new ReactiveMongoTemplate(reactiveMongoDatabaseFactory, converter);
	}
}   
```
上面是springboot中mongo的配置源码，根据我们的学习，基本能够直接明白`@ConditionalOnClass({ MongoClient.class, ReactiveMongoTemplate.class })`、`@ConditionalOnBean(MongoClient.class)`
和`@ConditionalOnMissingBean(ReactiveMongoDatabaseFactory.class)`的作用,它们的区别只是有些作用于类上，有些作用于方法上

我们自定义的条件类也可以不直接实现Condition接口，而直接继承SpringBootCondition
> Springboot中有大量的形如`@ConditionalOnXXX`的条件注解，它们的实现类为`OnXXXCondition`
>
> Springboot就是根据这些派生的条件注解进行自动配置的，Springboot是在何时执行那些Condition接口实现类的matches方法呢？我们在后面分析
>
***
## @SpringBootApplication注解
先看一下下面这段代码：
```java
@SpringBootApplication
public class TestApplication {

    public static void main(String[] args) {
        SpringApplication.run(TestApplication.class, args);
    }

}
```
我们对上面代码非常熟悉，它是SpringBoot应用的启动类，并标有`@SpringBootApplication`注解
* `@SpringBootApplication`是SpringBoot的重要注解，与自动配置息息相关，源代码如下：
```java
//......
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
  @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    //......
}
```
该注解包含多个其他注解，顾名思义，`@EnableAutoConfiguration`该注解与自动配置有关
* `@EnableAutoConfiguration`注解源码如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    
    //启用自动配置时可用于覆盖的环境属性
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    
    //排除特定的自动配置类，这样它们就永远不会被应用。
	Class<?>[] exclude() default {};
    
    //排除特定的自动配置类名，这样它们就永远不会被应用
	String[] excludeName() default {};
}
```
该注解上标有`@AutoConfigurationPackage`和`@Import(AutoConfigurationImportSelector.class)`这两个注解

这里可以猜测自动配置可能与类`AutoConfigurationImportSelector`有关

下面我们来分析`AutoConfigurationImportSelector`类和`@AutoConfigurationPackage`注解
***
* `AutoConfigurationImportSelector`实现了`DeferredImportSelector`接口，`DeferredImportSelector`接口又继承了`ImportSelector`接口，即实现了该接口的`selectImports`方法,
  `selectImports`的源代码如下：
  ```java
  public String[] selectImports(AnnotationMetadata annotationMetadata) {
      if (!isEnabled(annotationMetadata)) {
          return NO_IMPORTS;
      }
      AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
              .loadMetadata(this.beanClassLoader);
      AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
              annotationMetadata);
      return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
  }
  ```
  `selectImports`方法的具体逻辑由四个方法组成，我们来进行逐个分析：
    1. 首先**isEnabled**方法返回false，`selectImports`方法直接返回`NO_IMPORTS`字符串数组，即不导入，`isEnabled`方法代码如下：
  ```javascript
  protected boolean isEnabled(AnnotationMetadata metadata) {
    if (getClass() == AutoConfigurationImportSelector.class) {
      return getEnvironment().getProperty(EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class, true);
    }
    return true;
  }
  ```
  判断是否是AutoConfigurationImportSelector类型，如果不是返回true，如果是则从environment获取`spring.boot.enableautoconfiguration`，为null则取默认值true

  即当我们显示指定`spring.boot.enableautoconfiguration`为false时关闭自动配置
    2. **loadMetadata**方法会获取`META-INF/spring-autoconfigure-metadata.properties`的值
    3. **getAutoConfigurationEntry**方法获取了需要导入的配置类集合和需要排除的配置类集合，**该方法极为重要，请参照注释在自己的源码环境一行行调试阅读**，代码如下：
  ```java
      protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
              AnnotationMetadata annotationMetadata) {
          if (!isEnabled(annotationMetadata)) {
              return EMPTY_ENTRY;
          }
          //获取EnableAutoConfiguration中的参数，exclude()/excludeName()
          AnnotationAttributes attributes = getAttributes(annotationMetadata);
  
          //会获取META-INF/spring.factories文件中需要自动装配的配置类，
          //文件内容形如org.springframework.boot.autoconfigure.EnableAutoConfiguration=/com.xx.libs.middleware.DBSource
          //还会获取SpringBoot默认的自动配置类，如RabbitAutoConfiguration，JdbcTemplate,Redis等大概200多个
          //具体逻辑是从一个map缓存中获取，key为org.springframework.boot.autoconfigure.EnableAutoConfiguration,第一次没有值，则去生成这两种自动配置类路径，并存入缓存
          //该函数被多个地方调用，缓存cache不只与自动配置有关，还有各种Listener、Initializer等等，attributes参数传入后并没有使用，不知道原因
          //这里获取的自动配置类包括SpringBoot默认的+META-INF/spring.factories中的
          List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
  
          //通过set去重再转为list
          configurations = removeDuplicates(configurations);
  
          //获取exclude和excludeName的值，返回set是两者可能有重复
          Set<String> exclusions = getExclusions(annotationMetadata, attributes);
  
          //检查需要排除的类是否再configurations中，不在的筛选到一个集合，此集合不为空的话抛出异常,异常中说明这些类不需要排除，因为它们不会自动配置
          //即不能排除一个不会自动配置的类
          checkExcludedClasses(configurations, exclusions);
  
          //从configurations去除需要排除的类集合exclusions
          configurations.removeAll(exclusions);
  
          //autoConfigurationMetadata是外面在META-INF/spring-autoconfigure-metadata.properties获取并传入的
          //filter对configurations进行过滤，剔除掉不满足 spring-autoconfigure-metadata.properties所写条件的配置类
          configurations = filter(configurations, autoConfigurationMetadata);
  
          //监听器import事件回调，向每个监听者发送configurations和exclusions组合成event的评估信息？
          fireAutoConfigurationImportEvents(configurations, exclusions);
  
          //返回configurations和exclusions组，
          //注意此时configurations已经排除了exclusions并过滤了不满足META-INF/spring-autoconfigure-metadata.properties的配置类
          return new AutoConfigurationEntry(configurations, exclusions);
      }
  ```
  `filter`方法的作用也是过滤，源码如下：
  ```java
      private List<String> filter(List<String> configurations, AutoConfigurationMetadata autoConfigurationMetadata) {
          long startTime = System.nanoTime();
          String[] candidates = StringUtils.toStringArray(configurations);
          boolean[] skip = new boolean[candidates.length];
          boolean skipped = false;
          //分割线上部分只是处理需要过滤的路径，逻辑较简单，请自行阅读
          //=============================================================================================
          for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
              invokeAwareMethods(filter);
              boolean[] match = filter.match(candidates, autoConfigurationMetadata);
              for (int i = 0; i < match.length; i++) {
                  if (!match[i]) {
                      skip[i] = true;
                      candidates[i] = null;
                      skipped = true;
                  }
              }
          }
          //==============================================================================================
          if (!skipped) {
              return configurations;
          }
          List<String> result = new ArrayList<>(candidates.length);
          for (int i = 0; i < candidates.length; i++) {
              if (!skip[i]) {
                  result.add(candidates[i]);
              }
          }
          if (logger.isTraceEnabled()) {
              int numberFiltered = configurations.size() - result.size();
              logger.trace("Filtered " + numberFiltered + " auto configuration class in "
                      + TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime) + " ms");
          }
          return new ArrayList<>(result);
      }
  ```
  回顾一下，configurations中是自动配置的类路径  
  for循环getAutoConfigurationImportFilters()获取了一个filter集合，并调用了match方法返回一个布尔数组，对每个类路径而言，必须匹配所有filter才能不被过滤，即返回的布尔数组都为true。    
  `getAutoConfigurationImportFilters()`的外层源码如下：
  ```java
      protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
          return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
      }
  ```
    * 如果仔细阅读过前面从缓存cache中获取自动配置类的源码，也是调用的`SpringFactoriesLoader.loadFactories()`
    * 这里传入了一个`AutoConfigurationImportFilter.class`,内层代码会获取类路径为`org.springframework.boot.autoconfigure.AutoConfigurationImportFilter`,以该路径为key，也从缓存cache中进行获取
    * 这时cache已经有值了，类型为LinkedList，有三个值：`org.springframework.boot.autoconfigure.condition.OnBeanCondition`、`org.springframework.boot.autoconfigure.condition.OnClassCondition`、
      `org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition`,终于看到了我们熟悉的地方，这三个不就是条件注解的实现类嘛！通过源码搜索发现又有不一样，他没有直接继承`SpringBootCondition`类或实现
      `Condition`接口，而是继承`FilteringSpringBootCondition`类，而`FilteringSpringBootCondition`类继承了`SpringBootCondition`类并实现了`AutoConfigurationImportFilter`接口的`match`方法
    * 后面调用`instantiateFactory()`通过反射创建了实例，最终返回`List<AutoConfigurationImportFilter>`,这样我们就清楚了for循环中`getAutoConfigurationImportFilters()`获取的三个filter到底是什么了
  >我们来回顾一下之前的条件注解，条件注解类开始通过实现`Condition`接口进行使用，后来在SpringBoot源码中发现条件注解类通过继承`SpringBootCondition`类，`SpringBootCondition`类实现了`Condition`接口的`matches`方法，并提供给一个抽象方法，
  > `getMatchOutcome`供子类实现，`matches`方法中调用了`getMatchOutcome`,还增加了记录评估日志的能力(扩展了子类功能)，这就是**模板方法模式**。
  >
  > `OnBeanCondition`、`OnClassCondition`、`OnWebApplicationCondition`继承了`FilteringSpringBootCondition`类，`FilteringSpringBootCondition`类也提供了一个抽象方法`getOutcomes`，并在实现了`AutoConfigurationImportFilter`
  > 的`match`方法中调用了`getOutcomes`,这里也打印和记录了评估日志，并返回布尔数组，熟悉的模板方法模式
    *  回顾一下`ConditionOutcome`对象有两个字段，match布尔值记录是否满足条件，message记录评估日志，`FilteringSpringBootCondition`的`match`方法源码如下：
  ```java
      public boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
          ConditionEvaluationReport report = ConditionEvaluationReport.find(this.beanFactory);
          ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses, autoConfigurationMetadata);
          boolean[] match = new boolean[outcomes.length];
          for (int i = 0; i < outcomes.length; i++) {
              match[i] = (outcomes[i] == null || outcomes[i].isMatch());
              if (!match[i] && outcomes[i] != null) {
                  logOutcome(autoConfigurationClasses[i], outcomes[i]);
                  if (report != null) {
                      report.recordConditionEvaluation(autoConfigurationClasses[i], this, outcomes[i]);
                  }
              }
          }
          return match;
      }
  ```
  代码逻辑比较简单，入参autoConfigurationClasses即是最外层`filter.match()`传入，为SpringBoot默认的+META-INF/spring.factories中的自动配置类，  
  调用`getOutcomes`获取`ConditionOutcome`数组，返回所有的match值，`getOutcomes`由子类实现，这里只对其中一个进行分析，`OnClassCondition`顾名思义某些类存在时才会创建对象，源码如下:
  ```java
      protected final ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
              AutoConfigurationMetadata autoConfigurationMetadata) {
          // 如果有多个处理器可用，则拆分工作并在后台线程中执行一半。使用一个额外的线程似乎可以提供最佳的性能。线程越多，情况就越糟
          if (Runtime.getRuntime().availableProcessors() > 1) {
              return resolveOutcomesThreaded(autoConfigurationClasses, autoConfigurationMetadata);
          }
          else {
              OutcomesResolver outcomesResolver = new StandardOutcomesResolver(autoConfigurationClasses, 0,
                      autoConfigurationClasses.length, autoConfigurationMetadata, getBeanClassLoader());
              return outcomesResolver.resolveOutcomes();
          }
      }
  ```
    * 该源码逐条分析的话，又会是一大段，直接说结论会从一个Properties中查询某些值，比如SpringBoot默认加载自动配置`org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration`,这是
      configurations中的一个值，即`OnClassCondition`中查询个Properties的key为`org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration.OnClassCondition`,如果不存在则返回null，
      如果存在则返回字符串`org.springframework.amqp.rabbit.core.RabbitTemplate,com.rabbitmq.client.Channel`,两个类路径用逗号分割，如果你要问我值是怎么来的，细心的同学会发现SpringBoot源码`RabbitAutoConfiguration`类上有注解`@ConditionalOnClass({ RabbitTemplate.class, Channel.class })`,
      再结合代码上下文即可猜出。
    * 与此同时，我们也能推断出什么时候会返回null，该类没有`OnWebApplicationCondition`对应的注解，即返回null，这与`FilteringSpringBootCondition`的`match`方法中的`match[i] = (outcomes[i] == null || outcomes[i].isMatch());`一致，
      ，为空或match为true都会返回true
    * 拿到具体的值后以逗号分隔符拆分开，并去搜索类是否存在，不存在时评估日志message会有这样的记录：`@ConditionalOnClass did not find required class 'com.rabbitmq.client.Channel` `@ConditionalOnClass did not find required class 'org.springframework.amqp.rabbit.core.RabbitTemplate`,
      如果你要问什么时候不存在，大概是maven没有导入相关依赖时，这时我们对SpringBoot为什么导入依赖，配置yaml就能直接`@Autowire`使用有了更深的了解，这就是自动配置的原理

    4. 终于分析完了`filter`方法的过滤逻辑，``StringUtils.toStringArray(autoConfigurationEntry.getConfigurations())`直接返回了最终的configurations的字符串集合
* **总结一下，本节分析了从`@SpringBootApplication`到`@EnableAutoConfiguration`到`@Import(AutoConfigurationImportSelector.class)`，最终学习了类`AutoConfigurationImportSelector`的`selectImports`
  方法,该方法获取SpringBoot内置自动配置类+META-INF/spring.factories中需要自动配置的类，通过第一次`exclude`和`excludeName`过滤，再通过第二次filter过滤，得到了最终需要自动配置的类路径**
>**新的问题**：  
> SrpingBoot在启动时是如何执行到`selectImports`的呢？
>
> META-INF/spring.factories中需要自动配置的类通过读取文件获取，SpringBoot内置自动配置类的类路径是如何获取的呢？(上文没有对这个细节具体分析)
>
> 第二次filter过滤如何执行到条件注解的实现方法的？(我们可以推测，通过反射+类路径，可以获取类上的注解和注解的值，该值是一个class，即条件注解实现类，通过反射创建实例，通过执行实例方法返回的布尔值判断是否需要创建原实例)

***
### 自动配置测试
本节对上一节的内容进行测试
* 创建`B`、`C`、`D`、`E`四个类,形如下面的代码：
```java
package com.cx.model;

public class E {
}
```
* 在`resources`目录下创建`META-INF/spring.factories`,在`spring.factories`文件中添加如下内容：
```markdown
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.cx.model.B,\
com.cx.model.C,\
com.cx.model.D,\
com.cx.model.E
```
路径与自己的保持一致
* 创建测试代码如下：
```java
@RestController
public class TestController {

    @Autowired(required = false)
    private B b;
    @Autowired(required = false)
    private C c;
    @Autowired(required = false)
    private D d;
    @Autowired(required = false)
    private E e;

    @RequestMapping("test2")
    public void test2() {
        System.out.println("b的地址为："+b);
        System.out.println("c的地址为："+c);
        System.out.println("d的地址为："+d);
        System.out.println("e的地址为："+e);
    }
    
}
```
启动SpringBoot项目访问接口，控制台打印如下：
```shell
b的地址为：com.cx.model.B@6e3eb0cd
c的地址为：com.cx.model.C@421def93
d的地址为：com.cx.model.D@2b2954e1
e的地址为：com.cx.model.E@751ae8a4
```
与期望一致
* 设置`EnableAutoConfiguration`注解的`exclude`和`excludeName`,发现启动类只使用了`SpringBootApplication`注解，`@EnableAutoConfiguration`注解是`@SpringBootApplication`注解的元注解(注解的注解称为元注解)，
  `@SpringBootApplication`源码如下：
```java
//......
public @interface SpringBootApplication {
    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    Class<?>[] exclude() default {};

    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    String[] excludeName() default {};
    //......
```
`@AliasFor`用于定义注解属性别名，此时相当于`@SpringBootApplication`的`exclude`和`excludeName`分别是`@EnableAutoConfiguration`的`exclude`和`excludeName`的别名，因此设置
`@SpringBootApplication`的属性即可，修改代码如下：
```java
@SpringBootApplication(excludeName = "com.cx.model.D",exclude = com.cx.model.E.class)
public class TestApplication {

    public static void main(String[] args) {
        SpringApplication.run(TestApplication.class, args);
    }

}
```
我们通过两种方式分别排除了类`D`和类`E`的自动配置，此时重启SpringBoot项目打印类`D`和类`E`的实例地址应该为空，控制台打印如下：
```shell
b的地址为：com.cx.model.B@77f905e3
c的地址为：com.cx.model.C@777d191f
d的地址为：null
e的地址为：null
```
与期望一致,这种方式是通过**第一次过滤**实现的
* 我们能不能通过第二次过滤实现呢？答案是可以的。  
  回忆一下`filter`函数,我们获取了三个`AutoConfigurationImportFilter`,必须全部为true才不会过滤，它们的实际类型分别是`OnBeanCondition`、`OnClassCondition`、`OnWebApplicationCondition`,
  它们配置在SpringBoot源码`spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/spring.factories`下，内容如下：
```markdown
# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition
```
我们也能通过扩展`AutoConfigurationImportFilter`接口并配置实现过滤，在SpringBoot的demo项目添加类`MyFilter`,代码如下：
```java
package com.cx.filter;

import com.cx.model.C;
import org.springframework.boot.autoconfigure.AutoConfigurationImportFilter;
import org.springframework.boot.autoconfigure.AutoConfigurationMetadata;
import org.springframework.context.EnvironmentAware;
import org.springframework.core.env.Environment;

public class MyFilter implements AutoConfigurationImportFilter, EnvironmentAware {

    private Environment env;

    @Override
    public boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
        boolean[] matches = new boolean[autoConfigurationClasses.length];
        for (int i = 0; i < autoConfigurationClasses.length; i++) {
            matches[i] = true;
            System.out.println(i);
            if (C.class.getName().equals(autoConfigurationClasses[i])) {
                matches[i] = env.getProperty("c-auto-configuration.enabled",Boolean.class,true);
            }
        }
        return matches;
    }

    @Override
    public void setEnvironment(Environment environment) {
        env = environment;
    }
}
```
这段代码会获取`resources`下`application.yml`中`c-auto-configuration.enabled`的值，只有显示指定false时才不会自动注入  
还需要添加一些配置
* `resources/META-INF/spring.factories`修改后如下所示：
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.cx.model.B,\
com.cx.model.C,\
com.cx.model.D,\
com.cx.model.E

org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
com.cx.filter.MyFilter
```
* `resources/application.yml`增加如下配置
```yaml
c-auto-configuration:
  enabled: false
```
启动demo程序，访问接口控制台打印如下：
```shell
b的地址为：com.cx.model.B@18a25bbd
c的地址为：null
d的地址为：null
e的地址为：null
```
符合我们的期望，该方式是通过第二次过滤实现的
* 倘若现在有新的需求，只有当有类`C`的实例时才自动配置类`B`，可以使用注解`@ConditionalOnBean`，修改类`B`代码如下:
```java
package com.cx.model;

import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnBean(C.class)
public class B {
}
```
该注解只有使用在`@Configuration`修饰的类和`@bean`修饰的方法上才生效，启动demo程序控制台打印如下：
```shell
b的地址为：null
c的地址为：null
d的地址为：null
e的地址为：null
```
上一步我们已经过滤了类`C`的自动配置，因此类`B`不会创建实例，符合我们的期望
***

