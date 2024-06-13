---
title: "学习SpringSecurity"
date: 2022-11-03T14:22:04+08:00
draft: true
weight: 20
---

## SpringSecurity概览

在日常工作中，web应用的用户登录，用户认证，用户授权等都是常用到的功能，而与Spring相结合的SpringSecurity就是一个可高度定制化的安全框架。

{{< admonition info >}}

Spring Security is a powerful and highly customizable authentication and access-control framework. It is the de-facto standard for securing Spring-based applications.

Spring Security is a framework that focuses on providing both authentication and authorization to Java applications. Like all Spring projects, the real power of Spring Security is found in how easily it can be extended to meet custom requirements

{{< /admonition >}}

https://spring.io/projects/spring-security

https://spring.io/guides/topicals/spring-security-architecture

英语水平好的最好还是通读官网文档最好！

------

## 第一个SpringSecurity应用

https://spring.io/guides/gs/securing-web/

打开网址，可以直接clone一个官方的Springboot+SpringSecurity 简单应用，也可以选择仓库中之前released 版本下载

注意：在SpringSecurity 5.7版本中，SpringSecurity已经将需要继承WebSecurityConfigurerAdapter的方式改成了SecurityFilterChain 

 https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.7-Release-Notes

https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter



这是我自己学习用的项目：

https://github.com/shengtu0328/xrq-spring-security

------

