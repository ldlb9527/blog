+++

author = "旅店老板"
title = "SpringBoot原理(四):@Autowired"
date = "2023-06-06"
description = "SpringBoot原理(四):@Autowired"
tags = [
	"tcp",
]
categories = [
    "linux",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "SpringBoot.png"
mermaid = true

+++
## 开始之前
* 了解SpringBoot启动流程的源码或阅读[https://github.com/spring-projects/spring-boot/tree/v2.2.13.RELEASE](https://github.com/spring-projects/spring-boot/tree/v2.2.13.RELEASE)
* 准备一个SpringBoot测试项目,用于用例测试
* SpringBoot 2.2.13RELEASE的源码调试环境(该文章所使用的源码都来自该版本)  
  下载地址为：[https://github.com/spring-projects/spring-boot/tree/v2.2.13.RELEASE](https://github.com/spring-projects/spring-boot/tree/v2.2.13.RELEASE)
## SpringBoot启动流程图
复习一下SpringBoot启动流程图如下所示：
![SpringBoot启动流程](SpringBoot启动流程.drawio.png "SpringBoot启动流程")
* **SpringApplication.run()**:启动流程的入口
* **StopWatch.start()**:
* **SpringApplicationRunListeners.starting()**:
* **prepareEnviroment()**:
* **printBanner()**:
* **createApplicationContext()**:
* **prepareContext()**:
* **refreshContext()**:
* **afterRefresh()**:
* **StopWatch.stop()**:
* **SpringApplicationRunListeners.started()**
* **callRunners()**:
* **SpringApplicationRunListeners.running()**:
* **return context**
***
***
## @Component和@Autowired的使用
在测试项目中创建一个`service`和`controller`
* `service`代码如下：
```java
@Service
public class TestService  {
}
```
`@Service`是@Component的派生注解,我们知道SpringBoot读取注解是采用递归的方式,因此这里是能获取到`@Component`,最终存储到IOC容器中
* `controller`代码如下：
```java
@RestController
public class TestController {

    @Autowired(required = false)
    private TestService testService;
    
    @RequestMapping("/test")
    public void test2() {
        System.out.println("testService的地址为："+testService);
    }
    
}
```
代码较简单,提供一个`/Test`接打印`testService`的地址,启动程序访问该接口控制台打印如下：
```markdown
testService的地址为：com.cx.controller.TestService@5e5aafc6
```
这个比较好理解,使用`@Autowired`注解获取到了对应实例,正常打印了地址,这种也是我们使用SpringBoot最基本的方式。

我们修改`controller`，修改后的代码如下：
```java
@RestController
public class TestController implements BeanDefinitionRegistryPostProcessor  {

    @Autowired(required = false)
    private TestService testService;

    @RequestMapping("/test")
    public void test2() {
        System.out.println("testService的地址为："+testService);
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
    }
}
```
仅仅让`TestController`实现了`BeanDefinitionRegistryPostProcessor`接口，启动程序访问接口控制台打印如下：
```markdown
testService的地址为：null
```
没有获取到该对象,地址为null。

**如果你不知道获取不到`service`得原因,说明你不知道注解`@Autowired`和`@Component`如何产生作用，说明你不了解bean的生命周期。**
## 原理分析
在研究上述问题原因之前,我们先了解两个常见的概念：
* **BeanDefinition**：用一个BeanDefinition对象来描述bean,属性包含:全限定类名、Bean行为配置元素等
* **PostProcessor**:源码中常见这个单词的后缀，与后置处理器相关
### @Component
在流程图的`createApplicationContext()`中,`context`的构造方法最后调用`AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);`,
会注册`ConfigurationClassPostProcessor`的BeanDefinition(`context`是反射创建，需要你自己找到对应的构造方法)
>注册key为`org.springframework.context.annotation.internalConfigurationAnnotationProcessor`字符串,value为对应的BeanDefinition
>
>注册到哪里呢？
>注册到`DefaultListableBeanFactory`的`beanDefinitionMap`,该属性类型为`Map<String, BeanDefinition> `
>
>context与beanFactory的关系？  
>context拥有一个属性`beanFactory`,里面有非常多集合，顾名思义主要存储对象实例

`ConfigurationClassPostProcessor`实现了`BeanDefinitionRegistryPostProcessor`的`postProcessBeanDefinitionRegistry`方法，该方法获取获取启动类上的`ComponentScan`注解(调试发现启动类也有@Component注解)，最终调用`componentScanParser.parse()`根据注解配置的扫描路径(没有值则根据启动类路径设置
比如启动类为`com.cx.TestApplication`，扫描的`basePackage`为`com.cx`)，过滤参数等得到最终的`set`
集合,集合元素为`BeanDefinitionHolder`类型，该类有两个重要属性:`beanName`和`beanDefinition`。
>@Configuration也是@Component的派生注解


`beanName`记录bean的名称，比如`testController`,`beanDefinition`实际类型是一个`ScannedGenericBeanDefinition`,记录对应的class、是否单例(默认就是单例)、依赖的class、
initMethodName、destroyMethodName等等

接着扫描集合中的类有无`@Import`注解,会导入@Import的class（**注意这是一种递归扫描**）,还记得`@Import({AutoConfigurationImportSelector.class})`这个注解吗？它会加载
META-INF/spring.factories定义的bean(详情请参考文章)

还会获取`@Component`下的beanMethod（即带有`@bean`注解的方法),最终也会加载对应的bean，以前背八股文的时候说`@bean`必须结合`@Configuration`使用，
显然这是错误的，结合`@Component`也是一样的

**小结**：**我们调试阅读`ConfigurationClassPostProcessor`这个后置处理器的方法，从获取启动类的`ComponentScan`注解，到扫描到所有的`@Component`注解，并不只是如此，
还会扫描`@Import`注解加载对应的类，还会扫描`@bean`注解加载对应的类，该方法最终在哪被调用呢？流程图中的`refreshContext()`中的`invokeBeanFactoryPostProcessors`方法。**
***
### @Autowired
* 在`ConfigurationClassPostProcessor`同样的位置(间隔几行代码)还会注册一个`AutowiredAnnotationBeanPostProcessor`的BeanDefinition,`AutowiredAnnotationBeanPostProcessor`这个后置处理器
  与注解@Autowired息息相关。

* 在执行完`refreshContext()`中的`invokeBeanFactoryPostProcessors`方法后带有`@Component`注解的bean已经被包装成成`BeanDefinition`存储到`beanFactory.beanDefinitionMap`,还未真正实例化，
  但我们测试代码中`testController`作为一个后置处理器已经实例化

* 但`AutowiredAnnotationBeanPostProcessor`这个后置处理器还未执行，这是为什么呢？
  原来该后置处理器实现的是`MergedBeanDefinitionPostProcessor`接口，`ConfigurationClassPostProcessor`后置处理器实现的是`BeanDefinitionRegistryPostProcessor`接口，`invokeBeanFactoryPostProcessors`方法
  中专门获取`BeanDefinitionRegistryPostProcessor.class`的类型。

* 执行完`AutowiredAnnotationBeanPostProcessor`对应方法后后还会获取`testController`（测试代码中也实现了这个后置处理器接口）执行对应方法。

* `AutowiredAnnotationBeanPostProcessor`这个后置处理器什么时候被调用呢？  
  `invokeBeanFactoryPostProcessors`的下一行代码`registerBeanPostProcessors`会实例化`AutowiredAnnotationBeanPostProcessor`并注册到beanFactory，这两行代码都与后置处理器相关,
  但它们有一些命名上的区别`BeanFactoryPostProcessor`和`BeanPostProcessor`,这两个接口都是Spring对外暴露的扩展点。
    * `BeanFactoryPostProcessor`：是在实例化之前被调用
    * `BeanPostProcessor`：是在实例化过程中被调用

这两个是非常重要的扩展点，比如SpringBoot中的`@EnableScheduling`注解，用于定时任务，底层就是扩展的`BeanPostProcessor`接口
* 回到上面的问题调用流程图的`createApplicationContext()`中的`finishBeanFactoryInitialization`方法(在`invokeBeanFactoryPostProcessors`后面)将`BeanDefinition`实例化
* 实例化方法为`beanFactory.getBean()`，内层调用doGetBean,实例化bean时根据依赖项优先注入`testService`,再执行`@AutoWired`对应的后置处理器
***
## 小结
经过探索,我们能清晰地认识到产生上面的问题原因,在`testController`实现`BeanDefinitionRegistryPostProcessor`接口后,

在第一次调用后置器方法时,`testController`作为一个`BeanFactoryPostProcessor`已经被实例化,此时与`@Autowired`没有任何逻辑关系,  `ConfigurationClassPostProcessor`会将
`@Component`注解的类包装成`BeanDefinition`实例

后面在将`BeanDefinition`实例创建为真正的bean时,每个bean都会调用`@Autowired`关联的后置处理器(如果类中有注解)，每个bean在创建时会判断单例是否存在,
`testController`已存在就不会再创建bean，也就不会执行`@Autowired`相关的逻辑了
***
## 其他
SpringBoot中与bean生命周期有关的注解还有：`@PostConstrct`和`@PreDestroy`,它们对应的后置处理器为`CommonAnnotationBeanPostProcessor`。

`CommonAnnotationBeanPostProcessor`也是在与`AutowiredAnnotationBeanPostProcessor`同一时机被注册,实际上本文的三个后置处理器都是同一时机注册的。

解决问题的同时也会出现新的问题：**SpringBoot的后置处理器体系**、**bean的生命周期**,还需要进一步学习。













