+++

author = "旅店老板"
title = "java中的代理模式及其源码分析"
date = "2023-05-25"
description = "关于jdk、cglib动态代理例子及其源码分析"
tags = [
	"SpringBoot",
]
categories = [
"java",
"SpringBoot"
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "java.jpg"
mermaid = true

+++
# 代理模式
通常我们直接调用对象A的方法实现业务,代理是指我们不再直接调用原对象A的方法，而是创建一个代理对象，
在代理对象的方法中调用A的方法,好处是我们能对A的功能进行增强。

代理模式分为**静态代理**和**动态代理**。
## 静态代理
静态代理是指我们提前编写代理类,然后再编译运行。下面用一个java代码例子来进行说明：
```java
public interface Func {
    void apply();
}
```
创建一个接口`Func`,提供`apply`方法供实现，目标类实现如下：
```java
public class FuncImpl implements Func {
    @Override
    public void apply() {
        System.out.println("执行业务");
    }
}
```
这就是直接调用实现业务,我们来看一下静态代理如何实现：
```java
public class FuncProxy implements Func {

    private Func func;
    @Override
    public void apply() {
        System.out.println("前置增强");
        func.apply();
        System.out.println("后置增强");
    }

    public FuncProxy(Func func) {
        this.func = func;
    }
}
```
创建了一个代理类`FuncProxy`,也实现了相同接口,使用方式如下：
```java
public class ProxyTest {
    public static void main(String[] args) {
        FuncImpl func = new FuncImpl();
        FuncProxy funcProxy = new FuncProxy(func);
        funcProxy.apply();
    }
}
```
控制台打印如下：
```markdown
前置增强
执行业务
后置增强
```
静态代理很好理解,缺点也十分明显,每一个目标类我们都需要去创建对应的代理类。
***
## 动态代理
### jdk动态代理
动态代理就是为了解决上述静态代理的问题,让代理类在运行过程中产生。

创建代理方`DynamicProxy`,实现如下：
```java
public class DynamicProxy implements InvocationHandler {

    private Object object;

    public DynamicProxy(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("前置增强");
        method.invoke(object,args);
        System.out.println("后置增强");
        return null;
    }
}
```
代码与之前有一定类似,成员变量变为Object类型,而不是固定接口类型,代理方也不实现目标类的接口，转而实现`InvocationHandler`
接口的`invoke`方法。

使用方式如下所示：
```java
public class ProxyTest {
    public static void main(String[] args) {
        FuncImpl func = new FuncImpl();
        DynamicProxy dynamicProxy = new DynamicProxy(func);

        Func o = (Func)Proxy.newProxyInstance(func.getClass().getClassLoader(), func.getClass().getInterfaces(), dynamicProxy);
        o.apply();
    }
}
```
控制台打印如下：
```markdown
前置增强
执行业务
后置增强
```
使用比较类似,静态代理我们直接创建了代理类`FuncImpl`实例,动态代理虽然我们也创建了`FuncImpl`实例，但我们最终使用的代理类实例，是在运行时创建的

核心在于调用`Proxy.newProxyInstance`方法传入一个类加载器，一个接口class集合,一个`InvocationHandler`实现，即`DynamicProxy`实例

返回了一个对应接口的实例，该实例已与传入的实例一样，调用`apply`方法,输出的却是增强后的逻辑，就像是调用了`invoke`方法。

我们来分析下`Proxy.newProxyInstance`方法的源码,省略了一些判断逻辑：
```java
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        //......
        final Class<?>[] intfs = interfaces.clone();

        Class<?> cl = getProxyClass0(loader, intfs);
        
        try {
            //......
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```
`Class<?> cl = getProxyClass0(loader, intfs);`它的作用是查找或生成指定的代理类,`getProxyClass0`的源码如下:
```java
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        return proxyClassCache.get(loader, interfaces);
    }
```
该代码的逻辑比较简单直接从`proxyClassCache`中获取缓存的代理类副本,若没有将通过`ProxyClassFactory`创建代理类,获取到代理类后,执行
逻辑`final Constructor<?> cons = cl.getConstructor(constructorParams);`获取对应的构造方法,代理类构造函数的参数`constructorParams`是一个class数组:
```markdown
private static final Class<?>[] constructorParams = { InvocationHandler.class };
```
`InvocationHandler`就是我们`DynamicProxy`实现的接口

>**这个代理类的构造方法是怎么来的呢**？
>
>回到上面从缓存`proxyClassCache`的`get`方法逻辑,缓存没有则最终调用`subKeyFactory.apply(key, parameter)`生成,
>这里的`key`与我们传入的类加载器相关，`parameter`就是我们传入的接口class数组，比如测试代码中传入的`parameter = { Func.class }`
>
>`subKeyFactory.apply(key, parameter)`根据规则生成一个类路径，为`com.sun.proxy.$Proxy0`，后面的0是一个原子自增的数字,创建一个代理类则递增，
>最终会调用一个native方法`defineClass0`生成class,这就是我们动态生成的class
>
>因此获取带有`constructorParams`参数的构造方法应该是在native方法中生成的
>
>这个在运行中生成的代理类class与我们传入`FuncImpl`实例有什么关系,**实现了相同的接口Func**
***
通过对`newProxyInstance`方法的源码两行逻辑：`Class<?> cl = getProxyClass0(loader, intfs);`、
`final Constructor<?> cons = cl.getConstructor(constructorParams);`,我们了解这个运行时生成的代理类的类路径为`com.sun.proxy.$Proxy0`(第一个代理类为0,后面依次递增),
了解带`constructorParams`参数的构造方法来源

`constructorParams`参数中有`InvocationHandler`接口的class，我们可以推测接下来的逻辑就是将`FuncImpl`实例的`apply`和`DynamicProxy`实例的`invoke`
合成到一起，构成代理类接口的具体实现，即我们最终调用的增强的`apply`方法

第三行核心逻辑`cons.newInstance(new Object[]{h});`,直接用反射通过有参构造方法创建实例，这里的h参数就是我们的`DynamicProxy`实例,
与第二行核心逻辑获取的有参构造一致，似乎与我们的推断有出入，它在动态生成类class时已经组装完成。   

通过反射创建类路径为`com.sun.proxy.$Proxy0`实现了Func接口的`apply`方法，有个成员变量为接口的数组，通过有参构造赋值，
`apply`方法的实现就是调用这个成员变量`InvocationHandler`接口实例的`invoke`方法。  

这种方式的动态代理是通过反射机制实现的，动态生成的代理类必须基于统一的接口，这就是**jdk动态代理**。
***
### cglib动态代理
总所周知，java中除了jdk动态代理外，还有cglib动态代理,让我们来分析一下它的原理。  

* `pom.xml`引入cglib依赖
```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>2.2.2</version>
</dependency>
```
* 创建类`Student`,代码如下：
```java
public class Student {

    public void sayHello() {
        System.out.println("Hello");
    }

    public void sayHi() {
        System.out.println("Hi");
    }
}
```
该类没有实现任何接口,只有两个方法：`sayHello`和`sayHi`
* 创建测试代码如下：
```java
public class ProxyTest {
    public static void main(String[] args) {
        Student student = new Student();
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Student.class);
        enhancer.setCallback(new MethodInterceptor() {
            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                System.out.println("前置增强");
                method.invoke(student,objects);
                System.out.println("后置增强");
                return null;
            }
        });

        Student o = (Student)enhancer.create();
        o.sayHello();
        o.sayHi();
    }
}
```
第一步仍然创建仍然创建业务对象`Student`实例，接着创建了一个`Enhancer`实例，设置`enhancer.superclass`值为对应的业务类的class，
接着为`enhancer.callbacks`设置了实现具体实现，该实现的`intercept`方法与`InvocationHandler`接口的`invoke`方法有一点类似，
最终调用`enhancer.create()`创建了一个业务类`Student`实例，并调用对应方法。  

控制台打印如下：
```markdown
前置增强
Hello
后置增强
前置增强
Hi
后置增强
```
可以感觉到cglib动态代理要比jdk动态代理功能要强大一点，在jdk动态代理中必须要基于接口进行增强，而cglib动态代理可以直接对业务类的方法进行增强。  

接下来我们对cglib动态代理的源码进行分析，以便我们更能区别和掌握这两种动态代理。 

`Enhancer`的数据结构如下，只保留了部分字段进行说明：
```java
public class Enhancer extends AbstractClassGenerator {
    private static final CallbackFilter ALL_ZERO = new CallbackFilter() {
        public int accept(Method method) {
            return 0;
        }
    };
    private static final Source SOURCE;

    private Class[] interfaces;
    private Callback[] callbacks;
    private Class superclass;
    
    public Enhancer() {
        super(SOURCE);
    }
}
```
根据类定义可知类`Enhancer`继承了类`AbstractClassGenerator`,我们在测试代码中创建`Enhancer`实例，会调用无参构造方法，无参构造方法中使用`super()`即调用父类的构造方法。  

`SOURCE`是一个静态成员变量，数据结构如下：
```java
    protected static class Source {
        String name;
        Map cache = new WeakHashMap();

        public Source(String name) {
            this.name = name;
        }
    }
```
两个成员变量:`name`和`cache`

让我们看一下创建`Enhancer`实例时父类构造方法做了什么，源码如下：
```java
    protected AbstractClassGenerator(AbstractClassGenerator.Source source) {
        this.strategy = DefaultGeneratorStrategy.INSTANCE;
        this.namingPolicy = DefaultNamingPolicy.INSTANCE;
        this.useCache = true;
        this.source = source;
    }
```
设置了四个字段，`source`就是我们上面描述的数据结构；`useCache`设置为true表示使用缓存；`namingPolicy`设置为`DefaultNamingPolicy.INSTANCE`,
顾名思义该字段是命名策略,我们来看一下具体实现：
```java
public class DefaultNamingPolicy implements NamingPolicy {
    public static final DefaultNamingPolicy INSTANCE = new DefaultNamingPolicy();

    public DefaultNamingPolicy() {
    }

    public String getClassName(String prefix, String source, Object key, Predicate names) {
        if (prefix == null) {
            prefix = "net.sf.cglib.empty.Object";
        } else if (prefix.startsWith("java")) {
            prefix = "$" + prefix;
        }

        String base = prefix + "$$" + source.substring(source.lastIndexOf(46) + 1) + this.getTag() + "$$" + Integer.toHexString(key.hashCode());
        String attempt = base;

        for(int var7 = 2; names.evaluate(attempt); attempt = base + "_" + var7++) {
        }

        return attempt;
    }
}
```
`INSTANCE`在这里用到了单例模式，其中有个`getClassName`方法用于生成类路径的，还记得jdk东莞太代理生成的`com.sun.proxy.$Proxy0`吗，这个方法的作用也类似。  

还有个`strategy`字段设置为`DefaultGeneratorStrategy.INSTANCE`,这个`INSTANCE`也是一个单例，它实现的方法请自行查看。  

我们分析完了`Enhancer`实例初始化的过程,来看一下`enhancer.setSuperclass(Student.class);`的逻辑，`setSuperclass`的源码为：
```java
    public void setSuperclass(Class superclass) {
        if (superclass != null && superclass.isInterface()) {
            this.setInterfaces(new Class[]{superclass});
        } else if (superclass != null && superclass.equals(class$java$lang$Object == null ? (class$java$lang$Object = class$("java.lang.Object")) : class$java$lang$Object)) {
            this.superclass = null;
        } else {
            this.superclass = superclass;
        }

    }
```
判断我们传入的class不为空且为接口的话,设置`enhancer.interfaces`的值，这个字段是否很熟悉呢？(回忆jdk动态代理中的这个字段)，如果不是接口的设置
`enhancer.superclass`的值就为我们传入的class,测试代码中传入的`Student.calss`。  

```java
        enhancer.setCallback(new MethodInterceptor() {
            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                System.out.println("前置增强");
                method.invoke(student,objects);
                System.out.println("后置增强");
                return null;
            }
        });
```
接着我们看上面这部分代码的具体实现，`setCallback`方法中我们传参为`MethodInterceptor`接口的实例(实现其`intercept`方法)，`setCallback`方法源码为：
```java
    public void setCallback(Callback callback) {
        this.setCallbacks(new Callback[]{callback});
    }

    public void setCallbacks(Callback[] callbacks) {
        if (callbacks != null && callbacks.length == 0) {
            throw new IllegalArgumentException("Array cannot be empty");
        } else {
            this.callbacks = callbacks;
        }
    }
```
该方法接受的参数是一个`Callback`接口，最终将传入的实现包装成一个`Callback`数组,设置到`enhancer.callbacks`,就跟我们方法名的意思一样。 

我们来看一下`Callback`接口的结构，源码如下：
```java
public interface Callback {
}
```
竟然是一个空接口，想必`MethodInterceptor`接口继承了`Callback`接口，`MethodInterceptor`接口的结构如下：
```java
public interface MethodInterceptor extends Callback {
    Object intercept(Object var1, Method var2, Object[] var3, MethodProxy var4) throws Throwable;
}
```

了解了以上代码，来到了cglib动态代理的核心逻辑`Student o = (Student)enhancer.create();`，即`create`方法，该方法在运行时动态生成了一个目标类的代理类，
`create`方法的源码如下:
```java
    public Object create() {
        this.classOnly = false;
        this.argumentTypes = null;
        return this.createHelper();
    }
```
设置了`enhancer.classOnly`为false,代表不仅仅是类，还有可能是接口？ 最终调用自身的`createHelper`方法，`createHelper`方法的源码如下：
```java
    private Object createHelper() {
        this.validate();
        if (this.superclass != null) {
            this.setNamePrefix(this.superclass.getName());
        } else if (this.interfaces != null) {
            this.setNamePrefix(this.interfaces[ReflectUtils.findPackageProtected(this.interfaces)].getName());
        }

        return super.create(KEY_FACTORY.newInstance(this.superclass != null ? this.superclass.getName() : null, ReflectUtils.getNames(this.interfaces), this.filter, this.callbackTypes, this.useFactory, this.interceptDuringConstruction, this.serialVersionUID));
    }
```
`this.validate();`校验逻辑还有一些赋值请自行阅读；接着if判断区分了类和接口,为两种情况设置不同的`enhancer.namePrefix`,我们测试代码中是类,此时设置的
值为`com.cx.proxy.Student`；最终调用了父类的`create`方法。  

`KEY_FACTORY.newInstance`方法生成了一个`Object`类型的key作为`create`方法的入参，`create`方法源码较长但逻辑简单,与jdk动态代理从缓存中获取代理类相似。
具体逻辑也是从缓存中获取，如果没有，回去加载前面生成的类路径，这里实际会发生一个异常`ClassNotFoundException`，最终则则调用反射中的native方法`newInstance`动态生成代理类

## 动态代理类查看
通过上一节我们了解并熟悉了动态代理类生成的过程，但我们不知道这个动态生成的类到底长啥样，接下来我们将尝试将这个类的class文件保存下来。

### cglib动态代理类  
只需要在`main`方法开头加上一句代码`System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, ".\\");`,第二个参数为路径，表示与src同级目录。  

结果会保存动态生成的类在src下生成两个目录`com`和`net`,动态代理类在`com`目录下，省略的类代码如下：
```java
public class Student$$EnhancerByCGLIB$$3373ac0e extends Student implements Factory {

    final void CGLIB$sayHello$0() {
        super.sayHello();
    }

    public final void sayHello() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$sayHello$0$Method, CGLIB$emptyArgs, CGLIB$sayHello$0$Proxy);
        } else {
            super.sayHello();
        }
    }

    final void CGLIB$sayHi$1() {
        super.sayHi();
    }

    public final void sayHi() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$sayHi$1$Method, CGLIB$emptyArgs, CGLIB$sayHi$1$Proxy);
        } else {
            super.sayHi();
        }
    }
    //......
}
```
* 上述代码只保留了我们定义的`sayHello`和`sayHi`方法，这个新生成的类继承了我们的`Student`(难怪之前八股文说cglib的动态代理是基于继承的,按照我们的使用，接口方式也是支持的)
* 我们简单分析一下`sayHello`方法(`sayHi`方法原理一样)
  * 首先给`var10000`赋值，它是一个`MethodInterceptor`类型(还记得吗，我们测试代码中设置回调就是该接口的实现类，是实现了`intercept`方法),值变量名称`this.CGLIB$CALLBACK_0`也说明是设置的回调
  * 然后`var10000`不为null则调用`var10000.intercept()`
  * 如果`var10000`为null则调用` super.sayHello();`，这就是我们最初目标类的方法
  
### jdk动态代理类
jdk动态代理也有方法能保存动态生成的类,在`main`方法开始加上一行如下代码：
* jdk1.8及jdk1.8以前
```markdown
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```
* jdk1.8以后
```markdown
System.getProperties().put("jdk.proxy.ProxyGenerator.saveGeneratedFiles", "true");
```

这里我们就不进行具体分析，原理不外乎方法中调用`InvocationHandler`接口实例的`invoke`方法

## 思考
* 经过上面对jdk、cglib动态代理的测试代码学习和源码阅读我们能够清楚地理解其原理。在面向切面编程(AOP)、Feign远程调用等有着非常广泛地应用。  


* 如果你能对java框架中动态代理相关地源码能够熟练阅读和理解，表明你已经掌握70%了  


* 如果你能自己运行动态代理实现一个@Feign注解,表明你已经掌握90%了(后续可能会出一篇相关文章)


* 我们知道两种动态代理，最终都调用了一个native方法生成类，倘如要我们自己实现这个方法啊呢，这要求我们对JVM内部，class文件格式，以及字节码的指令集都很熟悉，
这就需要我们掌握达到100%




















