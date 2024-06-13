---
title: "Spring_session"
date: 2023-08-25T10:15:37+08:00
draft: trueme个
---







没登录帐号，只是打开页面

10:36分创建   ，1692932760000 代表了 2023-08-25 11:06:00 差了正好半小时

localhost:0>keys *

 \1)  "spring:session:sessions:a10709ca-3f18-46b8-b69b-c0194e116ad9"

 \2)  "spring:session:sessions:expires:a10709ca-3f18-46b8-b69b-c0194e116ad9"

 \3)  "spring:session:expirations:1692932760000"



hash结构

 \1)  "spring:session:sessions:a10709ca-3f18-46b8-b69b-c0194e116ad9"



​            其中 creationTime（创建时间），lastAccessedTime（最后访问时间），maxInactiveInterval（session 失效的间隔时长） 等字段是系统字段，sessionAttr:xx 可能会存在多个键值对，用户存放在 session 中的数据如数存放于此。









string结构



 \2)  "spring:session:sessions:expires:a10709ca-3f18-46b8-b69b-c0194e116ad9"





set结构

 \3)  "spring:session:expirations:1692932760000"

















- DefaultCookieSerializer.writeCookieValue



![](set_cookie.png)

原理简要总结
当请求进来的时候，SessionRepositoryFilter 会先拦截到请求，将 request 和 response 对象转换成 SessionRepositoryRequestWrapper 和 SessionRepositoryResponseWrapper 。后续当第一次调用 request 的getSession方法时，会调用到 SessionRepositoryRequestWrapper 的getSession方法。这个方法是被从写过的，逻辑是先从 request 的属性中查找，如果找不到；再查找一个key值是"SESSION"的 Cookie，通过这个 Cookie 拿到 SessionId 去 Redis 中查找，如果查不到，就直接创建一个RedisSession 对象，同步到 Redis 中。

说的简单点就是：拦截请求，将之前在服务器内存中进行 Session 创建销毁的动作，改成在 Redis 中创建。
-----------------------------------
©著作权归作者所有：来自51CTO博客作者秃了也弱了的原创作品，请联系作者获取转载授权，否则将追究法律责任
spring-session的使用及其原理——分布式session解决方案
https://blog.51cto.com/u_13540373/5853098

