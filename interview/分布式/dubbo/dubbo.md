# 1.dubbo spi 扩展机制

http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi.html

什么样的框架才算有良好的可扩展性呢？它必须要做到以下两点:

1. 作为框架的维护者，在添加一个新功能时，只需要添加一些新代码，而不用大量的修改现有的代码，即符合开闭原则。
2. 作为框架的使用者，在添加一个新功能时，不需要去修改框架的源码，在自己的工程中添加代码即可。

Dubbo很好的做到了上面两点。这要得益于Dubbo的微内核+插件的机制



## 1.1. 可扩展的几种解决方案

通常可扩展的实现有下面几种:

- Factory模式
- IoC容器
- OSGI容器

Dubbo作为一个框架，不希望强依赖其他的IoC容器，比如Spring，Guice。OSGI也是一个很重的实现，不适合Dubbo。最终Dubbo的实现参考了Java原生的SPI机制，但对其进行了一些扩展，以满足Dubbo的需求

Java SPI有以下的不足:

- 需要遍历所有的实现，并实例化，然后我们在循环中才能找到我们需要的实现。
- 配置文件中只是简单的列出了所有的扩展实现，而没有给他们命名。导致在程序中很难去准确的引用它们。
- 扩展如果依赖其他的扩展，做不到自动注入和装配
- 不提供类似于Spring的IOC和AOP功能
- 扩展很难和其他的框架集成，比如扩展里面依赖了一个Spring bean，原生的Java SPI不支持

## 1.2 Dubbo扩展点机制基本概念

### 扩展点(Extension Point 是一个Java的接口。

### 扩展(Extension)  扩展点的实现类。

###  扩展实例(Extension Instance)   扩展点实现类的实例。

###  扩展自适应实例(Extension Adaptive Instance)

如果称它为扩展代理类，可能更好理解些。扩展的自适应实例其实就是一个Extension的代理，它实现了扩展点接口。在调用扩展点的接口方法时，会根据实际的参数来决定要使用哪个扩展。比如一个IRepository的扩展点，有一个save方法。有两个实现MysqlRepository和MongoRepository。IRepository的自适应实例在调用接口方法的时候，会根据save方法中的参数，来决定要调用哪个IRepository的实现。如果方法参数中有repository=mysql，那么就调用MysqlRepository的save方法。如果repository=mongo，就调用MongoRepository的save方法。和面向对象的延迟绑定很类似。为什么Dubbo会引入扩展自适应实例的概念呢？

- Dubbo中的配置有两种，一种是固定的系统级别的配置，在Dubbo启动之后就不会再改了。还有一种是运行时的配置，可能对于每一次的RPC，这些配置都不同。比如在xml文件中配置了超时时间是10秒钟，这个配置在Dubbo启动之后，就不会改变了。但针对某一次的RPC调用，可以设置它的超时时间是30秒钟，以覆盖系统级别的配置。对于Dubbo而言，每一次的RPC调用的参数都是未知的。只有在运行时，根据这些参数才能做出正确的决定。
- 很多时候，我们的类都是一个单例的，比如Spring的bean，在Spring bean都实例化时，如果它依赖某个扩展点，但是在bean实例化时，是不知道究竟该使用哪个具体的扩展实现的。这时候就需要一个代理模式了，它实现了扩展点接口，方法内部可以根据运行时参数，动态的选择合适的扩展实现。而这个代理就是自适应实例。 自适应扩展实例在Dubbo中的使用非常广泛，Dubbo中，每一个扩展都会有一个自适应类，如果我们没有提供，Dubbo会使用字节码工具为我们自动生成一个。所以我们基本感觉不到自适应类的存在。后面会有例子说明自适应类是怎么工作的。

### 5.5 @SPI

@SPI注解作用于扩展点的接口上，表明该接口是一个扩展点。可以被Dubbo的ExtensionLoader加载。如果没有此ExtensionLoader调用会异常。

### 5.6 @Adaptive

@Adaptive注解用在扩展接口的方法上。表示该方法是一个自适应方法。Dubbo在为扩展点生成自适应实例时，如果方法有@Adaptive注解，会为该方法生成对应的代码。方法内部会根据方法的参数，来决定使用哪个扩展。 @Adaptive注解用在类上代表实现一个装饰类，类似于设计模式中的装饰模式，它主要作用是返回指定类，目前在整个系统中AdaptiveCompiler、AdaptiveExtensionFactory这两个类拥有该注解。

### 5.7 ExtensionLoader

类似于Java SPI的ExtensionLoader，负责扩展的加载和生命周期维护。

### 5.8 扩展别名

