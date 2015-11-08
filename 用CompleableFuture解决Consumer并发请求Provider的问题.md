## 用CompleableFuture解决Consumer并发请求Provider的问题

### 问题
在使用dubbo进行服务化过程中，API粒度的控制是非常重要的.Dubbo官方文档的最佳实践推荐每个服务接口应该尽可能的大粒度，每个服务方法应该代表一个功能，而不是某个功能的一个步骤，否则将面临分布式事务的问题，而Dubbo并未提供分布式事务。
我们还是会遇到这样的一些场景，例如Dashboard的首页，API层提供给前端的接口需要同时获取扶翼、龙渊、轩辕等产品线的数据。如果使用一个统一的服务接口返回所有的数据至少会带来两个方面的问题：
+ 这个接口在其他地方将很难被复用，容易引起接口的爆炸
+ 增加一条产品线会导致接口的变化，复用的难度进一步加大

### 实例
为了更方便的说明问题，我们可以把上面的问题转换成为我们生活中熟悉的事例。如下图所示：

![][image-1]

吃饭的过程进行如下分解：
+ 做菜：1. 洗菜(wash vegetable)；2. 炒菜(cook vegetable)
+ 做米饭：1. 淘米(wash rice)；2. 煮饭(cook rice)
+ 烧开水(用来喝)(boil water)
+ 当前三项工作都完成后，就可以开始吃饭了(person.eat)

#### 建立实体类
##### dubbo-demo-api

``` java
`public class Vegetable implements Serializable {
private static final long serialVersionUID = -2766776078548112038L;
}
public class Rice implements Serializable {
private static final long serialVersionUID = 3369278509187557523L;
}
public class Water implements Serializable {
private static final long serialVersionUID = -3108169097597662406L;
}
```
`
#### 顺序处理
传统的方法是逐个依次调用

[image-1]:	https://github.com/wuqiangxjtu/dubbo-docs/blob/master/pics/1.png