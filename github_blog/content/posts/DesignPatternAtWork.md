---
title: "工作中使用到的设计模式"
date: 2022-11-25T18:30:10+08:00
draft: true
---

## 策略模式(Strategy Pattern)

 SpringBoot 实现策略模式



1.接口





2.反射通过动态传参，指定beanname和 methodname



优点：解耦，如果又新增了一种模式，只需要添加模式本身的代码，不需要改动调用模式的方法。 

**优点：** 1、算法可以自由切换。 2、避免使用多重条件判断。 3、扩展性良好。

**缺点：** 1、策略类会增多。 2、所有策略类都需要对外暴露。



## 门面模式(Facade Pattern)

接口 Slf4j 

实现 logback ，log4j

优点：解耦，如果我想从log4j 切换到 logback，不需要改代码，只要改依赖就行

如果需要更换日志系统时，只需要将相关的日志系统的依赖替换即可。这儿选择替换成了

1.1 门面模式（外观模式）
门面模式（Facade Pattern),也称之为外观模式，其核心为：外部与一个子系统的通信必须通过一个统一的外观对象进行，使得子系统更易于使用。门面模式相当于一个统一的接口，不论外部对象是怎么样的，与子系统通信都是用的统一的接口（门面模式）。

1.2 日志门面
日志框架也需要一个统一的接口，与代码整合，这样切换日志框架的时候就可以不修改代码内容。
————————————————
版权声明：本文为CSDN博主「trh_csdn」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/trh_csdn/article/details/127098291

## 工厂模式(Factory Pattern)

spring中 实例化 bean 的 Factory



优点: 解耦，如果bean的创建实例的方法比较复杂，如mybatis 只写了接口，靠工厂类动态代理生成bean的实例，可以先实现一个bean Factory，然后在其中修改，方便解耦。







## 观察者模式(Observer Pattern)

springboot启动时，SpringApplicationRunListeners会发布事件



优点: 

```
// 事件解耦例子
@Configuration
public class A48_1 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(A48_1.class);
        context.getBean(MyService.class).doBusiness();
        context.close();
    }

    static class MyEvent extends ApplicationEvent {
        public MyEvent(Object source) {
            super(source);
        }
    }

    @Component
    static class MyService {
        private static final Logger log = LoggerFactory.getLogger(MyService.class);
        @Autowired
        private ApplicationEventPublisher publisher; // applicationContext
        public void doBusiness() {
            log.debug("主线业务");
            // 主线业务完成后需要做一些支线业务，下面是问题代码
            publisher.publishEvent(new MyEvent("MyService.doBusiness()"));
        }
    }

//    @Component
    static class SmsApplicationListener implements ApplicationListener<MyEvent> {
        private static final Logger log = LoggerFactory.getLogger(SmsApplicationListener.class);
        @Override
        public void onApplicationEvent(MyEvent event) {
            log.debug("发送短信");
        }
    }

    @Component
    static class EmailApplicationListener implements ApplicationListener<MyEvent> {
        private static final Logger log = LoggerFactory.getLogger(EmailApplicationListener.class);
        @Override
        public void onApplicationEvent(MyEvent event) {
            log.debug("发送邮件");
        }
    }

}
```





## 责任链模式Chain of Responsibility Pattern）

SpringSecurity 过滤器链



优点: 

 3、增强给对象指派职责的灵活性。通过改变链内的成员或者调动它们的次序，允许动态地新增或者删除责任。 

4、增加新的请求处理类很方便。





## 享元模式(Flyweight Pattern)

当需要重用数量有限的同一类对象时

**2.1** **包装类**

在JDK中 Boolean，Byte，Short，Integer，Long，Character 等包装类提供了 valueOf 方法，例如 Long 的

valueOf 会缓存 -128~127 之间的 Long 对象，在这个范围之间会重用对象，大于这个范围，才会新建 Long 对

象：

**2.2 String** **串池**，连接池

**2.3 BigDecimal BigInteger**





## 装饰器模式(Decorator Pattern）

ReentrantLock  用了AQS

优点: 

饰类和被装饰类可以独立发展，不会相互耦合，装饰模式是继承的一个替代模式，装饰模式可以动态扩展一个实现类的功能。

## 单例模式(Singleton Pattern)

```
public class Runtime {
    private static Runtime currentRuntime = new Runtime();

    /**
     * Returns the runtime object associated with the current Java application.
     * Most of the methods of class <code>Runtime</code> are instance
     * methods and must be invoked with respect to the current runtime object.
     *
     * @return  the <code>Runtime</code> object associated with the current
     *          Java application.
     */
    public static Runtime getRuntime() {
        return currentRuntime;
    }
    





private static volatile Console cons = null;
    /**
     * Returns the unique {@link java.io.Console Console} object associated
     * with the current Java virtual machine, if any.
     *
     * @return  The system console, if any, otherwise <tt>null</tt>.
     *
     * @since   1.6
     */
     public static Console console() {
         if (cons == null) {
             synchronized (System.class) {
                 cons = sun.misc.SharedSecrets.getJavaIOAccess().console();
             }
         }
         return cons;
     }
    
    
    
    
    
    
    
    
```
















## 代理模式(Singleton Pattern)

**主要解决：**在直接访问对象时带来的问题，比如说：要访问的对象在远程的机器上。在面向对象系统中，有些对象由于某些原因（比如对象创建开销很大，或者某些操作需要安全控制，或者需要进程外的访问），直接访问会给使用者或者系统结构带来很多麻烦，我们可以在访问此对象时加上一个对此对象的访问层。

aop 

cglib 

 **`从静态代理的代码中可以发现，静态代理的缺点显而易见，那就是当真实类的方法越来越多的时候，这样构建的代理类的代码量是非常大的，所以就引进动态代理.`**
