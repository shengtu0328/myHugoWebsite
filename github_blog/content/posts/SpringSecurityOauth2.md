---
title: "学习SpringSecurityOauth2"
date: 2022-11-17T18:04:01+08:00
draft: true
---

## SpringSecurityOauth2几个重要注解

```
#创建一个类 加上这两个注解 ，就实现了认证服务器。不需要自己写一行代码
@Configuration
@EnableAuthorizationServer
```

```
创建一个类 加上这两个注解 ，就实现了资源服务器。不需要自己写一行代码
@Configuration
@EnableResourceServer
```

{{< admonition >}}

认证服务器和资源服务器 在逻辑上是两个概念， 在物理上，可以是一个服务。但在本例中分成两个服务。

{{< /admonition >}}

## SpringSecurityOauth2的过滤器链

SpringSecurityOauth2的AuthorizationServer相比SpringSecurity 默认多了 ClientCredentialsTokenEndpointFilter 

![](FilterChainAuthorizationServer.png)

SpringSecurityOauth2的ResourceServer相比SpringSecurity 默认多了 OAuth2AuthenticationProcessingFilter 

![](FilterChainResourceServer.png)


------

## SpringSecurityOauth2在SpringSecurity基础之上还做了什么

可以用 mvn dependency:tree   命令看到 SpringSecurityOauth2是依赖SpringSecurity的

```
[INFO] +- org.springframework.security.oauth:spring-security-oauth2:jar:2.3.5.RELEASE:compile
[INFO] |  +- org.springframework:spring-beans:jar:5.2.3.RELEASE:compile
[INFO] |  +- org.springframework:spring-context:jar:5.2.3.RELEASE:compile
[INFO] |  +- org.springframework.security:spring-security-core:jar:5.2.1.RELEASE:compile
```

SpringSecurityOauth2 为了实现Oauth2，其实无非也只是在SpringSecurity上基础上加了以下几点

**生成token的请求处理**

1. **BasicAuthenticationFilter或者**添加了**ClientCredentialsTokenEndpointFilter**，通过ClientDetailsUserDetailsService认证请求中的client信息

   ```
   @EnableAuthorizationServer
   public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {  
       @Override
       public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
           //ClientCredentialsTokenEndpointFilter默认不开启，要配置此行才能开启
           security.allowFormAuthenticationForClients();
       }
   }
   ```

2. 添加了TokenEndpoint类，生成token的接口，其中包括校验client信息的逻辑

3. 添加了CheckTokenEndpoint类，/oauth/check_token用于校验token

4. 添加了TokenGranter类，实现Oauth2的 多种grant_type模式，生成token



**访问资源时的请求处理**

- 在SpringSecurityChain里添加了一个**OAuth2AuthenticationProcessingFilter** ,它通过**OAuth2AuthenticationManager**->**ResourceServerTokenServices**来完成request中token的校验功能。(就好像SpringSecurity里BasicAuthenticationFilter处理request中的httpbasic参数一样)
- OAuth2AuthenticationProcessingFilter->OAuth2AuthenticationManager->ResourceServerTokenServices
- 并且因为Oauth2本身是支持分布式的场景，对 **ResourceServerTokenServices**进行了扩展，如**RemoteTokenServices**，可以向AuthorizationServer发送http校验token。









最左侧代表代码所处的位置，**A**代表AuthorizationServer，**R**代表ResourceServer

下面的表格类比了SpringSecurityOauth2 和 SpringSecurity的功能，调用链顺序自上而下的。

