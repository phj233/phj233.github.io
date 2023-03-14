---
title: Java 笔记
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
        implementation 'org.springframework.cloud:spring-cloud-dependencies:2022.0.1'
        implementation 'com.alibaba.cloud:spring-cloud-alibaba-dependencies:2021.1'
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
- `subprojects{}`添加`apply{}`
- 在`apply{}`添加`plugin`

