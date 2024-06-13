---
title: "Interview_questions"
date: 2023-08-03T09:32:04+08:00
draft: true
---

### java代码执行流程



  ![](jvm.png)

```
public class Main{
    public static void main(string[]args){
    Student stu=new Student();
    stu.study();
    stu.hashCode();
    stu=null;
    }
}
```

1. java源代码编译成**字节码**

2. 执行 java命令，就会创建java**虚拟机**

   创建了一个main 主**线程**，从jvm虚拟机栈分配**栈内存**,

   方法中的**局部变量，对象引用** 和**方法参数**和也都会存储在线程内栈内存

3. 发现了没有加载过的类，**类加载器**就会从**方法区**加载类**字节码信息**(类里的成员变量，方法，引用的其他类都会在方法区)

4. 在**堆**内存new对象

5. 普通java方法（栈）

   本地方法（hashcode，wait，notify）

6. 线程切换，用程序计数器记录代码执行行数

7. 不再使用 会被 **垃圾回收**

8. **解释器**将字节码翻译成机器码

9. **JIT即时编译器**负责热点代码

每一个类都有自己的常量池，每个方法都有自己的局部变量表



### 类加载执行流程



#### 1.加载

1. 将类的字节码载入方法区，并创建类.class对象

2. 如果此类的父类没有加载，先加载父类

3. 类加载是懒惰的, 首次用到时才加载

   


以下几种方式会导致类的加载：

1. 使用了类.class    (类对象存在于堆)
2. 用类加载器的 loadClass 方法加载类





#### 2.链接（字节码层面的准备工作）

1. 验证-验证类是否符合Class规范，合法性、安全性检查

2. 准备-为static变量分配空间，设置默认值，

   如果是final修饰的基本类型的static变量 会直接赋予代码中配置的值（不会在 3.初始化 中执行），

   如果不是final修饰的static变量 那就是给变量分配对应类型的默认值 

3. 解析-将常量池的符号引用解析为直接引用 （是一步一步解析的，运行的时候哪里用到了某个对象，执行到这一步的时候再去解析）



#### 3.初始化

