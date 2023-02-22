---
title: Spring Security相关
date: 2023-02-21 18:10:07
categories:
- Program
tags:
- 笔记
- Spring Security

---

## Spring Security的配置

`Spring Security 5.7`开始实现`WebSecurityConfigurerAdapter`来配置的方式已经被废弃了。可直接在配置类上打@Configuration注解，并直接注入SecurityFilterChain及其他相关的方法。

```java
@Configuration
@EnableWebSecurity	// 添加 security 过滤器
//@EnableGlobalMethodSecurity(prePostEnabled = true)	// 启用方法级别的权限认证
public class SecurityConfig{
    //Spring Security 5.7开始WebSecurityConfigurerAdapter已经被废弃了

    /**
     * 配置安全拦截策略
     * @param http
     * @return
     * @throws Exception
     */
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        //链式配置拦截策略
        http.csrf().disable()//关闭csrg跨域检查
                .authorizeRequests(auth -> auth
                        .antMatchers("/index.html","/css/**","/img/**","/js/**").permitAll() //index.html直接通过
                        .antMatchers("/mobile/**").hasAuthority("mobile") //配置资源权限
                        .antMatchers("/salary/**").hasAuthority("salary")//hasXx方法有很多，有什么权限
                        .antMatchers("/common/**").permitAll() //common下的请求直接通过
                        .anyRequest().authenticated() //其他请求需要登录)
                )//开启请求认证
                .formLogin(login -> {
                            try {
                                login
//                                        .loginPage("/index.html").loginProcessingUrl("/login")//实现自定义的登录页面,页面源码 DefaultLoginPageGeneratingFilter
                                        .usernameParameter("username").passwordParameter("password")//自定义登录参数
                                        .defaultSuccessUrl("/main.html")//可从默认的login页面登录，并且登录后跳转到main.html)
                                        .failureUrl("/common/loginFailed")
                                        .and()
                                        .rememberMe();//开启记住我功能 具体实现RememberMeAuthenticationFilter
                            } catch (Exception e) {
                                throw new RuntimeException(e);
                            }
                        }
                );
        return http.build();
    }
    //配置密码加密方式
//    @Bean
//    public PasswordEncoder passwordEncoder() {
//        return new BCryptPasswordEncoder();
//    }


    //配置内存用户
    @Autowired
    protected void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
//        auth
//                .inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
//                .withUser("worker").password(new BCryptPasswordEncoder().encode("worker")).roles("worker").authorities("mobile","salary")
//                .and()
//                .withUser("admin").password(new BCryptPasswordEncoder().encode("admin")).roles("admin").authorities("mobile","salary");
    }
}
```

## 关于CSRF

`csrf`全称是**Cross—Site Request Forgery** 跨站点请求伪造，这是一种安全攻击手段。利用存在客户端的信息伪造成正常用户来进行攻击。

> 例如你访问网站A登录后，未退出又打开一个tab页访问网站B，这时候网站B就可以利用保存在浏览器中的sessionId伪造成你的身份访问网站 A。

### Spring Security对CSRF的检查机制。

他的思想就是在后台的session中加入一个csrf的token值，向后端发送请求时，对于GET、HEAD、TRACE、OPTIONS以外的请求，例如POST、PUT、DELETE等，会要求带上这个token值进行比对。

我们访问默认的登录页时，可在登录form看到有一个name为csrf的隐藏字段，这个就是csrf的token。

Spring Security后台有一个CsrfFilter专门负责对Csrf参数进行检查。他会调用 HttpSessionCsrfTokenRepository生成一个CsrfToken，并将值保存到Session中。

## 方法级别的注解支持

可直接在Controller中进行权限控制。在Configuration配置类中打`@EnableGlobalMethodSecurity()`注解，参数有`prePostEnabled、securedEnabled、jsr250Enabled`，分别对应`@PreAuthorize、@Secured、@RolesAllowed(等价于@Secured)`，使用时在Controller的方法上打对应注解并配置权限就行。



## 认证流程

