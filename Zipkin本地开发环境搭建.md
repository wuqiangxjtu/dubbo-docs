### Zipkin本地开发环境搭建
#### 下载并安装ScalaIDE
大家都会，废话不多说。吐槽一下，IDE各种难用，scala版本还不一致，各种麻烦
#### 下载代码
进入一个文件夹，比如zipkin-home，clone源代码
`git clone https://github.com/wuqiangxjtu/zipkin.git`

#### 切换分支
+ 因为线上使用的是1.5.1版本的zipkin，目前的master做了较大幅度的修改，存储结构发生了一些变化，为了同目前线上的版本保持一致，我们基于1.5.1进行开发。使用` git checkout -b forhunter1.0.0 v1.5.1`创建了新的分支。
+ 如果已经有forhunter1.0.0分支，则需要执行`git checkout forhunter1.0.0`进行切换

#### build
如果直接切换到了forhunter1.0.0分支，则不用再做修改
+ 修改build.gradle
	``` groovy
	/**
	* 1. 新增镜像
	*/
	repositories {
	        maven { url 'http://repo.typesafe.com/typesafe/releases/' }
	        maven { url 'http://mirrors.hypo.cn/twttr-maven/'} //新增twitter镜像
	        maven { url 'https://maven.twttr.com/' }
	}
	
	/*
	*增加eclipse plugin
	*/
	apply plugin: 'ch.netzwerg.release'
	apply plugin: 'eclipse' //新增
	
	allprojects {
	    apply plugin: 'eclipse' //新增
	    apply plugin: 'java'
	    apply plugin: 'scala'
	    apply plugin: 'maven'
	    apply plugin: 'maven-publish'
	    apply plugin: 'com.jfrog.bintray'
	}
	``` 
> *需要说明的是，1.5.1的代码都是在scala2.10.x下编译的，需要修改。没找到IDE在哪里改，所以我只能用笨办法，导入以后一个一个改*
+ 进入zipkin-home, 执行`./gradlew eclipse`，时间比较长
+ 导入zipkin-common
+ 进入zipkin-scrooge，该项目中依赖了zipkin-thrift生成thriftscala，所以需要先执行gradle build, 导入zipkin-scrooge
+ 导入zipkin-collector
+ 导入zipkin-redis
+ 导入zipkin-cassandra
+ 导入zipkin-anormdb
+ 导入zipkin-query
+ 导入zipkin-query-core
+ 导入zipkin-query-service, 如果config文件夹编译报错，remove from build path
+ 导入zipkin-Hunter(如果没有，就需要在后续的步骤中新建)
+ 导入zipkine-web

#### 执行zipkin-query-service
+ 找到projectDir＝/Users/wuqiang/program/zipkin/zipkin-hunter/zipkin/zipkin-query-service
+ 在config/query-redis.scala中修改redis地址
``` 
//如果不是本地，改成远程redis地址
val storeBuilder = Store.Builder(
  redis.StorageBuilder("172.16.190.141", 6379), 
  redis.IndexBuilder("172.16.190.141", 6379)
)
```
+ 执行Main.scala, 设置参数为-f /Users/wuqiang/program/zipkin/zipkin-hunter/zipkin/zipkin-query-service/config/query-redis.scala

#### 执行zipkin-web
+ 找到projectDir＝/Users/wuqiang/program/zipkin/zipkin-hunter/zipkin/zipkin-web
+ 执行Main.scala, 设置参数为-zipkin.web.resourcesRoot=/Users/wuqiang/program/zipkin/zipkin-hunter/zipkin/zipkin-web/src/main/resources

#### 新建zipkin-hunter
> 如果已经有这个工程，就忽略此步骤

+ 新建一个gradle项目，zipkin-hunter
+ 修改build.gradle
``` groovy
dependencies {
    compile project(':zipkin-redis')
    compile project(':zipkin-query')

    compile "com.twitter:twitter-server_${scalaInterfaceVersion}:${commonVersions.twitterServer}"
    compile "com.twitter:finagle-zipkin_${scalaInterfaceVersion}:${commonVersions.finagle}"
    compile "com.twitter:finagle-stats_${scalaInterfaceVersion}:${commonVersions.finagle}"
}
```
+ 添加子项目：在settings.gradle中添加zipkin-Hunter
```
////////////////////////////////////////////////////
// Libraries wrapping clients to external services //
/////////////////////////////////////////////////////

include 'zipkin-zookeeper'
include 'zipkin-cassandra'
include 'zipkin-anormdb'
include 'zipkin-redis'
include 'zipkin-hunter' //新添加
```