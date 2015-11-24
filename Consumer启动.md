---
title: Consumer启动
notebook: dubbo
tags:dubbo源码, dubbo
---

### Consumer启动
懒，暂时不加时序图了

#### 启动Container
这里的逻辑和Provider是一致的，不同的是Spring的配置文件中，Consumer使用的是`<dubbo:reference id="demoService" interface="com.alibaba.dubbo.demo.DemoService" timeout="10000" />`

#### ReferenceBean
调用了ReferenceConfig的getObject()方法

#### ReferenceConfig
调用了getObject()->get()->init()->createProxy()

##### createProxy
###### 生成proxy
+ 获取一个Proxy，代理所有的请求。没有直接利用反射，但方式是类似的，通过一个InvokerInvocationHandler的invoke方法代理了所有调用，在InvokerInvocationHandler中，会把请求转发给参数invoker，`invoker.invoke(new RpcInvocation(method, args)).recreate();`。

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}

```

###### 获取invoker
+ 这里的过程和provider的处理类似，Invoker是通过`invoker = refprotocol.refer(interfaceClass, url);`语句获取的,refprotocol是AdaptiveProtocol，获取到的Protocol的扩展是,`ProtocolFilterWrapper->ProtocolListenerWrapper->DubboProtocol`
+ `DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);`
+ 请求最终会转发到AbstractInvoker的invoke方法，最后由DubboInvoker的doInvoker(Invocation invocation)方法处理

##### 共享连接
+ getClients, 如果使用了共享连接，则会调用getShareClient(URL url)

```java
    /**
     *获取共享连接 
     */
    private ExchangeClient getSharedClient(URL url){
        String key = url.getAddress();
        //ConcurrentHashMap<String, ReferenceCountExchangeClient>(),用来缓存链接
        ReferenceCountExchangeClient client = referenceClientMap.get(key);
        if ( client != null ){
            if ( !client.isClosed()){
                client.incrementAndGetCount();
                return client;
            } else {
                referenceClientMap.remove(key);
            }
        }
        //如果没有连接，则新建一个， client = Exchangers.connect(url ,requestHandler);
        ExchangeClient exchagneclient = initClient(url);
        
        client = new ReferenceCountExchangeClient(exchagneclient, ghostClientMap);
        referenceClientMap.put(key, client);
        ghostClientMap.remove(key);
        return client; 
    }
```

##### 异步转同步
异步转同步是dubbo中比较有特点的一部分，流程如下：
+ 如果是双工通信，并且是同步的，则会调用一下语句发送请求并返回结果

```java
(Result) currentClient.request(inv, timeout).get();
```
1. 首先，currentClient是HeaderExchangeChannel，request方法实现如下, 说明见注释

```java
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }

    //1. 构建request， new Request()有mId属性，用来标识每次请求
    Request req = new Request();
    req.setVersion("2.0.0");
    req.setTwoWay(true);
    req.setData(request);

    //2. 构建Future
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try{

    //3. 发送请求，这里的Channel是一个接口，可以用netty，mina等不同的框架实现,NettyClient实现了Channel，
    //所以这里调用的实际上是NettyClient的send方法，不过NettyClient的send方法是在其父类AbstractClient中实现的
        channel.send(req);

    }catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    //4. 返回Future
    return future;
}

```

2. 得到DefaultFuture后，调用get()方法，代码如下, 流程见注释

```java
private final Lock                            lock = new ReentrantLock();
private final Condition                       done = lock.newCondition();

public Object get(int timeout) throws RemotingException {
    if (timeout <= 0) {
        timeout = Constants.DEFAULT_TIMEOUT;
    }
    //1. 判断response是否已经生成
    if (! isDone()) {
        long start = System.currentTimeMillis();
        lock.lock();
        try {
        	//2. 如果没有生成，等待到生成或超时
            while (! isDone()) {
            	//3. 等待是通过Condition对象来实现的
                done.await(timeout, TimeUnit.MILLISECONDS);
                if (isDone() || System.currentTimeMillis() - start > timeout) {
                    break;
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            lock.unlock();
        }
        if (! isDone()) {
            throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
        }
    }
    //4. 返回response
    return returnFromResponse();
}

public boolean isDone() {
    return response != null;
}

private Object returnFromResponse() throws RemotingException {
    Response res = response;
    if (res == null) {
        throw new IllegalStateException("response cannot be null");
    }
    if (res.getStatus() == Response.OK) {
        return res.getResult();
    }
    if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
        throw new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage());
    }
    throw new RemotingException(channel, res.getErrorMessage());
}   
```

3. 那么`done.await(timeout, TimeUnit.MILLISECONDS);`是何时得到通知的呢，我们知道，await一般是用signal来通知的。
+ client发送请求以后，await进入等待状态
+ 服务端处理request,然后返回response
+ client的HeaderExchangerHanlder的receive方法被触发，收到的message是Response，所以会调用handleResponse方法`handleResponse(channel, (Response) message);`
+ HandleResponse调用`DefaultFuture.received(channel, response);`

```java
//保存了Future
private static final Map<Long, DefaultFuture> FUTURES   = new ConcurrentHashMap<Long, DefaultFuture>();

//
public static void received(Channel channel, Response response) {
    try {
    	//获取Response id对应的Future，这里的id和request里的id是一致的
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
        	//通知Future，结果已经获取
            future.doReceived(response);
        } else {
            logger.warn("The timeout response finally returned at " 
                        + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date())) 
                        + ", response " + response 
                        + (channel == null ? "" : ", channel: " + channel.getLocalAddress() 
                            + " -> " + channel.getRemoteAddress()));
        }
    } finally {
        CHANNELS.remove(response.getId());
    }
}

private void doReceived(Response res) {
    lock.lock();
    try {
        response = res;
        if (done != null) {
            done.signal();
        }
    } finally {
        lock.unlock();
    }
    if (callback != null) {
        invokeCallback(callback);
    }
}

```



