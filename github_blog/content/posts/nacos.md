---
title: "Nacos"
date: 2023-12-25T09:02:21+08:00
draft: true
---

Nacos 本身就是一个 springboot项目

```
nacos-1.4.2
|-- address
|-- api
|-- auth
|-- client
|-- cmdb
|-- common
|-- config
|-- consistency
|-- console #http://127.0.0.1:8848/nacos/index.html 打开的页面就是在访问console 模块
|-- console-ui
|-- core
|-- distribution
|-- doc
|-- example
|-- istio
|-- logs
|-- naming #实际项目间进行服务注册发现用的是nacos提供的 restful的openapi   https://nacos.io/zh-cn/docs/open-api.html
|-- resources
|-- style
|-- sys
|-- test
`-- work
```



nacos默认的数据库是什么

Nacos 以standalone（单机模式）启动后，默认使用的内嵌式数据库 Derby ，平时在nacos控制台用的namespace 和配置管理都是存储在这里

  ![](derby-data.png)

### 服务注册

服务注册到Nacos以后，会保存在一个本地注册表中，其结构如下：

```
/**
 * Map(namespace, Map(group::serviceName, Service)).
 */
private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
```

  ![](注册表结构.png)





1. namespace
2. group
3. Service
4. Cluster
5. Instance



首先最外层是一个Map，结构为：`Map<String, Map<String, Service>>`：

- key：是namespace_id，起到环境隔离的作用。namespace下可以有多个group
- value：又是一个`Map<String, Service>`，代表分组及组内的服务。一个组内可以有多个服务
  - key：代表group分组，不过作为key时格式是group_name:service_name
  - value：分组下的某一个服务，例如userservice，用户服务。类型为`Service`，内部也包含一个`Map<String,Cluster>`，一个服务下可以有多个集群
    - key：集群名称
    - value：`Cluster`类型，包含集群的具体信息。一个集群中可能包含多个实例，也就是具体的节点信息，其中包含一个`Set<Instance>`，就是该集群下的实例的集合
      - Instance：实例信息，包含实例的IP、Port、健康状态、权重等等信息

### 服务注册

#### 服务注册大概流程（服务端）

```plain
/nacos/v1/ns/instance
```

服务在向nacos注册时，客户端向nacos发起http resutful请求。先会在nacos服务本地的Map中添加元素，之后也会调用consistencyService进行消息发送，使得多台nacos服务信息一致

同一个service的多个实例进行注册的时候，还会对servcie用synchronized加锁，处理并发情况，并且将旧的instance和新的instance 结合，之后会调用consistencyService进行消息发送（本地+集群）（实现方式都是异步放入任务队列，任务被当成线程处理）

#### 如何避免并发读写冲突（服务端）

1. 注册服务实例 addInstance 接口时，同一个service的多个实例进行注册的时候，还会对servcie用synchronized加锁，处理addInstance 并发情况
2. 任务队列的消费者是个单线程的线程池
3. Nacos在更新实例列表时，会采用CopyOnWrite技术，首先将旧的实例列表拷贝一份，然后更新拷贝的实例列表，再用更新后的实例列表来覆盖旧的实例列表。

4. 读取实例列表的时候获得的是旧的实例列表



#### 服务注册（客户端）

在客户端 引入 nacos客户端依赖 

```
<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
```

在此依赖下的自动装配里的NacosServiceRegistryAutoConfiguration中的NacosAutoServiceRegistration 类，实现了ApplicationListener<WebServerInitializedEvent>接口(此接口在springboot项目启动过程中会监听发送的事件)

1. 监听WebServerInitializedEvent事件，onApplicationEvent，会调用start方法进行注册实例 addInstance (以http请求的方式)

#### 心跳检测(客户端主动)

1. 并且**对临时实例**会进行一次**心跳检测** ，在客户端用ScheduledExecutorService.schedule来发送一次请求，成功后nacos**服务端**会接受请求ScheduledExecutorService.schedule更新实例的心跳时间





#### 心跳检测（服务端主动）

注册服务实例 addInstance 接口时，在init方法会调用 进行ScheduledExecutorService.scheduleWithFixedDelay进行定时检测调用，定时检测服务是否健康，不健康或者被删除



### 服务发现

**客户端** 

在DynamicServerListLoadBalancer 类中 ，通过HostReactor.UpdateTask 定时执行，定时的实现是每次在finally 代码块中执行 代码    executor.schedule(this, Math.min(delayTime << failCount, DEFAULT_DELAY * 60), TimeUnit.MILLISECONDS);



服务a 调用服务b 的时候 也是通过 懒加载的形式 ，会在第一次fegin调用时,使用DynamicServerListLoadBalancer 开启定时任务，每次定时执行都会将实例列表暂存到本地， 之后每次fegin调用时，直接从本地实例列表获取服务b的地址



**服务端**

服务端接受http 心跳请求， 使用udp和客户端进行定时的服务列表消息推送

### 获取配置

客户端会通过 ScheduledExecutorService 定时发起http请求访问 nacos中的配置信息
