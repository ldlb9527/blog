+++

author = "旅店老板"
title = "100行代码实现一个极简版的feign"
date = "2023-06-13"
description = "运用SpringBoot的原理简单实现Feign"
tags = [
"SpringBoot",
]
categories = [
"java",
"SpringBoot"
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "feign.png"
mermaid = true

+++
## 开始之前
* 创建一个SpringBoot的基础项目，用于代码测试。
* 熟悉动态代理的原理或已阅读 [java中的代理模式及其源码分析](https://ldlb.site/p/java%E4%B8%AD%E7%9A%84%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F%E5%8F%8A%E5%85%B6%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)  
## @EnableFeignClients和@FeignClient
熟悉SpringBoot的小伙伴对`@EnableFeignClients`和`@FeignClient`这两个注解不会陌生，前者使用在启动类上标志开启Feign,后者使用在接口上，最终实现rpc调用。  

feign就是基于SpringBoot的rpc框架。接下来具体说明这两个注解的基础源码和作用
***
### @EnableFeignClients
其源码如下：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({FeignClientsRegistrar.class})
public @interface EnableFeignClients {
    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

    Class<?>[] defaultConfiguration() default {};

    Class<?>[] clients() default {};
}
```
* 当我们配置了`basePackages`属性时，会扫描该数组中路径的`@FeignClient`,创建发送http的代理对象，不配置该属性默认扫描注解的类所在的包路径下，通常即启动类所在路径。  
* 因此我们配置了`basePackages`属性可以降低扫描范围，减少项目启动时间  
* `@EnableFeignClients`注解上有一个核心注解`@Import({FeignClientsRegistrar.class})`,导入了一个`FeignClientsRegistrar`类，该类实现了`ImportBeanDefinitionRegistrar`接口
  (不了解该接口也没关系，接着往下看)  
***
### @FeignClient
其源码如下：
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FeignClient {
    @AliasFor("name")
    String value() default "";

    @AliasFor("value")
    String name() default "";

    String url() default "";
    //......
}
```
* 省略了大部分属性，主要关注`name`和`url`,`value`属性为`name`的别名。
* 通常我们设置`name`未指定`url`时，会根据`name`的值去注册中心获取一个url(通常是eureka,feign作为一个独立的rpc框架，未与eureka强绑定),当我们设置了`url`后，feign就会使用我们设置的url。
***
## 自定义rpc注解
接下来我们实现上述两个注解的功能。 
### @Client
自定义注解，用于替换`@FeignClient`注解，其代码如下：
```java
public @interface Client {
    String url() default "";
    String name() default "";
}
```
* 只有两个属性，我们重点使用url属性
***
### @EnableClients
自定义注解，用于替换`@EnableFeignClients`注解，表示开启扫描`@Client`。开发源码如下：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({ClientsRegistrar.class})
public @interface EnableClients {

    String[] basePackages() default {};

}
```
* 我们自定义的这个注解与`@EnableFeignClients`非常相似，也导入了一个`ClientsRegistrar.class`,后续我们也会为该类实现`ImportBeanDefinitionRegistrar`接口
* 该注解只有一个`basePackages`属性(我们只是实现一个简化版的feign)  

`ClientsRegistrar.class`的部分代码实现如下：
```java
public class ClientsRegistrar implements ResourceLoaderAware, EnvironmentAware {

    private ResourceLoader resourceLoader;
    private Environment environment;
    
        @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
    
}
```
* 首先创建两个私有成员变量`resourceLoader`和`environment`，并实现`ResourceLoaderAware`和`EnvironmentAware`接口未它们赋值，后面会用到  

接着我们实现`ImportBeanDefinitionRegistrar`接口，实现对应方法的代码如下：
```java
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        try {

            Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableClients.class.getName());
            //获取要扫描的包路径
            String[] basePackages = (String[])attrs.get("basePackages");
            Set<String> basePackageSet = new HashSet<>(Arrays.asList(basePackages));
            //如果未配置basePackages，默认启动类所在目录下
            if (basePackageSet.isEmpty()) basePackageSet.add(ClassUtils.getPackageName(metadata.getClassName()));

            //扫描包路径下的@Client
            Set<BeanDefinition> candidateComponents = findClient(basePackageSet);
            //注册扫描到的@Client
            register(registry,candidateComponents);

        }catch (Exception e) {
            e.printStackTrace();
        }
    }
