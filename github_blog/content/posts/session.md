---
title: "Session"
date: 2022-12-30T15:30:56+08:00
draft: true
---

servlet中 HttpSession是在tomcat中实现的。

StandardSession是HttpSession的默认实现`StandardSession implements HttpSession`。

Session默认是存储在`ManagerBase`的 `protected Map<String, Session> sessions = new ConcurrentHashMap<>(); `中（就好像spring的单例map一样)。

  ![](sessionsConcurrentHashMap.png)

Session1

  ![](session1.png)

Session2

  ![](session2.png)

所以说session 是保存在服务端这句话可以直白的理解为：默认是存储在服务端的一个map中，key是sessionId，value是Session。

所以说使用session会有共享问题，是因为多台tomcat默认只保存自己这台接收的session，当请求切换到另一台tomcat时，用户a在tomcat1中登录了，但是第二个请求负载均衡到了tomcat2，tomcat2说用户没有登录，因为tomcat1的session只存在与tomcat1中。

流程：由服务端生成sessionId，set进Cookie并传给浏览器，浏览器带着Cookie:sessionId访问服务端， 服务端去map中取得到当前用户的信息。
