---
title: 用CompleableFuture解决Consumer并发请求Provider的问题
notebook: dubbo
tags:dubbo
---
## 用CompleableFuture解决Consumer并发请求Provider的问题

### 问题
在使用dubbo进行服务化过程中，API粒度的控制是非常重要的.Dubbo官方文档的最佳实践推荐每个服务接口应该尽可能的大粒度，每个服务方法应该代表一个功能，而不是某个功能的一个步骤，否则将面临分布式事务的问题，而Dubbo并未提供分布式事务。
我们还是会遇到这样的一些场景，
例如Dashboard的首页，API层提供给前端的接口需要同时获取扶翼、龙渊、轩辕等产品线的数据。如果使用一个统一的服务接口返回所有的数据至少会带来两个方面的问题：
+ 这个接口在其他地方将很难被复用，容易引起接口的爆炸
+ 增加一条产品线会导致接口的变化，复用的难度进一步加大

### 实例
为了更方便的说明问题，我们可以把上面的问题转换成为我们生活中熟悉的事例。如下图所示：

![图片1](http://7xo7zr.com1.z0.glb.clouddn.com/1.png)

吃饭的过程进行如下分解：
+ 做菜：1. 洗菜(wash vegetable)；2. 炒菜(cook vegetable)
+ 做米饭：1. 淘米(wash rice)；2. 煮饭(cook rice)
+ 烧开水(用来喝)(boil water)
+ 当前三项工作都完成后，就可以开始吃饭了(person.eat)

#### 建立实体类
##### dubbo-demo-api

```java

public class Vegetable implements Serializable {
  private static final long serialVersionUID = -2766776078548112038L;
}
public class Rice implements Serializable {
  private static final lo ng serialVersionUID = 3369278509187557523L;
}
public class Water implements Serializable {
  private static final long serialVersionUID = -3108169097597662406L;
}

```

#### 新建Service接口
##### dubbo-demo-api

```java

public interface VegetableService {
	Vegetable wash(Vegetable vegetable);
	Vegetable cook(Vegetable vegetable);
}
public interface RiceService {
	Rice wash(Rice rice);
	Rice cook(Rice rice);
}
public interface WaterService {
	Water boil(Water water);
}
public interface PersonService {
	void eat(Vegetable vegetable, Rice rice, Water water);
}
```

#### 新建ServiceImpl
##### dubbo-demo-provider

```java
public class VegetableServiceImpl implements VegetableService{
    
    public Vegetable wash(Vegetable vegetable) {
        System.out.println("------->wash vegetable");
        SleepUtil.sleep(1500);
        return vegetable;
    }
    
    public Vegetable cook(Vegetable vegetable) {
        System.out.println("------->cook vegetable");
        SleepUtil.sleep(1500);
        return vegetable;
    }
}

public class VegetableServiceImpl implements VegetableService{
    
    public Vegetable wash(Vegetable vegetable) {
        System.out.println("------->wash vegetable");
        SleepUtil.sleep(1500);
        return vegetable;
    }
    
    public Vegetable cook(Vegetable vegetable) {
        System.out.println("------->cook vegetable");
        SleepUtil.sleep(1500);
        return vegetable;
    }
}

public class WaterServiceImpl implements WaterService{

    public Water boil(Water water) {
        System.out.println("------->boil water");
        SleepUtil.sleep(1000);
        return water;
    }
}

public class PersonServiceImpl implements PersonService{

    public void eat(Vegetable vegetable, Rice rice, Water water) {
        System.out.println("------->person eat");
        SleepUtil.sleep(500);
    }

}
```

#### 顺序处理
传统的方法是逐个依次调用各个服务接口，如下所示：
##### dubbo-demo-consumer

```java
Vegetable vegetable = new Vegetable();
Rice rice = new Rice();
Water water = new Water();

System.out.println("------------->serial begin<---------");
long begin = System.currentTimeMillis();
vegetableService.wash(vegetable);
vegetableService.cook(vegetable);
riceService.wash(rice);
riceService.cook(rice);
waterService.boil(water);
personService.eat(vegetable, rice, water);
System.out.println("Cost " + (System.currentTimeMillis() - begin) + " millis");
System.out.println("--------------->serial end<---------");
```

从结果可以看出，每个service都是顺序调用的。

```
Provider：
------->wash vegetable
------->cook vegetable
------->wash rice
------->cook rice
------->boil water
------->person eat 

Consumer：
------------->serial begin<---------
Cost 7786 millis
--------------->serial end<---------
```

如果服务接口非常多，并且有的接口比较耗时，那么整个调用消耗的时间非常长。

#### Future
可以把服务接口放在Future中执行，Dubbo本身也提供了异步调用的方法[Dubbo异步调用](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-%E5%BC%82%E6%AD%A5%E8%B0%83%E7%94%A8)。
具体的代码这里就不再赘述，但是这样做也有一个显著的缺点：`vegetableService.wash`可以异步执行，但是在该方法返回之前，无法调用`vegetableService.cook`方法，也就是说，只有`vegetableService.wash;riceService.wash;waterService.boil`三个方法可以并发执行。

#### CompleableFuture
CompleableFuture是Java8提供的新功能，使用CompleableFuture重构过的代码如下：

```java
System.out.println("------------->concurrent begin<---------");
long begin2 = System.currentTimeMillis();
final CompletableFuture<Vegetable> vegetableFuture = CompletableFuture.supplyAsync(
        () -> {vegetableService.wash(vegetable); return vegetable;})
        .thenApply((v)->{vegetableService.cook(v); return v;});
final CompletableFuture<Rice> riceFuture = CompletableFuture.supplyAsync(
        () -> {riceService.wash(rice); return rice;})
        .thenApply((r)->{riceService.cook(r); return r;});
final CompletableFuture<Water> waterFuture = CompletableFuture.supplyAsync(
        ()->{waterService.boil(water);return water;});
personService.eat(vegetableFuture.get(), riceFuture.get(), waterFuture.get());
System.out.println("Cost " + (System.currentTimeMillis() - begin2) + " millis");
System.out.println("--------------->concurrent end<---------");
```

调用的结果如下，可以看出，每条支线是并发执行的：

```
Provider:
------->wash vegetable
------->wash rice
------->boil water
------->cook rice
------->cook vegetable
------->person eat

Consumer:
------------->concurrent begin<---------
Cost 3541 millis
--------------->concurrent end<---------
```


#### 时间对比
>顺序执行：Cost 7786 millis

>并发执行：Cost 3541 millis