```
* 首先获取`@EnableClients`注解的`basePackages`属性值，是一个字符串数组。
* Set用于去重，如果为空，则添加一个注解所在类的路径(通常是启动类)
* `findClient`方法是查找路径下所有的`@Client`注解
* `register`方法注册每个`@Client`注解的类为一个`BeanDefinition`实例，我们会获取注解的`name`和`url`以及所在类的`className`,用`ClientFactoryBean.class`包装
* 完整的`ClientsRegistrar`类代码如下，可以直接复制到自己的项目中测试：
```java
import org.springframework.beans.factory.annotation.AnnotatedBeanDefinition;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanDefinitionHolder;
import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.BeanDefinitionReaderUtils;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.context.EnvironmentAware;
import org.springframework.context.ResourceLoaderAware;
import org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider;
import org.springframework.context.annotation.ImportBeanDefinitionRegistrar;
import org.springframework.core.env.Environment;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.core.type.filter.AnnotationTypeFilter;
import org.springframework.util.ClassUtils;
import org.springframework.util.MultiValueMap;
import java.util.Arrays;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

public class ClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {

    private ResourceLoader resourceLoader;
    private Environment environment;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        try {

            Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableClients.class.getName());
            //获取要扫描的包路径
            String[] basePackages = (String[])attrs.get("basePackages");
            Set<String> basePackageSet = new HashSet<>(Arrays.asList(basePackages));
            //如果未配置basePackages，默认启动类所在目录下
            if (basePackageSet.isEmpty()) basePackageSet.add(ClassUtils.getPackageName(metadata.getClassName()));

            //扫描包路径下的@Client
            Set<BeanDefinition> candidateComponents = findClient(basePackageSet);
            //注册扫描到的@Client
            register(registry,candidateComponents);

        }catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void register(BeanDefinitionRegistry registry,Set<BeanDefinition> candidateComponents) {
        for (BeanDefinition candidateComponent : candidateComponents) {
            AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition)candidateComponent;
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
            String className = annotationMetadata.getClassName();
            MultiValueMap<String, Object> attributes = annotationMetadata.getAllAnnotationAttributes(Client.class.getCanonicalName());
            String url = (String) attributes.getFirst("url");
            String name = (String) attributes.getFirst("name");

            BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(ClientFactoryBean.class);
            definition.addPropertyValue("url",url);
            definition.addPropertyValue("name",name);
            definition.addPropertyValue("type",className);
            definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
            AbstractBeanDefinition bd = definition.getBeanDefinition();
            bd.setPrimary(true);
            BeanDefinitionHolder holder = new BeanDefinitionHolder(bd, className, null);
            BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
        }
    }

    private Set<BeanDefinition> findClient(Set<String> basePackageSet) {
        Set<BeanDefinition> candidates = new HashSet<>();

        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        for (String basePackage : basePackageSet) {
            Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents(basePackage);
            candidates.addAll(candidateComponents);
        }

        return candidates;
    }

    private ClassPathScanningCandidateComponentProvider getScanner() {
        ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false, this.environment) {
            protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
                boolean isCandidate = false;
                if (beanDefinition.getMetadata().isIndependent() && !beanDefinition.getMetadata().isAnnotation()) {
                    isCandidate = true;
                }

                return isCandidate;
            }
        };
        scanner.setResourceLoader(this.resourceLoader);
        AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(Client.class);
        scanner.addIncludeFilter(annotationTypeFilter);
        return scanner;
    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```

* `ClientFactoryBean.class`的核心在于实现`FactoryBean`接口，其代码如下:
```java
import cn.hutool.http.HttpUtil;
import org.springframework.beans.factory.FactoryBean;
import org.springframework.core.annotation.AnnotatedElementUtils;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import java.lang.annotation.Annotation;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;


public class ClientFactoryBean implements FactoryBean<Object> {

    private Class<?> type;
    private String url;
    private String name;

    public Class<?> getType() {
        return type;
    }