1. 用户提交用户名、密码被`SecurityFilterChain`中的`UsernamePasswordAuthenticationFilter` **过滤器**获取到，封装为请求 `Authentication`，通常情况下是`UsernamePasswordAuthenticationToken`这个实 现类。

2. 然后过滤器将`Authentication`提交至**认证管理器**`（AuthenticationManager）` 进行认证。

3. 认证成功后，`AuthenticationManager` **身份管理器**返回一个被填充满了信息的 （包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除） **Authentication 实例**。

4. `SecurityContextHolder` 安全上下文容器将第3步填充了信息的 `Authentication` ，通过`SecurityContextHolder.getContext().setAuthentication()`方法，设置到其中。

   

   可以看出AuthenticationManager接口（认证管理器）是认证相关的核心接 口，也是发起认证的出发点，它的实现类为ProviderManager。而Spring Security 支持多种认证方式，因此ProviderManager维护着一个List 列表，存放多种认证方 式，最终实际的认证工作是由AuthenticationProvider完成的。咱们知道web表单 的对应的AuthenticationProvider实现类为DaoAuthenticationProvider，它的内 部又维护着一个UserDetailsService负责UserDetails的获取。最终 AuthenticationProvider将UserDetails填充至Authentication。

   > 调试代码从`UsernamePasswordAuthenticationFilter` 开始跟踪。 最后的认证流程在`AbstractUserDetailsAuthenticationProvider`的 `authenticate`方法中。获取用户在`retrieveUser`方法。密码比较在 `additionalAuthenticationChecks`方法

### AuthenticationProvider接口：认证处理器

```java
public interface AuthenticationProvider {
    //认证的方法
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
	//支持哪种认证
    boolean supports(Class<?> authentication);
}
```

```java
//AbstractUserDetailsAuthenticationProvider
public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
```

这里对于`AbstractUserDetailsAuthenticationProvider`，他的`support`方法就表明他可以处理用户名密码这样的认证。

### Authentication接口：认证信息

```java
public interface Authentication extends Principal, Serializable {
//获取权限信息列表
Collection<? extends GrantedAuthority> getAuthorities();
//获取凭证信息。用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。
Object getCredentials();
//细节信息，web应用中的实现接口通常为 WebAuthenticationDetails，它记录了访问者的ip地 址和sessionId的值。
Object getDetails();
//身份信息，大部分情况下返回的是UserDetails接口的实现类
Object getPrincipal();
boolean isAuthenticated();
void setAuthenticated(boolean var1) throws IllegalArgumentException;
}

```

继承自`Principal`类，代表一个抽象主体身份。继承了一个getName()方法来表示主 体的名称。



### UserDetailsService接口: 获取用户信息

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

获取用户信息的接口，只有根据用户名获取用户信息的方法。

> 在`DaoAuthenticationProvider`的`retrieveUser`方法中，会获取spring容器中的 `UserDetailsService`。如果我们没有自己注入UserDetailsService对象，那么在 `UserDetailsServiceAutoConfiguration`类中，会在启动时默认注入一个带user用户 的`UserDetailsService`。 我们可以通过注入自己的UserDetailsService来实现加载自己的数据。



### UserDetails: 用户信息实体

代表了一个用户实体，包括用户、密码、权限列表、账号过期、认证过期、是否启用、是否锁定。

```java
public interface UserDetails extends Serializable {
Collection<? extends GrantedAuthority> getAuthorities();
String getPassword();
String getUsername();
boolean isAccountNonExpired();
boolean isAccountNonLocked();
boolean isCredentialsNonExpired();
boolean isEnabled();
}
```



###  PasswordEncoder 密码解析器

```java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);

    boolean matches(CharSequence rawPassword, String encodedPassword);

    default boolean upgradeEncoding(String encodedPassword) {
        return false;
    }
}

```

对密码进行加密解密

> `DaoAuthenticationProvider`在`additionalAuthenticationChecks`方法中会获取 Spring容器中的**PasswordEncode**r来对用户输入的密码进行比较。



#### BCryptPasswordEncoder