## 认证&授权
{{< admonition info >}}
Application security <u>boils down to(归结为）</u> two more or less independent problems: authentication (who are you?) and authorization (what are you allowed to do?). Sometimes people say “access control” instead of "authorization", which can get confusing, but it can be helpful to think of it that way because “authorization” is overloaded in other places. Spring Security has an architecture that is designed to separate authentication from authorization and has strategies and extension points for both.{{< /admonition >}}

-----
##  一些重要的对象
{{< admonition tip >}}
先对每个对象有所了解，之后再看源码就会清晰很多
{{< /admonition >}}


| 用户对象           |                                                              |
| ------------------ | ------------------------------------------------------------ |
| UserDetails        | 描述SpringSecurity中得的用户                                 |
| GrantedAuthority   | 用户拥有的权限                                               |
| UserDetailsService | **对用户的查询Service，loadUserByUsername方法应该只查询用户存不存在，不做密码正确与否的校验,但是不通过一般会抛出异常。到ProviderManager 中才会进行密码正确的校验 如DaoAuthenticationProvider.additionalAuthenticationChecks()** |
|                    |                                                              |


| 认证对象                              |                                                              |
| ------------------------------------- | ------------------------------------------------------------ |
| Authentication                        | 认证的请求信息 也是认证结果信息                              |
| Authentication.principal              | 用户名 比如xrq                                               |
| Authentication.credentials            | 用户凭证 比如密码 123456                                     |
| AuthenticationManager,ProviderManager | 处理认证逻辑的对象                                           |
| SecurityContextHolder.getContext()    | 默认是包装了一个ThreadLocal实现的，通过它在FilterChain中多个Filter中获取到当前请求里认证信息，它很重要。就好像失败总是贯穿人生始终！ |
|                                       |                                                              |

https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authentication





The [`Authentication`](https://docs.spring.io/spring-security/site/docs/5.7.5/api/org/springframework/security/core/Authentication.html) serves two main purposes within Spring Security:

- An input to [`AuthenticationManager`](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-authenticationmanager) to provide the credentials a user has provided to authenticate. When used in this scenario, `isAuthenticated()` returns `false`.
- Represents the currently authenticated user. The current `Authentication` can be obtained from the [SecurityContext](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-securitycontext).

The `Authentication` contains:

- `principal` - identifies the user. When authenticating with a username/password this is often an instance of [`UserDetails`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/user-details.html#servlet-authentication-userdetails).
- `credentials` - often a password. In many cases this will be cleared after the user is authenticated to ensure it is not leaked.
- `authorities` - the [`GrantedAuthority`s](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-granted-authority) are high level permissions the user is granted. A few examples are roles or scopes.



## 过滤器链

![securityfilterchain](https://docs.spring.io/spring-security/reference/_images/servlet/architecture/securityfilterchain.png)

有两个过滤器链，ApplicationFilterChain内嵌套了FilterChainProxy$VirtualFilterChain

ApplicationFilterChain 过滤器链

![](ApplicationFilterChain.png) 

FilterChainProxy$VirtualFilterChain 过滤器链 （也就是 SpringSecurityFilterChain）

![](FilterChainProxy$VirtualFilterChain.png) 

https://docs.spring.io/spring-security/reference/servlet/architecture.html



------

## SecurityContextPersistenceFilter

SecurityContextPersistenceFilter 主要的功能如果登录的filter成功了 会将Authentication放入 session中。以便在登录成功以后，下次访问其他接口，从session 中取出，保持登录状态。 并且每次请求过后都会清空 threadloca即SecurityContextHolder.clearContext();



每次请求处理是有状态还是无状态是通过SecurityContextPersistenceFilter实现的，在SecurityContextPersistenceFilter代码中执行chain.doFilter(holder.getRequest(), holder.getResponse())前，中配置不同的private SecurityContextRepository repo，会执行this.repo.loadContext(holder)方法，是否在此次请求中加载authentication

| SessionCreationPolicy           | SecurityContextRepository repo       |
| ------------------------------- | ------------------------------------ |
| SessionCreationPolicy.STATELESS | NullSecurityContextRepository        |
| 默认值                          | HttpSessionSecurityContextRepository |



```
#SecurityContextPersistenceFilter
private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
      throws IOException, ServletException {
   // ensure that filter is only applied once per request
   if (request.getAttribute(FILTER_APPLIED) != null) {
      chain.doFilter(request, response);
      return;
   }
   request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
   if (this.forceEagerSessionCreation) {
      HttpSession session = request.getSession();
      if (this.logger.isDebugEnabled() && session.isNew()) {
         this.logger.debug(LogMessage.format("Created session %s eagerly", session.getId()));
      }
   }
   HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request, response);
   SecurityContext contextBeforeChainExecution = this.repo.loadContext(holder);
```
------

## ThreadLocalSecurityContextHolderStrategy

默认策略是把当前登录信息存在 threadlocal里，

![](ThreadLocalSecurityContextHolderStrategy.png) 

并且在外层的 SecurityContextPersistenceFilter里 ，会及时手动清理threadlocal 对象



![](SecurityContextPersistenceFilter_ThreadLocal.png) 



------
##  Session和Cookie



servlet中 HttpSession是在tomcat中实现的。生成sessionId，并将sessionId set进cookie，用来区分不同的用户请求。

![](doGetSession.png) 




------
##  LogoutFilter

处理注销操作的LogoutFilter 只会处理注销请求，其余放行

------
##  FormLogin认证模式

FormLogin是Spring Security默认的认证模式。

有状态的，有session的概念。第一此登录成功后，服务端就将用户信息放入session中，并将sessionId给客户端。之后的校验不需要再查数据库，但是占用服务器内存。默认如果多台服务器的话，那多台服务器之间是没有共享session的，所以是有状态的。不需要查库，省去了大量重复校验，性能比Httpbasic要好。

所以说**UsernamePasswordAuthenticationFilter**中，每次请求时并不会访问到此过滤器，会有**SecurityContextPersistenceFilter**先从session里面取。



{{< admonition >}}
UsernamePasswordAuthenticationFilter 只会处理登录接口的请求，其他的一律放行。因此它的作用还可以让你少写一个登录Controller方法
{{< /admonition >}}

![](FormLogin.png)




![loginurlauthenticationentrypoint](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/unpwd/loginurlauthenticationentrypoint.png)

![usernamepasswordauthenticationfilter](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/unpwd/usernamepasswordauthenticationfilter.png)

https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html

------
##  HttpBasic认证模式

  Authorization: Basic eHJxOjEyMzQ1Ng==

  Basic 后面跟的是用户名:密码的 base64 如 xrq:123456

 HttpBasic是无状态的，不使用Cookie，也没有会话和注销的概念。每次请求时都要在Request Header加上   Authorization: Basic eHJxOjEyMzQ1Ng==。服务端每次接收的请求都要去库里校验。

所以说**BasicAuthenticationFilter**中的doFilterInternal()方法在每次请求中都是会被调用。





![basicauthenticationfilter](https://docs.spring.io/spring-security/reference/_images/servlet/authentication/unpwd/basicauthenticationfilter.png)

https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/basic.html

------
## 自定义Filter

自定义个filter 比如jwtFilter

{{< admonition >}}

只是说加在UsernamePasswordAuthenticationFilter.class 位置前面一个加一个jwtAuthenticationTokenFilter，并没有添加UsernamePasswordAuthenticationFilter.class。

{{< /admonition >}}

```
http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
```



如果当前request 是登陆状态的 ，则在FilterSecurityInterceptor 之前的某个 filter就应该将setAuthentication 好


------
## FilterSecurityInterceptor

https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-requests.html#servlet-authorization-filtersecurityinterceptor



总而言之：认证和授权这两部（authentication (who are you?) and authorization (what are you allowed to do?) ）

都是在FilterSecurityInterceptor 中完成，需要两个参数

**Authentication**（当前登录用户的信息,包括用户名，拥有的权限等等）  和 **当前请求的url需要的authorities**

|      | 判断条件                                         |
| ---- | ------------------------------------------------ |
| 认证 | Authentication 是否为空                          |
| 授权 | Authentication 是否包含当前请求需要的authorities |

如果在AccessDecisionManager没有抛出异常的话，就继续doFilter()



{{< admonition >}}

注意：在SpringSecurity 5.7版本中 

`FilterSecurityInterceptor` is in the process of being replaced by [`AuthorizationFilter`](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html). Consider using that instead.

{{< /admonition >}}

------
## AuthorizationFilter

`FilterSecurityInterceptor` is in the process of being replaced by [`AuthorizationFilter`](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html). Consider using that instead.





------



## FilterSecurityInterceptor 中的AccessDecisionManager

{{< admonition info >}}
It has pretty simple flow. It gets authentication object which has role information for user, security metadata which has required role for certain url pattern, and requested url.
{{< /admonition >}}









AccessDecisionManager 接收三个参数：

Authentication authentication：用户的认证信息，认证成功的话是应该会有这个用户所拥有的权限

Object object： 当前请求访问的url地址

Collection<ConfigAttribute> configAttributes：当前这个url地址所需要的权限

https://backendstory.com/spring-security-authorization-mechanism/



------

### 请求/d 

```
.antMatchers("/d").hasRole("roled")
```

FilterSecurityInterceptor 实现

![](AccessDecisionManager2.png) 

------

### 请求/g

```
@PreAuthorize("hasAuthority('authorityg')")
@RequestMapping("/g")
public String g(){
    return "g";
}
```

已经追踪源码发现是执行完全部的Filter Chain，进入 DispatcherServlet 类后，最终通过MethodSecurityInterceptor 这个拦截器处理的，

![](AccessDecisionManager.png) 



FilterSecurityInterceptor 和  MethodSecurityInterceptor都实现了AbstractSecurityInterceptor，AbstractSecurityInterceptor其实也设计了只执行一次的过滤逻辑





------



## ExceptionTranslationFilter



![](ExceptionTranslationFilter.png) 



ExceptionTranslationFilter作用：

FilterSecurityInterceptor 是处理 do认证授权的地方

处理 FilterSecurityInterceptor 抛出的异常；判断是认证异常，还是授权异常，分别进行处理。，也就是对 FilterSecurityInterceptor 认证授权的结果  进行最终处理.



```
private void handleSpringSecurityException(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain, RuntimeException exception) throws IOException, ServletException {
   if (exception instanceof AuthenticationException) {
      handleAuthenticationException(request, response, chain, (AuthenticationException) exception);
   }
   else if (exception instanceof AccessDeniedException) {
      handleAccessDeniedException(request, response, chain, (AccessDeniedException) exception);
   }
}
```



![exceptiontranslationfilter](https://docs.spring.io/spring-security/reference/_images/servlet/architecture/exceptiontranslationfilter.png)

https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-security-filters





------


## 流程总结

|                                              | 作用                                                         |
| -------------------------------------------- | ------------------------------------------------------------ |
| SecurityContext中setAuthentication()的Filter | 主要是认证成功后，完成SecurityContext中setAuthentication()，如BasicAuthenticationFilter |
| FilterSecurityInterceptor                    | 对SecurityContext中getAuthentication()，也就是从ThreadLoacl取出当前请求的用户认证结果进行认证和授权的验证。 |

用几句话概括SpringSecurity

SpringSecurity为我们项目中添加了一个过滤器链。在完成认证这一步的Filter里（根据request的参数去UserDetail验证)，此过滤器一般是过滤器链中间位置，

如果认证通过，就应该在SecurityContext中setAuthentication(); 然后调用chain.doFilter(request, response);

如果认证失败，可以直接返回失败结果，不继续或者继续执行chain.doFilter(request, response)。参考BasicAuthenticationFilter的实现

在过滤器的最后一个FilterSecurityInterceptor最终都会进行Authentication认证和授权校验。没有通过的最终都会被拦截。

------








## 过滤器执行顺序


有一点需要说明，虽然说SpringSecurity 中的过滤器链是按照过滤器链中的顺序依次执行doFilter()，但并不是说 执行完 Filter1 doFilter()全部代码后，再执行Filter2。

就如 SecurityContextPersistenceFilter中doFilter()

![](SecurityContextPersistenceFilter.png) 





------



## @PreAuthorize("hasAuthority('authorityg')") 是怎么处理的？

猜测：

是在服务启动时将所有带有@PreAuthorize 注解的方法收集好，然后在SpringSecutiyFilterChain里完成权限的认证？

还是直接在借口调用的时候，调用具体的资源方法时前进行拦截？



结果：

追踪源码发现是执行完全部的Filter Chain，进入 DispatcherServlet 类后，最终通过MethodSecurityInterceptor 这个拦截器处理的.

这个请求也会正常通过FilterSecurityInterceptor. 的accessDecisionManager 方法，但是如果是普通资源，默认是需要登陆，所以说正常登录的用户可以正常通过这层判断。之后再走到MethodSecurityInterceptor。



![](FilterSecurityInterceptor.png)



![](MethodSecurityInterceptor.png)





------



## PermissionEvaluator 是什么？

![](PermissionEvaluator.png)

PermissionEvaluator 还是在MethodSecurityInterceptor 中里被调用的。

 hasPermission()和   hasAuthority()相比较 ， hasPermission()是可以自定义实现判断规则， hasAuthority()只能用默认的实现

根据SecurityExpressionRoot 中

![](SecurityExpressionRoot.png)



![](SecurityExpressionRoot2.png)



------



## 如果UsernamePasswordAuthenticationFilter的UserDetailsService中验证失败，抛出了自定义异常，是怎么处理的？



UserDetailService.loadUserByUsername 抛异常

![](loadUserByUsername.png)



并且 AuthenticationProvier 也不会处理异常，也只会继续抛异常

![](DaoAuthenticationProvider.png)



![](ProviderManager.png)







直到 UsernamePasswordAuthenticationFilter中catch住，交给AuthenticationFailureHandler 处理。比如AuthenticationFailureHandler会重定向或者其他。

![](BasicAuthenticationFilter.png)









 UsernamePasswordAuthenticationFilter 

doFilter()->attemptAuthentication()->

AuthenticationManager

authenticate()->

AuthenticationProvider

authenticate()->





## 如果在自定义的Filter中，抛出了自定义异常，是怎么处理的？

处理不了。所以不要在自定义filter中抛出异常，如果验证逻辑不通过，不setAuthentication即可，最终会自动交给FilterSecurityInterceptor 判断处理。可以参考BasicAuthenticationFilter的实现。

如果是AuthenticationManager 中 校验用户抛出了异常，是可以在外层filter中处理的

------


## AccessDeniedHandler是在哪里调用的

在ExceptionTranslationFilter 对FilterSecurityInterceptor 进行try catch后，FilterSecurityInterceptor抛出的进行处理，也就是handleSpringSecurityException ()方法







------


## AuthenticationEntryPoint是在哪里调用的

在ExceptionTranslationFilter 对FilterSecurityInterceptor 进行try catch后，FilterSecurityInterceptor抛出的进行处理，也就是handleSpringSecurityException ()->handleAccessDeniedException()->sendStartAuthentication()调用



![](AuthenticationEntryPoint.png) 



------

## 未登录的状态取访问需要登录的资源时，是在哪里处理的

还是在MethodSecurityInterceptor的attemptAuthorization 方法中，会返回 AccessDeniedException ，然后交给

ExceptionTranslationFilter.handleSpringSecurityException处理



## PasswordEncoder

密码在数据库最好不以明文存储，



BCryptPasswordEncoder 是强散列算法对密码进行散列操作

BCrypt 将在内部随机产生盐值，并且为了这种随机盐值能正常工作，BCrypt将盐值存在hash值本身中，例如，以下

```
BCryptPasswordEncoder encoder=new BCryptPasswordEncoder();
System.out.println(encoder.encode("123456"));
System.out.println(encoder.encode("123456"));

$2a$10$TS.YfXaPNbhCFDljBda04u6YQgbL18TBWMgPgkdJb6djKJO9nt9tq
$2a$10$H9Ub8PIfhKaE9mafKrW4s.XlYbzZcU6gdUqZxAHOWnKeo1ML9is0q
```

用$分割三个字段

2a 代表BCrypt 算法版本

10 代表算法强度

H9Ub8PIfhKaE9mafKrW4s. 代表随机产生的盐值，前22个一般是salt盐值，后面剩下的就是密码结合盐值生成的散列内容

总长度60





为什么要加盐？

因为123456加密后固定是abc，所以万一知道了abc明文值是123456的话，那么只要是123456作为密码的，就都被破解了。

但如果有了盐值，每次都随机生成一个字符串，混淆到原始的明密码中，

1. 盐 qwer
2. 明文密码 123456
3. 加了盐的明文密码12q3w345e6r
4. 对加了盐的明文密码加密  jljsahpq
这样就算拿到了 加密密码 jljsahpq就算知道原始对应值12q3w345e6r也没法破解


------








------


## HttpSecurity配置

### Session控制

在HttpBasic认证中，每次请求都带着  Authorization: Basic eHJxOjEyMzQ1Ng==，之前期望httpbasic是无状态的，每次都根据eHJxOjEyMzQ1Ng== base64解码的用户名和密码取库里校验。但实际上SpringSecutiy默认开着从session获取，是有状态的。第一次请求服务端产生了sessionId，第第二次请求客户端会带着sessionId，服务器会从根据sessionId取到session及其中的数据。

通过HttpSecurity配置可以设置问无状态的请求模式

```
http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
```

------

### roles()和authorities()

```
    @Bean
    @Override
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withDefaultPasswordEncoder()
                .username("user")
                .password("password")
                .authorities("authorityg")
                //authorities()和roles()只应该定义一个
                //.roles("admin","rolei")
                .build();

        return new InMemoryUserDetailsManager(user);
    }
```

roles() 和 authorities（）会会覆盖当前用户的 authorities，所以roles()和  authorities（）只能调用一个，一般是用authorities().



### 多个 antMatchers（）执行顺序

```
.authorizeRequests()

//本项目所需要的授权范围,这个scope是写在auth服务的配置里的
.antMatchers("/**").access("#oauth2.hasScope('scope1')")
.antMatchers( "/a").hasAuthority("authorityOauth2")
```

写在前面的会覆盖 写在后面的，如 访问/a 实际需要的权限判断的条件是.antMatchers("/**").access("#oauth2.hasScope('scope1')")







------

## CSRF

https://docs.spring.io/spring-security/reference/features/exploits/csrf.html



- 如果你只是创建一个非浏览器客户端使用的服务,你可能会想要禁用CSRF保护

检查Referer字段

添加校验token

------

## rememberMe

springsecurity 的 rememberMe是在服务端的一种实现，和前端**记住密码**有区别。

**记住密码**是服务端登录成功后，set账密的cookie，然后前端以后每次登录都会根据cookie提交账密。

**记住我**是指一旦用户的会话超时过期，就要再次登录，这样太过于烦琐。记住我可以让用户会话过期之后，还能保持认证状态，就会方便很多。

主要实现步骤

1. UsernamePasswordAuthenticationFilter登录成功会->successfulAuthentication()->this.rememberMeServices.loginSuccess

   生成一个  bas64encode（username+  expiryTime+md5（ username + ":" + tokenExpiryTime + ":" + password + ":" + getKey()））

   set 到cookie里

   ![](makeTokenSignature.png)

2. RememberMeAuthenticationFilter 发现如果session为空，但是session中的   remember-me属性不为空时，就再base64decode，

   在 md5进行验签，成功后SecurityContextHolder.setContext(context);登录成功。

![](RememberMeAuthenticationFilter.png)



------

## 访问/a，提示登录，登陆成功后，自动跳转到a页面是怎么实现的？

HttpSessionRequestCache 实现的 ，在session中存了上一次的request信息

![](HttpSessionRequestCache.png)





## 表单登录，验证码登陆，登陆成功后在哪里执行SecurityContextHolder.getContext().setAuthentication(authResult);

![](SecurityContextHolder.getContext().setAuthentication.png)





## TokenEnhancer配置了后什么时候调用

![](TokenEnhance.png)
