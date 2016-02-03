## Hunter系统的设计与实现
------------------

### 说明
后续可能涉及一些概念，例如：Trace，Span，Annotation等等，都来源于google的那篇论文。

#### 概述
如下图所示，是我们需要跟踪的一种常见的情况，有平行的调用，也有嵌套的调用。整个可以看成是一个树形结构，也就是一个跟踪树。图中画的是rpc调用，方法调用也是类似的道理。

![pic](https://github.com/wuqiangxjtu/share/tree/master/pics/4.png)

跟踪树也可以表示成下面这种形式：

![pic](https://github.com/wuqiangxjtu/share/tree/master/pics/5.png)

对应上图，我们明确一些概念：
+ Trace：我们可以把一个跟踪树看成是一个trace，可以是方法的一系列调用，也可以是服务之间的一系列调用，甚至两者混合。
+ Span：树中的每个节点称为一个Span，根节点被称为root Span。一个Span可能包含多个信息，例如：方法调用开始的信息和方法调用结束的信息，RPC span还会包含来自客户端和服务端两者的信息。如下图所示：

![pic](https://github.com/wuqiangxjtu/share/tree/master/pics/6.png)

+ annotation：span中的一个点，包含事件名称，时间等信息。前边所谓的一个Span包含多个信息，实际上就是包含多个annotation
+ host: 标识一个服务，包含ip，port，服务名称等信息。对于方法调用来说，一个trace的host是可能是完全一样的

对于我们来说，由于我们的收集和展示都使用了zipkin，所以还有"存储结构"、“数据读取”、“数据展示”等方面的问题，都不需要我们关心太多。

### 模块划分
系统的主要功能有两个模块`hunter-core`和`hunter-core-http`, 其中：
+ hunter-core：提供了跟踪的基本功能
+ hunter-core-http: 针对http服务进行了简化和加强
    + 通过HttpServletRequest获取信息直接用于初始化，简化参数
    + 系统间通过http请求进行调用时，能够在一个Trace中统一跟踪

系统还提供一些示例项目：例如`hunter-mvc-spring`,`hunter-mvc-struts2`等：
+ hunter-mvc-spring: 针对Spring MVC的示例工程
+ hunter-mvc-strtus2: 针对Struts2的示例工程

### Hunter-core
#### 使用方式
Hunter-core的实现不依赖与Spring，需要显式的调用方法发送跟踪日志。从一个Testcase开始：
+ 首先，在调用跟踪方法前，需要进行初始化；调用完成后，需要进行清理。
``` java
    public void testService1() {
        Hunter.startTracer("127.0.0.1", 8090, "test-service", spanCollector, null);//初始化
        firstService.serviceA(); //需要跟踪的方法
        Hunter.endTrace();//清理
    }
```
+ 针对需要跟踪的方法，在方法内的开始部分新建一条跟踪记录；在方法退出前，发送该跟踪记录。
``` java
public void serviceA() {
        Hunter.newSpanWithServerRecvAnnotation("serviceA"); //新建跟踪记录
        //here, do all the things， serviceB for example
        Hunter.submitServerSendAnnotationAndCollect();//提交跟踪记录  
}
```
``` java
public void serviceB() {
    Hunter.newSpanWithServerRecvAnnotation("serviceB"); //新建跟踪记录
    try { //do something
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    Hunter.submitServerSendAnnotationAndCollect();//提交跟踪记录
}
```

执行结果如下：
**A：** [2015-10-22 16:32:31,216]main - [ Trace id: -3032503177755793289 ] - brave.LoggingSpanCollector:collect INFO   - Span(trace_id:-3032503177755793289, name:serviceB, id:6941959007409225568, parent_id:-3032503177755793289, annotations:[Annotation(timestamp:1445502750209000, value:sr, host:Endpoint(ipv4:2130706433, port:8090, service_name:test-service)), Annotation(timestamp:1445502751213000, value:ss, host:Endpoint(ipv4:2130706433, port:8090, service_name:test-service))], binary_annotations:null)

**B: ** [2015-10-22 16:32:31,217]main - [ Trace id: -3032503177755793289 ] - brave.LoggingSpanCollector:collect INFO   - Span(trace_id:-3032503177755793289, name:serviceA, id:-3032503177755793289, annotations:[Annotation(timestamp:1445502750209000, value:sr, host:Endpoint(ipv4:2130706433, port:8090, service_name:test-service)), Annotation(timestamp:1445502751216000, value:ss, host:Endpoint(ipv4:2130706433, port:8090, service_name:test-service))], binary_annotations:null)

有以下需要注意的点：
+ 两条数据的Trace id是相同的, 合起来是一个Trace
+ 每条数据都是一个Span，中间调用了两个方法，所以一共有三个Span，包含一个root Span
+ A的parent_id是B的id，说明B是A的父节点
+ 一个Span中有两个Annotation，value分别是ss和sr，分别代表了”service send”和“service recevie”
+ 这里binary_annotation对我们不重要，暂时忽略

*具体例子可以查看HunterTest.java*

#### 生成与保存Trace
+ Hunter可能需要记录每次操作，而操作通常是通过不同的工作线程执行的，所以自然而然可以想到使用一个ThreadLocal变量来保存。
+ 在每个

#### 生成与保存Span
因为方法调用的顺序是 A send -> B send -> B receive -> A receive，
对于Span A，最先开始，但是最晚结束，所以自然可以想到使用Stack来处理。


#### Hunter.java
Hunter类是外部访问的入口，其中最重要的是：
`public static ThreadLocal<Tracer> TRACER = new ThreadLocal<Tracer>();`
TRACER是一个ThreadLocal变量，针对每个线程保存了跟踪所需要的信息。Hunter会调用TRACER进行具体的工作。例如newSpan方法：
``` java
public static void newSpan(String spanName) {
    try {
        Hunter.TRACER.get().newSpan(spanName); //调用线程变量的newSpan
    } catch (Exception e) {
        //对所有方法进行try catch，在出错时也避免影响正常功能
        LOG.warn("New Span with spanName:" + spanName + " exception", e);
    }
}
```