|      | SpringSecurityOauth2                                         | 功能                                                         | SpringSecurity                  | 功能                                  |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------- | ------------------------------------- |
|      | **生成token的请求处理 **/oauth/token                         |                                                              |                                 |                                       |
| A    | BasicAuthenticationFilter(主要的)或者ClientCredentialsTokenEndpointFilter | 校验**ClientDetails**信息（两种方式，根据http head参数 或者 表单参数来校验） | 同样是BasicAuthenticationFilter | 校验**Userdetails**                   |
| A    | ClientDetailsUserDetailsService                              | 查询client信息                                               | UserDetailsService              | 查询User信息                          |
|      |                                                              |                                                              |                                 |                                       |
| A    | TokenEndpoint                                                | 处理/oauth/token请求                                         |                                 |                                       |
| A    | TokenGranter                                                 | 具体的生成token逻辑                                          |                                 |                                       |
| A    | AuthorizationCodeTokenGranter                                | 认证码模式                                                   |                                 |                                       |
| A    | RefreshTokenGranter                                          | refresh token                                                |                                 |                                       |
| A    | ImplicitTokenGranter                                         | 简化模式                                                     |                                 |                                       |
| A    | ResourceOwnerPasswordTokenGranter                            | 密码模式                                                     |                                 |                                       |
| A    | DaoAuthenticationProvider                                    | 查询UserDetail                                               |                                 |                                       |
|      |                                                              |                                                              |                                 |                                       |
|      |                                                              |                                                              |                                 |                                       |
|      |                                                              |                                                              |                                 |                                       |
|      | \|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\|\| |                                                              |                                 |                                       |
|      | **访问资源时的请求处理**  /hello                             |                                                              |                                 |                                       |
| R    | OAuth2AuthenticationProcessingFilter                         | 请求ResourceServer的资源时，对请求中token的进行校验filter    | BasicAuthenticationFilter       | 请求资源时，对请求中httpbasic进行校验 |
| R    | OAuth2AuthenticationManager                                  | AuthenticationManager                                        | ProviderManager                 | AuthenticationManager                 |
| R    | ResourceServerTokenServices                                  | 进行rpc调用                                                  | DaoAuthenticationProvider       | 查库进行校验                          |
| R    | restTemplate                                                 | 具体的操作类，会调用AuthorizationServer中/oauth/check_token接口 | UserDetailsService              | 具体的操作类                          |
| A    | CheckTokenEndpoint                                           | AuthorizationServer的/oauth/check_token接口                  |                                 |                                       |

![](TokenGranter.png)



{{< admonition >}}

并不是SpringSecurityOauth2 比 SpringSecurity 多了一个FilterChain，而是在SpringSecurity的FilterChain 替换了一个OAuth2AuthenticationProcessingFilter？



总结下

一. /oauth/token 的做了几件事

1. 校验ClientDetail
2. 校验UserDetail
3. 生成token

二. /hello 的做了几件事

1. 验校token

2. 如果还需要校验权限的话，一般交给SpringSecuity的MethodSecurityInterceptor 来处理

   

   

{{< /admonition >}}







## SpringSecurityOauth2的HttpSecurity和SpringSecurity的HttpSecurity配置问题

首先HttpSecurity的功能是是为项目中的url配置所需的权限。

测试下来 WebSecurityConfigurerAdapter的HttpSecurity任何配置都没有生效，生效的只有ResourceServerConfigurerAdapter的HttpSecurity。





## 一些问题

1.refresh_token 接口调用后，会颁发一个新的access_token,并让旧的access_token失效















## 总结

SpringSecurityOauth2为了实现oauth2，

通过添加 BasicAuthenticationFilter/ClientCredentialsTokenEndpointFilter ,TokenEndPoint等实现了AuthorizationServer。

通过添加 OAuth2AuthenticationProcessingFilter等实现了AuthorizationServer和ResourceServer。

AuthorizationServer实现生成token，check token，refresh token的职责。

用户首先从AuthorizationServer获取token，然后在带着token访问ResourceServer时，会去AuthorizationServer校验token，成功的话会得到当前用户所有信息(包括权限数据)，然后在resourceServer内对@PreAuthorize("hasAnyAuthority('admin')")进行校验。 



与SpringSecurity相同的点都是在FilterChain中先进行认证后进行授权。 
