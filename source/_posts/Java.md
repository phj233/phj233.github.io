---
title: Java
date: 2023-01-16 16:00:48
categories:
- Program
tags:
- 笔记
- Java
- Gradle
---

## 1.stream filter map结果集相关

stream流的`filter()`和`map()`返回的结果集是一个**新的**集合。

对结果集进行新增或删除元素，都不会改变原有集合。但对结果集中的元素进行修改，会改变原有集合中元素，因为结果集中的元素和原有集合中的元素是同一引用地址，也就是同一个对象

## 2.实现Java不可变集合

`List.of(x,x,x)`或`List.copyOf(list)`



## 3.List去重

-  循环添加去重

  ```java
  public void remove1(){
          List<String> list = new ArrayList<>(initList);
          List<String> list2 = new ArrayList<>();
          for (String element : list) {
              if (!list2.contains(element)) {
                  list2.add(element);
              }
          }
          //添加element进List2时，判断List2中是否已经存在该element，如果不存在则添加，否则不添加
      }
  ```

- 双循环去重

  ```java
  public void remove2(){
          List<String> list = new ArrayList<>(initList);
          for (int i = 0; i < list.size() - 1; i++) {
              for (int j = list.size() - 1; j > i; j--) {
                  if (list.get(j).equals(list.get(i))) {
                      list.remove(j);
                  }
              }
          }
          //利用双循环判断是否有重复元素，如果有则删除
      }
  ```

- 循环重复坐标去重

  ```java
  public void remove3(){
          List<String> list = new ArrayList<>(initList);
          List<String> list2 = new ArrayList<>(initList);
          for (String element : list2){
              if(list.indexOf(element) != list.lastIndexOf(element)){
                  list.remove(list.lastIndexOf(element));
              }
          }
          //复制一个List2再循环List2，判断List中的元素首尾出现的坐标是否一致，不一致就是重复的元素，移除List中的最后一个元素
      }
  ```

- Set去重

  ```java
  public void remove4(){
          List<String> list = new ArrayList<>(initList);
          List<String> list2 = new ArrayList<>(new HashSet<>(list));  //不保证顺序性
          List<String> list3 = new ArrayList<>(new LinkedHashSet<>(list2));   //保证顺序性
      }
      //set不包含重复元素，把List先装进set再装回来
  ```

- Stream去重

  ```java
  public void remove5(){
          List<String> list = new ArrayList<>(initList);
          list = list.stream().distinct().collect(Collectors.toList());
      }
      //distinct()去重
  ```


## 4.Gradle多模块配置



```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.0.4'
    id 'io.spring.dependency-management' version '1.1.0'
}

group = 'info.phj233'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

subprojects {
    apply {
        plugin 'java'
        plugin 'org.springframework.boot'
        plugin 'io.spring.dependency-management'
    }
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        implementation 'org.springframework.boot:spring-boot-starter-web'
        implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-discovery:2022.0.0.0-RC1'
        implementation 'org.springframework.cloud:spring-cloud-dependencies:2022.0.1'
        implementation 'com.alibaba.cloud:spring-cloud-alibaba-dependencies:2022.0.0.0-RC1'
        compileOnly 'org.projectlombok:lombok'
        runtimeOnly 'com.mysql:mysql-connector-j'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }

    tasks.named('test') {
        useJUnitPlatform()
    }
}
```

- 在父模块`build.gradle`中添加`subprojects{}`

- 把`dependencies`与`tasks.named('test')`移动进`subprojects{}`

- `subprojects{}`添加`apply{}`  (这里图方便直接在子模块加入所有依赖)

- 在`apply{}`添加`plugin`

  ![image-20230317002825573](https://s2.loli.net/2023/03/17/hNdUlFBXeWYv2ik.png)

## 5.jakarta.validation.NoProviderFoundException on application startup

主要报错：

```java
jakarta.validation.NoProviderFoundException: Unable to create a Configuration, because no Jakarta Bean Validation provider could be found. Add a provider like Hibernate Validator (RI) to your classpath.
```

添加依赖`spring-boot-starter-validation`

具体原因为：[问题 #1979 ·Springdoc/Springdoc-openAPI (github.com)](https://github.com/springdoc/springdoc-openapi/issues/1979)

## 6.k8s和Docker关系

Docker提供了容器化的能力，而K8s提供了对容器化应用程序的自动化管理能力。

## 7.hpa和pod啥关系

HPA通过增加或减少Pod的数量来实现自动水平伸缩。通过动态调整Pod的数量，可以根据实际需求来提供更好的性能和可用性，并有效地利用资源。

## 8.shell和bash啥区别

发行版差异：不同的操作系统发行版可能使用不同的`Shell`

默认情况下，Linux一般使用Bash作为默认的Shell，而其他UNIX系统可能使用不同的Shell实现。

## 9.多线程  锁   事务还有spring中的事务之间的联系场景

多线程业务中通常会使用**锁**保证共享变量的数据安全
事务通常是指一个操作中的所有动作需要全部成功或全部失败
spring中的事务是一种事务的实现。
多线程和锁有关系，事务和它们没有关系。
集群单例场景中，很多代码的功能在单机环境是没问题的，但是到了集群就会出现重复执行的情况下，这个时候就需要分布式锁保证集群单例。
