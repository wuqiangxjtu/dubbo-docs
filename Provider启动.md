---
title: Provider启动
notebook: dubbo
tags:dubbo,dubbo源码
---

### Provider启动主流程
#### 启动container
container的启动就是调用container的start方法，log4jcontainer的启动没有什么特别，我们重点看一下spring container的启动。

```java
public void start() {
       String configPath = ConfigUtils.getProperty(SPRING_CONFIG);
       if (configPath == null || configPath.length() == 0) {
	       //classpath*:META-INF/spring/*.xml
           configPath = DEFAULT_SPRING_CONFIG;
       }
       context = new ClassPathXmlApplicationContext(configPath.split("[,\\s]+"));
       context.start();
}
```

ClassPathXmlApplicationContext启动容器的时候，因为dubbo扩展了spring的标签，所以会调用DubboNamespaceHandler的中配置的方法。

```java
registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
 //service
registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
```

示例配置了service，所以会调用`DubboBeanDefinitionParser(ServiceBean.class, true)`来解析。

```xml
<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService" />
``` 

DubboBeanDefinitionParser获取到beanDefinition：Root bean: class [com.alibaba.dubbo.config.spring.ServiceBean];

#### ServiceBean
接下来会初始化ServiceBean，ServiceConfig是基类，所以先进行初始化。
在ServiceConfig中，会先初始化两个静态属性Protocal和ProxyFactory. Protocal的实现有dubbo，hession，rmi, http等。ProxyFactory的实现有jdk和javaassit

```java
//传输的协议类
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
//生成代理类的工厂
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```

##### Protocal 初始化
+ 从cachedDefaultName得到，默认是dubbo
+ 读取配置：com.alibaba.dubbo.rpc.Protocol
+ registry-api包：
  * registry=com.alibaba.dubbo.registry.integration.RegistryProtocol
+ rpc-api包：
  * filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
  * listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
  * mock=com.alibaba.dubbo.rpc.support.MockProtocol
  * filter、listener都加入了ExtensionLoader的wrappers属性
+ rpc-default包:
  * dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
+ rpc-injvm包：
  * injvm=com.alibaba.dubbo.rpc.protocol.injvm.InjvmProtocol
+ Protocal初始化为：`alibaba.dubbo.config.support.AdpativeProtocol`
**AdpativeProtocol本来不存在，真实环境中是使用javaassit编译生成的。这里反编译出来是为了调试方便。**

##### ProxyFactory 初始化
+ 从cacheDefaultName得到，默认是javassist
+ 读取配置，rpc-api包：
  * stub=com.alibaba.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper
  * jdk=com.alibaba.dubbo.rpc.proxy.jdk.JdkProxyFactory
  * javassist=com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory

##### afterPropertiesSet方法
设置provider，application，module，monitor，protocol等。

##### onApplicationEvent方法
调用ServiceConfig.export()->ServiceConfig.doExport()->doExportUrls()

#### doExport 调用流程
+ **`checkDefault()`**
+ **`checkInterfaceAndMethods(interfaceClass, methods);`**
+ **`checkRef()`**
+ **`checkApplication()`**
+ **`checkRegistry()`**
+ **`checkProtocol()`**
+ **`appendProperties()`**
+ **`checkStubAndMock(interfaceClass)`**
+ **`doExportUrls()`**

#### loadRegistries
+ 通过注册中心的配置，得到注册中心的registryURLs:
[registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&owner=william&pid=12702&registry=zookeeper&timestamp=1437635431172]
+ protocols是通信所使用的协议：例如`[<dubbo:protocol name="dubbo" port="20880" id="dubbo" />]`
+ 随后，对于每个protocol（protocolConfig）都调用一次doExportUrlsFor1Protocol(protocolConfig, registryURLs)

