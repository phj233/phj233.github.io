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

1. **拦截请求**，已认证用户访问受保护的web资源将被`SecurityFilterChain`中(实现类为`DefaultSecurityFilterChain`)的 `FilterSecurityInterceptor` 的子类拦截。
2. **获取资源访问策略**，`FilterSecurityInterceptor`会从 `SecurityMetadataSource` 的子类`DefaultFilterInvocationSecurityMetadataSource` 获取要访问当前资源所需要的权限`Collection`。
