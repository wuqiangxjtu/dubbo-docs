---
title: Provider启动
notebook: dubbo
tags:dubbo
---

### Provider启动
#### 启动container
container的启动就是调用container的start方法，log4jcontainer的启动没有什么特别，我们重点看一下springcontainer的启动。

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
返回Url的List，
[registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&owner=william&pid=12702&registry=zookeeper&timestamp=1437635431172]

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

+ final int defaultPort = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name).getDefaultPort()
getExtension(name)时会发现有两个protocol的wrapper：com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper和com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper，wrapper有点像装饰模式，可以进行AOP。

+ 准备URL
{methods=sayHello, timestamp=1437656363034, dubbo=2.0.0, application=demo-provider, side=provider, owner=william, pid=12702, interface=com.alibaba.dubbo.demo.DemoService, anyhost=true, loadbalance=roundrobin}
URL:
dubbo://10.209.76.72:20880/com.alibaba.dubbo.demo.DemoService