#### doExportUrlsFor1Protocol(protocolConfig, registryURLs)
+ 首先获取host，port
	host首先调用 host = InetAddress.getLocalHost().getHostAddress()；如果没有取到，则是向注册中心建立一个socket连接，然后获取本地的host。

```java
SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
socket.connect(addr, 1000);
host = socket.getLocalAddress().getHostAddress();
```

+ 根据不同的扩展获取不同的defaultPort

```java
final int defaultPort = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name).getDefaultPort();
if (port == null || port == 0) {
    port = defaultPort;
}
```

例如：ThriftProtocol的默认端口是40880

+ 拼接URL:dubbo://10.209.79.51:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&loadbalance=roundrobin&methods=sayHello&owner=wuqiang&pid=56203&side=provider&timeout=10000&timestamp=1447643527460
+ 首先进行本地暴露(exportLocal), 然后进行远程暴露

#### 本地暴露exportLocal
+ URL local = injvm://127.0.0.1/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&loadbalance=roundrobin&methods=sayHello&owner=wuqiang&pid=56203&side=provider&timeout=10000&timestamp=1447643527460
+ 本地暴露的主要逻辑都在`Exporter<?> exporter = protocol.export(proxyFactory.getInvoker(ref, (Class) interfaceClass, local));`中

#### proxyFactory.getInvoker(ref, (Class)interfaceClass, local)
+ proxyFactory调用的是AdaptiveProxyFactory, 默认情况下，会使用javassistFactory, 外层会使用StubProxyFactoryWrapper进行包装
+ 对于getInvoker操作，StubProxyFactoryWrapper并没有做额外的工作，而是直接调用了javassistFactory.getInvoker
+ javaassistFactory.getInvoker返回了一个AbstractProxyInvoker，invoker表示一个可以被调用的对象，consumer对provider的调用，最终会传递给invoker来执行。

##### JavaassistProxyFactory.getInvoker
+ 首先，根据proxy以及type等信息，生成一个Wrapper，wrapper用来代理所有对proxy的请求。(这里有个疑问，因为wrapper的底层依然要使用动态代理的invoke方法来调用服务端的方法,为什么不直接使用动态代理生成一个对象，而要使用这样一个统一的Wrapper)
+ 返回AbstractProxyInvoker，把调用委托给wrapper执行

```java
return new AbstractProxyInvoker<T>(proxy, type, url) {
    @Override
    protected Object doInvoke(T proxy, String methodName, 
                              Class<?>[] parameterTypes, 
                              Object[] arguments) throws Throwable {
        return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
    }
};
``` 

#### protocol.export
+ 这里的protocol是AdaptiveProtocol
+ 获取到的Protocol的扩展是,`ProtocolFilterWrapper->ProtocolListenerWrapper->InjvmProtocol`

##### ProtocolFilterWrapper.export
+ `protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));`
+ 首先调用buildInvokerChain构造了一个invoker chain，然后交给了ProtocolListenerWrapper.export

###### buildInvokerChain

+ 首先，获取到一个filter数组

```
com.alibaba.dubbo.rpc.filter.EchoFilter@1a5b6f42, 
com.alibaba.dubbo.rpc.filter.ClassLoaderFilter@5038d0b5, 
com.alibaba.dubbo.rpc.filter.GenericFilter@32115b28, 
com.alibaba.dubbo.rpc.filter.ContextFilter@2ad48653, 
com.alibaba.dubbo.rpc.protocol.dubbo.filter.TraceFilter@6bb4dd34, 
com.alibaba.dubbo.rpc.filter.TimeoutFilter@7d9f158f, 
com.alibaba.dubbo.monitor.support.MonitorFilter@45efd90f, 
com.alibaba.dubbo.rpc.filter.ExceptionFilter@4b8729ff
```

+ 然后倒序循环处理，先后获取的是ExceptionFilter
+ 把last赋值给next,然后通过调用`filter.invoke(next, invocation);`对next进行包装
+ 所以，可以看作用invoker包装了filter，invoker和next，调用invoker.invoke时，委托filter进行处理，并且把next传递进去
+ 最后返回last
+ 所以最后返回一个invoker, 调用invoke方法时，会依次调用filter，顺序为从echoFilter开始，exceptionFilter结束

