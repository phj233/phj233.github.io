---
title: Spring-Boot相关
date: 2023-07-19 23:18:52
categories:
- Program
tags:
- 笔记
- Spring Boot
---

# 读取配置文件

## @Value

```java
@Value("${phj233.name}")
private String name;
```

> 必须是Bean,不能为final 或 static
>
> 读取的配置在,application.yml中必须存在



## @ConfigurationProperties

```java
@Component
@ConfigurationProperties(prefix = "phj233")
public class ValueTest {
    private String name;
    private Integer age;
}
```

> 必须被Spring托管
>
> 属性自动被key中对应的值赋值



## Environment接口

```java
public class ValueTest {
    @Resource
    private Environment env;

    public  void test() {
        String PHJ233_NAME = env.getProperty("phj233.name");
    }
}
```



## @PropertySources

获取自己新建的**phj233.properties**文件

```java
@PropertySources({
        @org.springframework.context.annotation.PropertySource(value = "classpath:phj233.properties", encoding = "UTF-8"),
})
public class ValueTest {
    @Value("${phj233.name}")
    private String name;
}
```

> 只能获取properties文件



## PropertySourcesPlaceholderConfigurer

获取自定义yaml文件的配置

```java
public class ValueTest {
    @Bean
    public static PropertySourcesPlaceholderConfigurer yaml() {
        PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
        YamlPropertiesFactoryBean yaml = new YamlPropertiesFactoryBean();
        yaml.setResources(new ClassPathResource("phj233.yml"));
        configurer.setProperties(Objects.requireNonNull(yaml.getObject()));
        return configurer;
    }
    @Value("${phj233.name}")
    private String name;
}
```