和Java SPI不同，Dubbo中的扩展都有一个别名，用于在应用中引用它们。比如

```bash
random=com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=com.alibaba.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
```

其中的random，roundrobin就是对应扩展的别名。这样我们在配置文件中使用random或roundrobin就可以了。

### 5.9 一些路径

和Java SPI从`/META-INF/services`目录加载扩展配置类似，Dubbo也会从以下路径去加载扩展配置文件:

- `META-INF/dubbo/internal`
- `META-INF/dubbo`
- `META-INF/services`

## 6. Dubbo的LoadBalance扩展点解读

在了解了Dubbo的一些基本概念后，让我们一起来看一个Dubbo中实际的扩展点，对这些概念有一个更直观的认识。

我们选择的是Dubbo中的LoadBalance扩展点。Dubbo中的一个服务，通常有多个Provider，consumer调用服务时，需要在多个Provider中选择一个。这就是一个LoadBalance。我们一起来看看在Dubbo中，LoadBalance是如何成为一个扩展点的。

### 6.1 LoadBalance接口

```java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {

    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```

LoadBalance接口只有一个select方法。select方法从多个invoker中选择其中一个。上面代码中和Dubbo SPI相关的元素有:

- @SPI([RandomLoadBalance.NAME](http://randomloadbalance.name/)) @SPI作用于LoadBalance接口，表示接口LoadBalance是一个扩展点。如果没有@SPI注解，试图去加载扩展时，会抛出异常。@SPI注解有一个参数，该参数表示该扩展点的默认实现的别名。如果没有显示的指定扩展，就使用默认实现。`RandomLoadBalance.NAME`是一个常量，值是"random"，是一个随机负载均衡的实现。 random的定义在配置文件`META-INF/dubbo/internal/com.alibaba.dubbo.rpc.cluster.LoadBalance`中:

```bash
random=com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=com.alibaba.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
leastactive=com.alibaba.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
consistenthash=com.alibaba.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
```

可以看到文件中定义了4个LoadBalance的扩展实现。由于负载均衡的实现不是本次的内容，这里就不过多说明。只用知道Dubbo提供了4种负载均衡的实现，我们可以通过xml文件，properties文件，JVM参数显式的指定一个实现。如果没有，默认使用随机。

![dubbo-loadbalance | left](https://raw.githubusercontent.com/vangoleo/wiki/master/dubbo/dubbo_loadbalance.png)

- @Adaptive("loadbalance") @Adaptive注解修饰select方法，表明方法select方法是一个可自适应的方法。Dubbo会自动生成该方法对应的代码。当调用select方法时，会根据具体的方法参数来决定调用哪个扩展实现的select方法。@Adaptive注解的参数`loadbalance`表示方法参数中的loadbalance的值作为实际要调用的扩展实例。 但奇怪的是，我们发现select的方法中并没有loadbalance参数，那怎么获取loadbalance的值呢？select方法中还有一个URL类型的参数，Dubbo就是从URL中获取loadbalance的值的。这里涉及到Dubbo的URL总线模式，简单说，URL中包含了RPC调用中的所有参数。URL类中有一个`Map parameters`字段，parameters中就包含了loadbalance。

### 6.2 获取LoadBalance扩展

Dubbo中获取LoadBalance的代码如下:

```java
LoadBalance lb = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalanceName);
```

使用ExtensionLoader.getExtensionLoader(LoadBalance.class)方法获取一个ExtensionLoader的实例，然后调用getExtension，传入一个扩展的别名来获取对应的扩展实例。

## 7. 自定义一个LoadBalance扩展

本节中，我们通过一个简单的例子，来自己实现一个LoadBalance，并把它集成到Dubbo中。我会列出一些关键的步骤和代码，也可以从这个地址(https://github.com/vangoleo/dubbo-spi-demo)下载完整的demo。

### 7.1 实现LoadBalance接口

首先，编写一个自己实现的LoadBalance，因为是为了演示Dubbo的扩展机制，而不是LoadBalance的实现，所以这里LoadBalance的实现非常简单，选择第一个invoker，并在控制台输出一条日志。

```java
package com.dubbo.spi.demo.consumer;
public class DemoLoadBalance implements LoadBalance {
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        System.out.println("DemoLoadBalance: Select the first invoker...");
        return invokers.get(0);
    }
}
```

### 7.2 添加扩展配置文件

添加文件:`META-INF/dubbo/com.alibaba.dubbo.rpc.cluster.LoadBalance`。文件内容如下:

```bash
demo=com.dubbo.spi.demo.consumer.DemoLoadBalance
```

### 7.3 配置使用自定义LoadBalance

通过上面的两步，已经添加了一个名字为demo的LoadBalance实现，并在配置文件中进行了相应的配置。接下来，需要显式的告诉Dubbo使用demo的负载均衡实现。如果是通过spring的方式使用Dubbo，可以在xml文件中进行设置。

```xml
<dubbo:reference id="helloService" interface="com.dubbo.spi.demo.api.IHelloService" loadbalance="demo" />
```

在consumer端的[dubbo:reference](dubbo:reference)中配置<loadbalance="demo">

### 7.4 启动Dubbo

启动Dubbo，调用一次IHelloService，可以看到控制台会输出一条`DemoLoadBalance: Select the first invoker...`日志。说明Dubbo的确是使用了我们自定义的LoadBalance。

## 总结

Dubbo SPI有以下的特点:

- 对Dubbo进行扩展，不需要改动Dubbo的源码
- 自定义的Dubbo的扩展点实现，是一个普通的Java类，Dubbo没有引入任何Dubbo特有的元素，对代码侵入性几乎为零。
- 将扩展注册到Dubbo中，只需要在ClassPath中添加配置文件。使用简单。而且不会对现有代码造成影响。符合开闭原则。
- dubbo的扩展机制设计默认值：@SPI("dubbo") 代表默认的spi对象
- Dubbo的扩展机制支持IoC,AoP等高级功能
- Dubbo的扩展机制能很好的支持第三方IoC容器，默认支持Spring Bean，可自己扩展来支持其他容器，比如Google的Guice。
- 切换扩展点的实现，只需要在配置文件中修改具体的实现，不需要改代码。使用方便。

# 二、Dubbo可扩展机制源码解析

http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi-2.html

## ExtensionLoader

ExtensionLoader 是最核心的类，负责扩展点的加载和生命周期管理。我们就以这个类开始吧。 ExtensionLoader 的方法比较多，比较常用的方法有:

- `public static  ExtensionLoader getExtensionLoader(Class type)`
- `public T getExtension(String name)`
- `public T getAdaptiveExtension()`

比较常见的用法有:

- `LoadBalance lb = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalanceName)`
- `RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getAdaptiveExtension()`

说明：在接下来展示的源码中，我会将无关的代码(比如日志，异常捕获等)去掉，方便大家阅读和理解。

1. getExtensionLoader方法 这是一个静态工厂方法，入参是一个可扩展的接口，返回一个该接口的ExtensionLoader实体类。通过这个实体类，可以根据name获得具体的扩展，也可以获得一个自适应扩展。

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        // 扩展点必须是接口
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        // 必须要有@SPI注解
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type without @SPI Annotation!");
        }
        // 从缓存中根据接口获取对应的ExtensionLoader
        // 每个扩展只会被加载一次
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            // 初始化扩展
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
}
    
private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

1. getExtension方法

```java
public T getExtension(String name) {
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        // 从缓存中获取，如果不存在就创建
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
}
```

getExtension 方法中做了一些判断和缓存，主要的逻辑在 createExtension 方法中。我们继续看 createExtension 方法。

```java
private T createExtension(String name) {
        // 根据扩展点名称得到扩展类，比如对于LoadBalance，根据random得到RandomLoadBalance类
        Class<?> clazz = getExtensionClasses().get(name);
        
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
              // 使用反射调用nesInstance来创建扩展类的一个示例
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 对扩展类示例进行依赖注入
        injectExtension(instance);
        // 如果有wrapper，添加wrapper
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
}
```

createExtension方法做了以下事情:

1. 先根据name来得到对应的扩展类。从ClassPath下`META-INF`文件夹下读取扩展点配置文件。
2. 使用反射创建一个扩展类的实例
3. 对扩展类实例的属性进行依赖注入，即IOC。
4. 如果有wrapper，添加wrapper。即AOP。

下面我们来重点看下这4个过程

1. 根据name获取对应的扩展类 先看代码:

```java
private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
}

// synchronized in getExtensionClasses
private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if (value != null && (value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName());
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }

        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
}
```

过程很简单，先从缓存中获取，如果没有，就从配置文件中加载。配置文件的路径就是之前提到的:

- `META-INF/dubbo/internal`
- `META-INF/dubbo`
- `META-INF/services`

1. 使用反射创建扩展实例 这个过程很简单，使用`clazz.newInstance())`来完成。创建的扩展实例的属性都是空值。
2. 扩展实例自动装配 在实际的场景中，类之间都是有依赖的。扩展实例中也会引用一些依赖，比如简单的Java类，另一个Dubbo的扩展或一个Spring Bean等。依赖的情况很复杂，Dubbo的处理也相对复杂些。我们稍后会有专门的章节对其进行说明，现在，我们只需要知道，Dubbo可以正确的注入扩展点中的普通依赖，Dubbo扩展依赖或Spring依赖等。
3. 扩展实例自动包装 自动包装就是要实现类似于Spring的AOP功能。Dubbo利用它在内部实现一些通用的功能，比如日志，监控等。关于扩展实例自动包装的内容，也会在后面单独讲解。

经过上面的4步，Dubbo就创建并初始化了一个扩展实例。这个实例的依赖被注入了，也根据需要被包装了。到此为止，这个扩展实例就可以被使用了。

## Dubbo SPI高级用法之自动装配

自动装配的相关代码在injectExtension方法中:

```java
private T injectExtension(T instance) {
    for (Method method : instance.getClass().getMethods()) {
        if (method.getName().startsWith("set")
                && method.getParameterTypes().length == 1
                && Modifier.isPublic(method.getModifiers())) {
            Class<?> pt = method.getParameterTypes()[0];
          
            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
                method.invoke(instance, object);
            }
        }
    }
    return instance;
}
```

要实现对扩展实例的依赖的自动装配，首先需要知道有哪些依赖，这些依赖的类型是什么。Dubbo的方案是查找Java标准的setter方法。即方法名以set开始，只有一个参数。如果扩展类中有这样的set方法，Dubbo会对其进行依赖注入，类似于Spring的set方法注入。 但是Dubbo中的依赖注入比Spring要复杂，因为Spring注入的都是Spring bean，都是由Spring容器来管理的。而Dubbo的依赖注入中，需要注入的可能是另一个Dubbo的扩展，也可能是一个Spring Bean，或是Google guice的组件，或其他任何一个框架中的组件。Dubbo需要能够从任何一个场景中加载扩展。在injectExtension方法中，是用`Object object = objectFactory.getExtension(pt, property)`来实现的。objectFactory是ExtensionFactory类型的，在创建ExtensionLoader时被初始化:

```java
private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

**objectFacory本身也是一个扩展，通过**`ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension())`来获取。

ExtensionFactory 有三个实现：

1. SpiExtensionFactory：Dubbo自己的Spi去加载Extension
2. SpringExtensionFactory：从Spring容器中去加载Extension
3. AdaptiveExtensionFactory: 自适应的AdaptiveExtensionLoader

这里要注意 AdaptiveExtensionFactory，源码如下:

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```

AdaptiveExtensionLoader类有@Adaptive注解。前面提到了，Dubbo会为每一个扩展创建一个自适应实例。如果扩展类上有@Adaptive，会使用该类作为自适应类。如果没有，Dubbo会为我们创建一个。所以`ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension())`会返回一个AdaptiveExtensionLoader实例，作为自适应扩展实例。 **AdaptiveExtensionLoader会遍历所有的ExtensionFactory实现**，尝试着去加载扩展。如果找到了，返回。如果没有，在下一个ExtensionFactory中继续找。Dubbo内置了两个ExtensionFactory，分别从Dubbo自身的扩展机制和Spring容器中去寻找。**由于ExtensionFactory本身也是一个扩展点，我们可以实现自己的ExtensionFactory，让Dubbo的自动装配支持我们自定义的组件。比如，我们在项目中使用了Google的guice这个 IOC 容器。我们可以实现自己的GuiceExtensionFactory，让Dubbo支持从guice容器中加载扩展。**

## Dubbo SPI高级用法之 AOP

在用Spring的时候，我们经常会用到AOP功能。在目标类的方法前后插入其他逻辑。比如通常使用Spring AOP来实现日志，监控和鉴权等功能。 Dubbo的扩展机制，是否也支持类似的功能呢？答案是yes。在Dubbo中，有一种特殊的类，被称为Wrapper类。通过装饰者模式，使用包装类包装原始的扩展点实例。在原始扩展点实现前后插入其他逻辑，实现AOP功能。

### 什么是Wrapper类

那什么样类的才是Dubbo扩展机制中的Wrapper类呢？Wrapper类是一个有复制构造函数的类，也是典型的装饰者模式。下面就是一个Wrapper类:

```java
class A{
    private A a;
    public A(A a){
        this.a = a;
    }
}
```

类A有一个构造函数`public A(A a)`，构造函数的参数是A本身。这样的类就可以成为Dubbo扩展机制中的一个Wrapper类。Dubbo中这样的Wrapper类有ProtocolFilterWrapper, ProtocolListenerWrapper等, 大家可以查看源码加深理解。

### 怎么配置Wrapper类

在Dubbo中Wrapper类也是一个扩展点，和其他的扩展点一样，也是在`META-INF`文件夹中配置的。比如前面举例的ProtocolFilterWrapper和ProtocolListenerWrapper就是在路径`dubbo-rpc/dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol`中配置的:

```text
filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=org.apache.dubbo.rpc.support.MockProtocol
```

在Dubbo加载扩展配置文件时，有一段如下的代码:

```java
try {  
  clazz.getConstructor(type);    
  Set<Class<?>> wrappers = cachedWrapperClasses;
  if (wrappers == null) {
    cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
    wrappers = cachedWrapperClasses;
  }
  wrappers.add(clazz);
} catch (NoSuchMethodException e) {}
```

这段代码的意思是，**如果扩展类有复制构造函数，就把该类存起来，供以后使用。有复制构造函数的类就是Wrapper类。通过`clazz.getConstructor(type)`来获取参数是扩展点接口的构造函数**。注意构造函数的参数类型是扩展点接口，而不是扩展类。 以Protocol为例。配置文件`dubbo-rpc/dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol`中定义了`filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper`。 ProtocolFilterWrapper代码如下：

```java
public class ProtocolFilterWrapper implements Protocol {

    private final Protocol protocol;

    // 有一个参数是Protocol的复制构造函数
    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
}
```

ProtocolFilterWrapper有一个构造函数`public ProtocolFilterWrapper(Protocol protocol)`，**参数是扩展点Protocol，所以它是一个Dubbo扩展机制中的Wrapper类。ExtensionLoader会把它缓存起来，供以后创建Extension实例的时候，使用这些包装类依次包装原始扩展点。**

## 扩展点自适应

前面讲到过，**Dubbo需要在运行时根据方法参数来决定该使用哪个扩展**，所以有了扩展点自适应实例。其实是一个扩展点的代理，将扩展的选择从Dubbo启动时，延迟到RPC调用时。**Dubbo中每一个扩展点都有一个自适应类，如果没有显式提供，Dubbo会自动为我们创建一个，默认使用Javaassist**。 先来看下创建自适应扩展类的代码:

```java
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                      instance = createAdaptiveExtension();
                      cachedAdaptiveInstance.set(instance); 
                }
            }        
    }

    return (T) instance;
}
```

继续看createAdaptiveExtension方法

```java
private T createAdaptiveExtension() {        
    return injectExtension((T) getAdaptiveExtensionClass().newInstance());
}
```

继续看getAdaptiveExtensionClass方法

```java
private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```

继续看createAdaptiveExtensionClass方法，绕了一大圈，终于来到了具体的实现了。看这个createAdaptiveExtensionClass方法，它首先会生成自适应类的Java源码，然后再将源码编译成Java的字节码，加载到JVM中。

```java
private Class<?> createAdaptiveExtensionClass() {
        String code = createAdaptiveExtensionClassCode();
        ClassLoader classLoader = findClassLoader();
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```

Compiler的代码，默认实现是javassist。

```java
@SPI("javassist")
public interface Compiler {
    Class<?> compile(String code, ClassLoader classLoader);
}
```

**createAdaptiveExtensionClassCode()方法中使用一个StringBuilder来构建自适应类的Java源码。方法实现比较长，这里就不贴代码了。这种生成字节码的方式也挺有意思的，先生成Java源代码，然后编译，加载到jvm中。通过这种方式，可以更好的控制生成的Java类**。而且这样也不用care各个字节码生成框架的api等。因为xxx.java文件是Java通用的，也是我们最熟悉的。只是代码的可读性不强，需要一点一点构建xx.java的内容。 下面是使用createAdaptiveExtensionClassCode方法为Protocol创建的自适应类的Java代码范例:

```java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```

大致的逻辑和开始说的一样，通过url解析出参数，解析的逻辑由@Adaptive的value参数控制，然后再根据得到的扩展点名获取扩展点实现，然后进行调用。如果大家想知道具体的构建.java代码的逻辑，可以看`createAdaptiveExtensionClassCode`的完整实现。 在生成的 Protocol$Adaptive 中，发现getDefaultPort和destroy方法都是直接抛出异常的，这是为什么呢？来看看Protocol的源码

```java
@SPI("dubbo")
public interface Protocol {

    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();
}
```

可以看到Protocol接口中有4个方法，但只有export和refer两个方法使用了@Adaptive注解。Dubbo自动生成的自适应实例，只有@Adaptive修饰的方法才有具体的实现。所以，Protocol$Adaptive 类中，也只有export和refer这两个方法有具体的实现，其余方法都是抛出异常。

