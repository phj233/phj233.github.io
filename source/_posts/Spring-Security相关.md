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