    public void setType(Class<?> type) {
        this.type = type;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public Object getObject() throws Exception {

        Object proxy = Proxy.newProxyInstance(ClientFactoryBean.class.getClassLoader(), new Class[]{type}, (o, method, objects) -> {
            RequestMapping methodMapping = AnnotatedElementUtils.findMergedAnnotation(method, RequestMapping.class);
            RequestMethod[] methods = methodMapping.method();
            if (methods.length == 0) {
                methods = new RequestMethod[] { RequestMethod.GET };
            }
            String name = methods[0].name().toLowerCase();
            String path = methodMapping.path()[0];

            Map<String, Object> queryMap = new HashMap<>();
            Annotation[][] parameterAnnotations = method.getParameterAnnotations();
            for (int i = 0; i < parameterAnnotations.length; i++) {
                RequestParam requestParam = RequestParam.class.cast(parameterAnnotations[i][0]);
                queryMap.put(requestParam.value(),objects[i]);
            }
            
            return request(name, url, path, queryMap);
        });
        return proxy;
    }

    private String request(String name, String url, String path, Map<String, Object> queryMap) {
        if ("get".equals(name)) {
            return HttpUtil.get(url + path, queryMap);
        }else if ("post".equals(name)) {
            return HttpUtil.post(url + path, queryMap);
        }else {
            throw new RuntimeException("Not Supported...");
        }
    }

    @Override
    public Class<?> getObjectType() {
        return this.type;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```
* 代码比较简单，核心在于`getObject`方法返回一个代理对象 
* 该类的属性为最开始注册`BeanDefinition`实例设置的值，创建对象时会通过`get`和`set`方法赋值
* 简单说明一下`FactoryBean`接口的作用，`@Autowired`执行对应的逻辑时会找到对应的`BeanDefinition`实例，  

我们在最开始注册的className为`@Client`所在类路径，值为`ClientFactoryBean`的`BeanDefinition`实例，会判断是否是`FactoryBean`的实现类，  

如果是则执行它的`getObject`方法获取实例(一般像`@Component`注解的类就不是`FactoryBean`的实现类，因此直接实例化`BeanDefinition`实例包装的类)，    

这也就是这里使用`@Autowired`没有直接实例化`ClientFactoryBean`的原因，而是实例化后调用`getObject`方法
* 我们的jdk动态代理实现中，简单地处理了`@RequestMapping`和`@RequestParam`注解，其他暂不处理。
* 最终通过`request`方法发送了一个http请求，网上随便找一个http工具类，实际这里我们使用的是`hutool`的HttpUtils，对应依赖为：
```markdown
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>5.7.19</version>
        </dependency>
```
* 实际上feign框架中解析`@RequestMapping`、`@RequestBody`等等注解都是使用springmvc的方法
***
## 测试
上面我们已经完成了所有代码开发，下面进行测试。
* 启动类上添加`@EnableClients`注解，代码如下：
```java
@SpringBootApplication
@EnableClients
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class,args);
    }
}
```
* 创建对应远程服务接口并添加`@Client`注解,代码如下：
```java
@Client(url = "127.0.0.1:8080", name = "notify")
public interface NotifyService {

    @RequestMapping("/hello")
    void sayHello();

    @RequestMapping("/hi")
    String sayHi(@RequestParam("name") String name, @RequestParam("age") Integer age);
}
```
* 创建测试`controller`，代码如下：
```java
@RestController
public class TestController {

    @Autowired
    private NotifyService notifyService;

    @RequestMapping("/test")
    public void test() {
        notifyService.sayHello();
        String result = notifyService.sayHi("张三", 15);
        System.out.println("result: "+result);
    }

    @RequestMapping("/hello")
    public void hello() {
        System.out.println("hello hello hello");
    }

    @RequestMapping("/hi")
    public String hi(@RequestParam("name") String name, @RequestParam("age") Integer age) {
        System.out.println("hi hi hi  "+name+"  "+age);
        return "请求成功";
    }
}
```
共创建了三个接口,`/test`接口模拟客服端，`/hi`、`/hello`接口模拟服务器端(虽然它们都在同一项目中)，启动项目访问`/test`接口控制台打印如下：
```markdown
hello hello hello
hi hi hi  张三  15
result: 请求成功
```
检查打印信息，符合我们的预期，通过自定义注解实现了feign的基本功能。
## 小结
* 如果你对`@Import`注解、`ImportBeanDefinitionRegistrar`接口、`FactoryBean`接口、`@Autowired`的原理不理解的话，可以结合SpringBoot的源码和 [SpringBoot原理(一):自动配置](https://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E4%B8%80%E8%87%AA%E5%8A%A8%E9%85%8D%E7%BD%AE/) 、 
[SpringBoot原理(二):Starter](https://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E4%BA%8Cstarter/) 、
[SpringBoot原理(三):启动流程分析](https://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E4%B8%89%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/) 、
[SpringBoot原理(四):常用注解分析](https://ldlb.site/p/springboot%E5%8E%9F%E7%90%86%E5%9B%9B%E5%B8%B8%E7%94%A8%E6%B3%A8%E8%A7%A3%E5%88%86%E6%9E%90/) 进行调试学习。  

* 是否能用cglib动态代理实现呢？
* 是否了解@RequestParam、@RequestBody、@RequestHeader等注解在SpringBoot中如何解析，启动时路由如何注册？（这与springmvc密切相关）
* 与feign类似的注解还有mytatis的`@MapperScan`和`@Mapper`注解，能否理解它的实现呢？
* 熟悉了动态代理和一些SpringBoot原理，能否实现一个切点表达式注解呢？