##### ProtocolListenerWrapper.export
+ 注册了一个listener，这里暂时不看
+ 调用后续的InjvmProtocol.export

##### InjvmProtocol.export
+ `new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);`包装一个InjvmExporter返回

#### 远程暴露
+ registryURL:[registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&client=curator&dubbo=2.0.0&owner=wuqiang&pid=56203&registry=zookeeper&timestamp=1447643514896]
+ 传入getInvoker: registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&client=curator&dubbo=2.0.0&export=dubbo%3A%2F%2F10.209.79.51%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26loadbalance%3Droundrobin%26methods%3DsayHello%26owner%3Dwuqiang%26pid%3D56203%26side%3Dprovider%26timeout%3D10000%26timestamp%3D1447643527460&owner=wuqiang&pid=56203&registry=zookeeper&timestamp=1447643514896
+ getInvoker的过程与本地暴露几乎一致
+ `Exporter<?> exporter = protocol.export(invoker);` 如果是Registry，则走if分支，调用RegistryProtocal的export方法。先调用doLocalExport暴露服务，然后把服务在注册中心中进行注册，然后返回一个包装过的Exporter。

#### protocol.export(invoker)
+ 获取到的Protocol的扩展是,`ProtocolFilterWrapper->ProtocolListenerWrapper->RegistryProtocol->AdaptiveProtocol`

```java
//ProtocolFilterWrapper
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        //如果是协议是Registry，直接调用ProtocolListenerWrapper.export
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
        }
        return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
}

//ProtocolListenerWrapper
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException { 
        //如果是协议是Registry，直接调用RegistryProtocol.export
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
        }
        return new ListenerExporterWrapper<T>(protocol.export(invoker), 
                Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                        .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
}

//RegistryProtocol
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker, 这里的doLocalExport与前面的本地暴露不同，这里的协议不是injvm的
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
    //registry provider
    final Registry registry = getRegistry(originInvoker);
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    registry.register(registedProviderUrl);
    // 订阅override数据
    // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //保证每次export都返回一个新的exporter实例
    return new Exporter<T>() {
        public Invoker<T> getInvoker() {
            return exporter.getInvoker();
        }
        public void unexport() {
          try {
            exporter.unexport();
          } catch (Throwable t) {
              logger.warn(t.getMessage(), t);
            }
            try {
              registry.unregister(registedProviderUrl);
            } catch (Throwable t) {
              logger.warn(t.getMessage(), t);
            }
            try {
              overrideListeners.remove(overrideSubscribeUrl);
              registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
            } catch (Throwable t) {
              logger.warn(t.getMessage(), t);
            }
        }
    };
}
```

##### doLocalExport
+ AdaptiveProtocol默认调用DubboProtocol进行发布
+ `ProtocolFilterWrapper->ProtocolListenerWrapper`调用过程与本地暴露时相同，直到调用DubboProtocol.export

```java
@SuppressWarnings("unchecked")
private <T> ExporterChangeableWrapper<T>  doLocalExport(final Invoker<T> originInvoker){
    String key = getCacheKey(originInvoker);
    ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
    if (exporter == null) {
        synchronized (bounds) {
            exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
            if (exporter == null) {
                final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
                exporter = new ExporterChangeableWrapper<T>((Exporter<T>)protocol.export(invokerDelegete), originInvoker);
                bounds.put(key, exporter);
            }
        }
    }
    return (ExporterChangeableWrapper<T>) exporter;
}
```

##### DubboProtocol.export(Invoker invoker)
+ DubboProtocol中有个属性:private ExchangeHandler requestHandler = new ExchangeHandlerAdapter(), 在接到客户端请求之后，会被调用。