> `SpringSecurity`中最常用的密码解析器。他使用**BCrypt**算法。他的特点是加密可以加**盐sault**，但是解密不需要盐。因为盐就在密文当中。这样可以通过每次添加不同的盐，而给同样的字符串加密出不同的密文。 密文形如： `$2a$10$vTUDYhjnVb52iM3qQgi2Du31sq6PRea6xZbIsKIsmOVDnEuGb/.7K` 
>
> $是分割符，无意义；2a是bcrypt加密版本号；10是cost的值；而后的前22 位是salt值；再然后的字符串就是密码的密文了



## 授权流程

在用户认证通过后，对访问资源的权限进行检查的过程。

Spring Security通过http.authorizeRequests()对web请求进行授权保护，使用 标准Filter建立对web请求的拦截，实现对资源的授权访问。

1. **拦截请求**，已认证用户访问受保护的web资源将被`SecurityFilterChain`中(实现类为`DefaultSecurityFilterChain`)的 `FilterSecurityInterceptor` 的子类拦截。

2. **获取资源访问策略**，`FilterSecurityInterceptor`会从 `SecurityMetadataSource` 的子类`DefaultFilterInvocationSecurityMetadataSource` 获取要访问当前资源所需要的权限`Collection`。

   > `SecurityMetadataSource`其实就是读取访问策略的抽象，读取的内容是在配置类中对`SecurityFilterChain filterChain(HttpSecurity http)`的配置

3. **最后**，`FilterSecurityInterceptor`调用`AccessDecisionManager`进行授权决策，若决策通过，则允许访问资源，否则将禁止访问。关于AccessDecisionManager接口，最核心的就是其中的`decide`方法。这个方法用来鉴定当前用户是否有访问对应受保护资源的权限。

   ```java
   public interface AccessDecisionManager {
       //通过传递的参数来决定用户是否有访问对应受保护资源的权限
       void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException;
   
       boolean supports(ConfigAttribute attribute);
   
       boolean supports(Class<?> clazz);
   }
   ```

   > **authentication**：要访问资源的访问者的身份
   >
   > **object**：要访问的受保护资源，web请求对应`FilterInvocation`
   >
   > **configAttributes**：是受保护资源的访问策略，通过`SecurityMetadataSource`获取。



## 决策流程

在`AccessDecisionManager`的实现类`ConsensusBased`中，使用投票的方式来确定是否能访问受保护的资源。

`AccessDecisionManager`中包含了一系列的`AccessDecisionVoter`讲会被用来对 `Authentication`是否有权访问受保护对象进行投票，`AccessDecisionManager`根据投票结果，做出最终角色。

> 投票是因为权限可以从多个方面来进行配置，有角色但是没有资源，这就需要有不同的处理策略

```java
public interface AccessDecisionVoter<S> {
    int ACCESS_GRANTED = 1;
    int ACCESS_ABSTAIN = 0;
    int ACCESS_DENIED = -1;

    boolean supports(ConfigAttribute attribute);

    boolean supports(Class<?> clazz);

    int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);
}
```

`vote()`是进行投票的方法。投票可以表示赞成、拒绝、弃权。

**Spring Security**内置了三个基于投票的实现类，分别是 `AffirmativeBased`,`ConsensusBasesd`,`UnanimaousBased`

- **AffirmativeBased**的逻辑：只要有一 个投票通过，就表示通过。 

  1、只要有一个投票通过了，就表示通过。 

  2、如果全部弃权也表示通过。

  3、如果没有人投赞成票，但是有人投反对票，则抛出`AccessDeniedException`。

- **ConsensusBased**的逻辑：多数赞成就通过。

  1、赞成票多于反对票则表示通过。

  2、反对票多于赞成票则抛出`AccessDeniedException`。

  3、赞成票与反对票相同且不等于0，并且属性 `allowIfEqualGrantedDeniedDecisions`为`true`，则表示通过，否则抛出 `AccessDeniedException`。参数`allowIfEqualGrantedDeniedDecisions`的值默认`true`。 

  4、如果所有的`AccessDecisionVoter`都弃权，则视参数`allowIfAllAbstainDecisions`的值而定，如果该值为`true`则通过，否则抛出异常`AccessDeniedException`。参数`allowIfAllAbstainDecisions`的值默认`false`