1. 执行静态代码块与非final静态变量的赋值

   什么是 clinit方法（类的初始化方法）？

   1) ( 非final修饰的静态变量 + final修饰的引用类型静态变量 + 静态代码块） 按照它们在代码中编写的顺序 自上到下写入到 字节码中 clinit方法当中

2. 切始化是赖惰执行



以下几种方式会导致类的初始化：

1. 首次访问静态方法或静态变量(非 final, 或 final的 引用类型)或者静态方法
2. new一个该类的对象时。
3. 调用Class.forName(String className)。
4. 执行Main方法的当前类。
5. new, clone, 反序列化时



### java线程有几种状态



![](os.thread.state.png)

  ![](Thread.State.png)

```
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;

@Slf4j(topic = "c.TestState")
public class TestState {
    public static void main(String[] args) throws IOException {
        Thread t1 = new Thread("t1") {
            @Override
            public void run() {
                log.debug("running...");
            }
        };

        Thread t2 = new Thread("t2") {
            @Override
            public void run() {
                while(true) { // runnable

                }
            }
        };
        t2.start();

        Thread t3 = new Thread("t3") {
            @Override
            public void run() {
                log.debug("running...");
            }
        };
        t3.start();

        Thread t4 = new Thread("t4") {
            @Override
            public void run() {
                synchronized (TestState.class) {
                    try {
                        Thread.sleep(1000000); // timed_waiting
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        t4.start();

        Thread t5 = new Thread("t5") {
            @Override
            public void run() {
                try {
                    t2.join(); // waiting
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        t5.start();

        Thread t6 = new Thread("t6") {
            @Override
            public void run() {
                synchronized (TestState.class) { // blocked
                    try {
                        Thread.sleep(1000000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        t6.start();

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("t1 state {}", t1.getState());
        log.debug("t2 state {}", t2.getState());
        log.debug("t3 state {}", t3.getState());
        log.debug("t4 state {}", t4.getState());
        log.debug("t5 state {}", t5.getState());
        log.debug("t6 state {}", t6.getState());
        System.in.read();
    }
}

```

```
10:03:16.549 c.TestState [t3] - running...
10:03:17.064 c.TestState [main] - t1 state NEW
10:03:17.065 c.TestState [main] - t2 state RUNNABLE
10:03:17.065 c.TestState [main] - t3 state TERMINATED
10:03:17.065 c.TestState [main] - t4 state TIMED_WAITING
10:03:17.065 c.TestState [main] - t5 state WAITING
10:03:17.065 c.TestState [main] - t6 state BLOCKED
```



### 双亲委派

在类加载的过程中，每个类加载器都会先检查是否已经加载了该类，如果已经加载则直接返回，否则会将加载请求委派给父类加载器。

如果所有的父类加载器都无法加载该类，则由当前类加载器自己尝试加载。所以看上去是自顶向下尝试加载

第二次再去加载相同的类，仍然会向上进行委派，如果某个类加载器加载过就会直接返回

**1.保证类加载的安全性**

通过双亲委派机制，让顶层的类加

载器去加载核心类，避免恶意代码

替换JDK中的核心类库，比如

java.lang.String，确保核心类

库的完整性和安全性。

**2.避免重复加载**

双亲委派机制可以避免同一个类被

多次加载，上层的类加载器如果加

载过类，就会直接返回该类，避免

重复加载。



### ClassLoader的原理

先来分析ClassLoader的原理，ClassLoader中包含了4个核心方法。

双亲委派机制的核心代码就位于loadClass方法中

  ![](ClassLoader.png)

  ![](ClassLoader_parent.png)







### 字节码 i=i++和i=++i 的问题



```
public class Demo1 {
    public static void main(String[] args) {
        int i=0;
        i = i++;

        //输出0
        System.out.println(i);
    }
}
```

```
public class Demo1_1 {
    public static void main(String[] args) {
        int i=0;
        i=++i;

        //输出1
        System.out.println(i);
    }
}
```

  ![](i++&++i.png)



### 方法区有哪几部分

运行时常量池（从.class文件中的常量池而来），串池，直接内存



### 栈内存有哪几部分

局部变量表，操作数栈

### clint 和 不包括init方法






### AOP有几种实现

有三种aop的实现

1. aspectj实现 是在 编译时产生修改原来的类

2. agent实现 是在 类加载时产生修改原来的类

3. proxy实现 是 动态生成了一个新的代理类，前两种没有生成新的类

   普通类是通过 A.java->A.class->类加载

   proxy代理是  没有.java的第一步   A$proxy.class->类加载









**JDK实现**

```
public class AopJdkProxyTest {


    /**
     * jdk的动态代理，只能针对接口代理
     * 总结：代理对象和目标对象是兄弟关系，都继承了Foo接口，代理对象类型不能强转成目标对象类型。
     *
     * jdk的代理类可以是final类型
     */
    @Test
    public void testJdkProxy() throws IOException {
        // 目标对象
        Target target = new Target();
        // 用来加载在运行期间动态生成的字节码
        ClassLoader loader = AopJdkProxyTest.class.getClassLoader();

        // 参数1: 类加载器加载运行时期生成的字节码
        // 参数2: 代理类要实现哪个接口，是一个数组
        // 参数3：InvocationHandler类，具体的代理类方法的实现
        // 代理类是实现Foo接口的，所以可以返回接口类型
        Foo fooProxy = (Foo) Proxy.newProxyInstance(

                loader,

                new Class[]{Foo.class},

                // 参数1: 代理对象自己
                // 参数2：正在执行的方法对象
                // 参数3：正在执行的方法参数
                (proxy, method, args) -> {
                    System.out.println("before...");
                    //这个方法在哪个目标对象上被执行+参数 。反射调用
                    Object result = method.invoke(target, args);
                    System.out.println("after...");
                    return result;  // 让代理也返回目标方法执行的结果
                });

        System.out.println(fooProxy.getClass());

        fooProxy.foo();

        System.in.read();

    }
}

interface Foo {
    void foo();
}

@Slf4j
final class Target implements Foo {
    @Override
    public void foo() {
        log.debug("target foo");
    }
}
```

1. jdk的目标类可以是final类型
2. jdk的动态代理，只能针对接口代理，目标类必须实现接口
3. jdk 代理类 和 目标类是 兄弟关系
4. Object result = method.invoke(target, args); 反射方法调用  
5. 原理是asm，在运行期间动态生成字节码，用java代码生成一个类，主要是有一个ClassWriter
6. 反射在多次调用后，实现类会有优化  java本地代码->第17次调用变成正常调用 类.静态方法  （代价是生成一个新的代理类.） 
6. 一个方法调用就生成一个代理





**cglib实现**

1. cglib的目标类不可以是final类型(如果是final 会报错)，方法也不能用final修饰

2. cglib 代理类是子类型 ，目标类是父类型

3. Object result = proxy.invoke(target, args); // 避免反射效率高，需要传目标类 （spring用的是这种）

4. cglib 和jdk 的区别

   cglib一上来就生成代理类，一个类的所有方法代理就只会生成两个代理类  

   主要是在  MethodProxy  不经过反射调用方法  save1Proxy = MethodProxy.create(Target.class, Proxy.class, "(I)V", "save", "saveSuper");

   下面两个 是用了   FastClass 实现无反射 
    return   methodProxy.invoke( target,p);  //无反射调用，需要代理类
    return methodProxy.invoke(target, args);  //无反射调用，需要Target类  ，args是方法参数

   用一句话来说 就是 cglib会当调用 MethodProxy 的 invoke 或 invokeSuper 方法时, 会动态生成两个类，期中一个方式叫getindex，里面记录了目标方法且与下标做映射，另一个就是   public Object invoke(int index, Object proxy, Object[] args)  根据方法编号, 正常调用目标对象方法   。

   //           在调用的时候调用了   这个 invoke方法避免了反射

   //              return   methodProxy.invoke( target,p);  //无反射调用，需要代理类
                   return methodProxy.invoke(target, args);  //无反射调用，需要Target类  ，args是方法参数

   



```
        Proxy proxy = new Proxy();

        Target target = new Target();

        proxy.setMethodInterceptor(new MethodInterceptor() {
            @Override
            public Object intercept(Object p, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {

                System.out.println("before");
                
                
   
   
   
//                return  method.invoke(target,args);   //反射调用


//              下面两个 是用了   FastClass 实现无反射 
//              return   methodProxy.invoke( target,p);  //无反射调用，需要代理类
                return methodProxy.invoke(target, args);  //无反射调用，需要Target类  ，args是方法参数
            }
        });


        proxy.save();
        proxy.save(1);
        proxy.save(2L);
```







 **spring选择代理**

proxyTargetclass默认false

a,proxyTargetclass=false,月标实现∫接口，用jdk实现
b.proxyTargetClass=false,目标没有实现接I,用cglib实现
c.proxyTargetclass=trUe,总是使用cglib实跑




### 代理创建时机

#### 代码参考

**org.springframework.aop.framework.autoproxy.A17_1**

#### 收获💡

1. 代理的创建时机
   * 初始化之后 (无循环依赖时)  AspectJAwareAdvisorAutoProxyCreator.postProcessAfterInitialization. wrapIfNecessary( )
   * 实例创建后, 依赖注入前 (有循环依赖时), 并暂存于二级缓存









### 线上接口响应慢怎么排查

1. 看日志，检查监控，分布式服务一般都是服务治理的，skywalking可以看整个调用链的时间图。普罗米修斯可以检查内存，cpu使用情况
1. 如果是数据库，redis访问慢，先检查下是不是网络慢
1. 检查代码 ，是否有锁竞争，io 操作
2. 确认JVM配置
2.  对dump的堆内存文件进行分析, 有没有大对象
2. 如果是数据库访问慢，那么就要看是不是有锁表的情况、是不是有全表扫描、索引加得是否合适、是否有JOIN操作、需不需要加缓存，数据库连接有没有用完
2. rocketmq 里有没有消息堆积
2. 通过jstack工具和其他指令，来定位CPU飙升问题
2. arths命令 trace 和 火焰图

  ![](oom.png)

  ![](线上问题排查.png)





### Oauth2

  OAuth 是一个开放标准，该标准允许用户让第三方应用访问该用户在某一网站上存储的私密资源 （如头像、照片、视频等），并且在这个过程中无须将用户名和密码提供给第三方应用。通过令牌 （token） 可以实现这一功能。每一个令牌授权一个特定的网站在特定的时间段内允许可访问特定的资源。OAuth 让用户可以授权第三方网站灵活访问它们存储在另外一些资源服务器上的特定信息，而非所有的内容。对于用户而言，我们在互联网应用中最常见的 OAuth 应用就是各种第三方登录，例如 QQ授权登录、微信授权登录、微博授权登录、GitHub 授权登录等。









### GC调优



1. 是需要 低延迟 还是 高吞吐量的 垃圾回收器？

1. 代码问题：是否数据返回量过大？，可以进行对象瘦身 Integer-int，是否有内存泄漏，threadlocal

2. 先从新生代调优（因为新生代是用复制算法 速度快，minor gc 远远小于full gc）

   新生代太小会频繁发生minor gc， 新生代调大，老年代相应的减少会发生fullgc。官方建议新生代大小控制在整个堆的 25%~40% 之间.

   新生代能容纳所有【并发量 * (请求-响应)】的数据

   幸存区大到能保留【当前活跃对象+需要晋升对象】

   晋升阈值配置得当，让长时间存活对象尽快晋升

   -XX:MaxTenuringThreshold=threshold
   
   -XX:+PrintTenuringDistribution
   
3. 老年代调优

   以 CMS 为例

   CMS 的老年代内存越大越好

   先尝试不做调优，如果没有 Full GC 那么已经...，否则先尝试调优新生代

   观察发生 Full GC 时老年代内存占用，将老年代内存预设调大 1/4 ~ 1/3

   -XX:CMSInitiatingOccupancyFraction=percent  一般百分之75



新生代调大了，每次回收的对象不是变多了吗？

新生代能存活下来的对象很少，即使发生回收速度也很快





案例1 Full GC 和 Minor GC频繁

增大了新生代的内存导致minorgc更少出发，并且survivor区增大，就不会让本不是生命周期那么长的对象进入老年区，从而给老年区节省空间，进一步就减少了老年区出发fullGC

案例2 请求高峰期发生 Full GC，单次暂停时间特别长 （CMS）  

cms重新标记的时间较长，使用【-XX:CMSScavengeBeforeRemark】参数可以做到在重新标记前先执行一次新生代GC。

案例3 老年代充裕情况下，发生 Full GC （CMS jdk1.7）

### redis主从同步

  ![](redis_replica_full.png)



  ![](redis_replica_increment.png)

### redis脑裂


脑裂，主从集群中，同时有两个主能接收写请求。Redis主从切换过程中，若发生脑裂，客户端数据就会写入原主，若原主被降为从库，这些新写入数据就丢了。

脑裂主要是因为原主库发生了假故障，假故障的原因：

- 和主库部署在同一台服务器上的其他程序临时占用了大量资源（例如CPU资源），导致主库资源使用受限，短时间内无法响应心跳。其它程序不再使用资源时，主库又恢复正常
- 主库自身遇到阻塞，如处理bigkey或是发生内存swap（你可以复习下第19讲中总结的导致实例阻塞的原因），短时间内无法响应心跳，等主库阻塞解除后，又恢复正常的请求处理了。

应对脑裂，你可以在主从集群部署时，通过合理地配置参数min-slaves-to-write和min-slaves-max-lag，来预防脑裂。

在实际应用中，可能会因为网络暂时拥塞导致从库暂时和主库的ACK消息超时。在这种情况下，并不是主库假故障，我们也不用禁止主库接收请求。

假设从库有K个，可将：

- min-slaves-to-write设置为K/2+1（如果K等于1，就设为1）
- min-slaves-max-lag设置为十几秒（例如10～20s）

### synchronized

偏向锁出现在（已过时）：
只有一个线程加锁的情况下，此时一定没有竞争！没有竞争！没有竞争！
加锁后markword里存储此线程地址（操作系统线程）

轻量级锁出现在：
有两（多）个线程都试图对此对象交替加锁色，没有竞争
加锁后markword里存储加锁线程地址（java线程）

  ![](synchronized_thread_0.png)

  ![](synchronized_thread_1.png)

重量级锁出现在：
有两（多）个线程都试图对此对象加锁酒，并且发生了竞争
加锁后markword里存储Monitor地址

![](synchronized_monitor.png)

自旋出现在重量级锁之后：

只要发生竞争，轻量级锁就会升级为重量级锁
升级（膨胀）成重量锁之后，未获得锁的线程，会进入_cXq
链表，自旋尝试获取锁
~如果自旋期间，能获取锁，则避免阻塞
如果自旋若干次，仍不能获取锁，则进入阻塞，将来由持锁线程唤醒





### java对象头 MarkWord

无论是加了轻锁还是重锁，前面的内容就会被替换掉，什么hash码，分代年龄都会被替换掉。

  ![](markword.png)



对象分为

对象头 object header 和  对象体 object body





object body =成员变量

object header=Mark Word+Klass Word （类型指针也就是类对象）

Mark Word =hashcode+分代年龄（age）+是不是偏向锁（biased_lock）+加锁状态位

加锁状态位 

01  : 无锁

00 : 轻量锁

10  : 重量级锁

Mark Word 中轻量锁 和 重量级锁存的东西是不一样的。

轻量级：lockRecord 对象

重量级：monitor对象



### 查看日志命令



less app.properties | grep comp -C 5

  ![](log.png)






### 该怎么对比选择MQ

1. 如果我们业务只是收发消息这种单一类型的需求，而且可以允许小部分数据丢失的可能性，但是又要求极高的吞吐量和高性能的话，就直接选Kafka就行了，就好比我们公司想要收集和传输用户行为日志以及其他相关日志的处理，就选用的Kafka中间件。
2. 如果自己所处公司业务比较平稳，未来几年内不会出现飞速发展，而且没有什么改源码的特殊需求的话，在面对选择MQ的时候就可以选用RabbitMQ。毕竟如今这样的中小公司也就是这么干的。
3. 如果自己所处公司发展迅猛，一年经常搞一些特别大的促销秒杀活动，公司技术栈主要是Java语言的话，就直接一步到位选择RocketMQ，这样会省很多事情。

Kafka：追求高吞吐量，一开始的目的就是用于日志收集和传输，**适合产生大量数据的互联网服务的数据收集业务**，大型公司建议可以选用，**如果有日志采集功能，肯定是首选 kafka。**

RocketMQ：**天生为金融互联网领域而生，对于可靠性要求很高的场景**，尤其是电商里面的订单扣款，以及业务削峰，在大量交易涌入时，后端可能无法及时处理的情况。RoketMQ 在稳定性上可能更值得信赖，这些业务场景在阿里双 11 已经经历了多次考验，**如果你的业务有上述并发场景，建议可以选择 RocketMQ。**

RabbitMQ：结合 erlang 语言本身的并发优势，性能较好，社区活跃度也比较高，但是不利于做二次开发和维护，不过 RabbitMQ 的社区十分活跃，可以解决开发过程中遇到的 bug。**如果你的数据量没有那么大，小公司优先选择功能比较完备的 RabbitMQ。**



作者：楼仔
链接：https://juejin.cn/post/7096095180536676365
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



https://www.cnblogs.com/lyhsblog/p/16113202.html

### Elasticsearch的特点/优点 为什么比MySQL快？

Elasticsearch会对所有输入的文本进行处理，建立索引放入内存中，从而提高搜索效率。在这一点上ES要优于MySQL的B+树的结构，MySQL需要将索引放入磁盘，每次读取需要先从磁盘读取索引然后寻找对应的数据节点，但是ES能够直接在内存中就找到目标文档对应的大致位置，最大化提高效率。 并且在进行组合查询的时候MySQL的劣势更加明显，它不支持复杂的组合查询比如聚合操作，即使要组合查询也要事先建好索引，但是ES就可以完成这种复杂的操作，默认每个字段都是有索引的，在查询的时候可以各种互相组合。

对比： 1）基于分词后的全文检索：例如select * from test where name like '%张三%'，对于mysql来说，因为索引失效，会进行全表检索；对es而言分词后，每个字都可以利用FST高速找到倒排索引的位置，并迅速获取文档id列表，大大的提升了性能，减少了磁盘IO。 2）精确检索：进行精确检索，有些时候可能mysql要快一些，当mysql的非聚合索引引用上了聚合索引，无需回表，则速度上可能更快；es还是通过FST找到倒排索引的位置比获取文档id列表，再根据文档id获取文档并根据相关度进行排序。但是es还有个优势，就是es即天然的分布式能够在大量数据搜索时可以通过分片降低检索规模，并且可以通过并行检索提升效率，用filter时，更是可以直接跳过检索直接走缓存。

https://www.51cto.com/article/646093.html

作者：锥栗
链接：https://juejin.cn/post/6998411731210879012
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。















### 为什么说MySQL单表行数不要超过2000w?

1. Mysql 的表数据是以页的形式存放的，页在磁盘中不一定是连续的。
2. 页的空间是 16K, 并不是所有的空间都是用来存放数据的，会有一些固定的信息，如，页头，页尾，页码，校验码等等。
3. 在 B+ 树中，叶子节点和非叶子节点的数据结构是一样的，区别在于，叶子节点存放的是实际的行数据，而非叶子节点存放的是主键和页号。
4. 索引结构不会影响单表最大行数，2kw 也只是推荐值，超过了这个值可能会导致 B + 树层级更高，影响查询性能



### ConcurrentHashMap

扩容



#### put方法

如果table都没初始化，会调用initTable() CAS实现

如果桶下标头节点是空，会用cas添加头元素

如果发现当前桶下标代表扩容，就会去帮助扩容 如果是迁移之前的下标区间，可以直接put，如果是数组下标正在迁移只能阻塞，如果访问了处理完成的链表可以帮忙扩容







如果通下标头节点不是空, 用synchronized关键词加锁 ，只对当前链表的头节点加锁，key相同覆盖，没有找到追加到链表最后，如果是红黑树按照红黑树的添加逻辑





#### transfer方法

元素个数>=3/4的capacity时就会扩容

capacity  并不是数组的长度，如果capacity  是16，factor  =3/4 .则是 数组长度是32，因为32的3/4是24 ，16小于24，所以capacity  =16 ， 数组长度32满足

factor  扩容因子 默认0.75，只是第一次构造中会用到，以后都是按照3/4扩容，如果factor不是3/4的话



### 项目亮点

1.引入了rokcetmq，将复杂的业务任务步骤解耦，也将一些耗时耗cpu内存的步骤独立到某一个topic方法后续节点扩展

2.引入了异步线程池去处理任务，

3.优化数据库慢sql

4.开了一个日志模块的starter组件给其他的项目中使用

5，与其他部门接口进行api联调测试

6，对一些视频文件预处理进行了调研，选择了jave2降低了 ffmepg的 部署复杂度