-  **UnanimousBased**相当于一票否决。

   1、受保护对象配置的某一个`ConfigAttribute`被任意的 `AccessDecisionVoter`反对了，则将抛出`AccessDeniedException`。 

  2、没有反对票，但是有赞成票，则表示通过。 

  3、如果全部弃权了，则将视参数`allowIfAllAbstainDecisions`的值而定，`true`通过，`false`抛出`AccessDeniedException`。

  

> **Spring Security**默认使用`AffirmativeBased`投票器，我们同样可以通过往 Spring容器里注入的方式来选择投票决定器



## 会话控制

用户认证通过后，为避免用户的每次操作都认证，可将用户的信息保存在会话中。**Spring security**提供会话管理，认证通过后将身份信息放入**SecurityContextHolder**上下文，**SecurityContext**与当前线程进行绑定，方便获取用户身份。



- `SecurityContextHolder.getContext().getAuthentication()`获取当前登录用户信息。

  ```java
  @GetMapping("/getLoginUser")
  public String getLoginUser(){
  Principal principal =
  (Principal)SecurityContextHolder.getContext().getAuthentication().getPrincipal();
  return principal.getName();
  }
  ```

  

- 可通过配置`sessonCreationPolicy`参数来了控制如何管理Session。

  ```java
  protected void configure(HttpSecurity http) throws Exception {
  http.sessionManagement()
  .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED) }
  ```

| 机制       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| always     | 如果没有Session就创建一个                                    |
| ifRequired | 如果需要就在登录时创建一个                                   |
| never      | SpringSecurity将不会创建Session。但是如果应用中其他地方创建了 Session，那么Spring Security就会使用。 |
| stateless  | SpringSecurity将绝对不创建Session，也不使用。适合于一些REST API 的无状态场景。 |



- 会话超时时间可以通过spring boot的配置读取。

  ```yml
  server:
    servlet:
      session:
        timeout: 
  ```

- session超时后，可以通过SpringSecurity的http配置跳转地址

  ```java
  .sessionManagement(session -> session
                          .invalidSessionUrl("/common/invalidSession")//session失效后跳转到这个页面
                          .maximumSessions(1)//最大session数
                          .maxSessionsPreventsLogin(true)//当达到最大值时，是否阻止新的登录
                          .expiredUrl("/common/invalidSession")//session失效后跳转到这个页面
                  );
  ```

  expired是指session过期，invalidSession指传入的sessionId失效。

  

- 我们可以使用httpOnly和secure标签来保护我们的会话cookie

  ```yml
  server:
    servlet:
      session:
        cookie:
          http-only: true
          secure: true
  ```

  **httpOnly**：如果为true，那么浏览器脚本将无法访问cookie **secure**：如果为true，则cookie将仅通过HTTPS连接发送

  

- Spring Security默认实现了logout退出，直接访问/logout就会跳转到登出页面，而ajax访问/logout就可以直接退出。

  ```java
  .logout(logout->logout
                          .logoutUrl("/logout")
                          .logoutSuccessUrl("/index.html")//退出登录后跳转到这个页面
                          .invalidateHttpSession(true)//清除session 默认为true
                  );//配置退出登录)
  ```

  在退出操作时，会做以下几件事情：

  1. 使HTTP Session失效。
  2. 清除SecurityContextHolder 
  3. 跳转到定义的地址。

- **logoutHandler** 

  一般LogoutHandler 的实现类被用来执行必要的清理，因而他们不应该抛出异常。



**下面是Spring Security提供的一些实现**

- PersistentTokenBasedRememberMeServices 基于持久化token的 **RememberMe**功能的相关清理
- TokenBasedRememberMeService 基于token的**RememberMe**功能的相关清理
- CookieClearingLogoutHandler 退出时**Cookie**的相关清理
- CsrfLogoutHandler 负责在退出时移除**csrfToken**
- CecurityContextLogoutHandler 退出时**SecurityContext**的相关清理

链式API提供了调用相应的 LogoutHandler 实现的快捷方式，比如 deleteCookies()。
