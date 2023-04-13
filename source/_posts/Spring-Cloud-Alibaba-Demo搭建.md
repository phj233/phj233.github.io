---
title: Spring_Cloud_Alibaba-Demo搭建
date: 2023-03-15 19:43:36
categories:
- Program
tags:
- 笔记
- SpringCloud-Alibaba
- necos
- SpringCloud
- sentinal

---

## 新建SpringCloud

![image-20230315221505362](https://s2.loli.net/2023/03/15/2RvXcgb7Us9ljmp.png)

- 使用`Spring Initializr`生成器，这里选择`Gradle`进行构建，JDK为17。**SpringBoot3.0**开始不再支持**JDK17**以下的版本

### 选择依赖Lombok、JPA、MySQL、Spring-Boot-starter-web

![image-20230315221712690](https://s2.loli.net/2023/03/15/czr4aBfeY6UQlOA.png)

### 新建模块

![](https://s2.loli.net/2023/03/15/rHOJpNWdYFhT2jl.png)

#### 新建shop-user、shop-common、shop-product、shop-order模块

![image-20230315222250621](https://s2.loli.net/2023/03/15/tSvZCcrNKmns6yu.png)

#### 配置父模块build.gradle

- 新建`subprojects`
- 将`dependencies`与`tasks.named('test')`移至`subprojects`
- 在`subprojects`中添加`apply`并添加`plugin`

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

### 搭建shop-user

![image-20230315225457993](https://s2.loli.net/2023/03/15/afgInOt1jCXLR9S.png)

#### 创建`UserEntity`实体类

`info.phj233.shop_user.model.dto.UserEntity`

```java
@Entity(name = "shop_user")
@Data
public class UserEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "uid")
    private Integer uid;
    @Column(name = "username")
    private String username;
    @Column(name = "password")
    private String password;
    @Column(name = "email")
    private String email;
    @Column(name = "phone")
    private String phone;
    @Column(name = "role")
    private String role;
}
```

#### 创建`dao`层与`service`层

`info.phj233.shop_user.dao.UserDao`

```java
@Repository
public interface UserDao extends JpaRepository<UserEntity, Integer> {
}
```

#### 配置application.yml

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/spring_cloud_alibaba?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=UTC
    username: root
    password: phj123456
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
  application:
    name: shop-user
server:
  port: 8071
```



### 搭建shop-product

![image-20230315230040162](https://s2.loli.net/2023/03/15/s1rKPXtcMnj4Fq2.png)

#### 创建`ProductEntity`实体类

`info.phj233.shop_product.model.dto`

```java
@Entity(name = "shop_product")
@Data
public class ProductEntity {
    @Id
    @Column(name = "pid")
    private Integer pid;
    @Column(name = "pname")
    private String pname;
    @Column(name = "pprice")
    private String pprice;
    @Column(name = "pstock")
    private String pstock;
    @Column(name = "pdescription")
    private String pdescription;
    @Column(name = "ptype")
    private String ptype;
    @Column(name = "pimage")
    private String pimage;
}
```

#### 创建`dao`层与`service`层

`info.phj233.shop_product.dao`

```java
@Repository
public interface ProductDao extends JpaRepository<ProductEntity, Integer> {
    ProductEntity findProductEntityByPid(Integer id);
}
```

`info.phj233.shop_product.service`

```java
public interface ProductService {
    ProductEntity findProductById(Integer id);
}
```

`info.phj233.shop_product.service.impl`

```java
@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {
    private final ProductDao productDao;

    @Override
    public ProductEntity findProductById(Integer id) {
        if (!ObjectUtils.isEmpty(id)) {
            return productDao.findProductEntityByPid(id);
        }
        return null;
    }
}
```



#### 配置application.yml

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/spring_cloud_alibaba?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=UTC
    username: root
    password: phj123456
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
  application:
    name: shop-product
server:
  port: 8081
```



### 搭建shop-order

![image-20230315230518863](https://s2.loli.net/2023/03/15/dXt8ju4JBS1hoY3.png)

#### 创建`OrderEntity`实体类

`info.phj233.shop_order.model.dto`

```java
@Entity(name = "shop_order")
@Data
public class OrderEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "oid")
    private Integer oid;
    @Column(name = "uid")
    private Integer uid;
    @Column(name = "username")
    private String username;
    @Column(name = "pid")
    private Integer pid;
    @Column(name = "pname")
    private String pname;
    @Column(name = "pprice")
    private Double pprice;
    @Column(name = "number")
    private Integer number;
}
```

#### 创建`dao`层与`service`层

`info.phj233.shop_order.dao`

```java
@Repository
public interface OrderDao extends JpaRepository<OrderEntity, Integer> {
    OrderEntity findByPid(Integer id);
}
```

`info.phj233.shop_order.service`

```java
public interface OrderService {
    Boolean insertOrder(OrderEntity orderEntity);
}
```

`info.phj233.shop_order.service.impl`

```java
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
    final private OrderDao orderDao;

    @Override
    public Boolean insertOrder(OrderEntity orderEntity) {
        if (!ObjectUtils.isEmpty(orderEntity)) {
            orderDao.save(orderEntity);
            return true;
        }
        return false;
    }
}
```

## 搭建Nacos

### Nacos下载

[Releases · alibaba/nacos (github.com)](https://github.com/alibaba/nacos/releases)

### 启动Nacos

直接启动可在 **nacos/bin**中打开**shell**环境执行`startup.cmd -m standalone`，现可配置于IDEA

- 点击运行/调试配置 中的编辑配置

![image-20230315231451814](https://s2.loli.net/2023/03/15/bRs6h5CMFSZlx92.png)

- 点击“+”，选中Shell Script

  ![image-20230315231711651](https://s2.loli.net/2023/03/15/IuPTzRswBLpoZqi.png)

- 如下配置

  ![image-20230315231929791](https://s2.loli.net/2023/03/15/KDU3IjJSGrgfuTO.png)



## 微服务注册进Nacos

### 修改所有子模块application.yml

添加：

```yaml
spring：
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848	//nacos端口
```



### 测试微服务调用

#### 新增OrderController

`info.phj233.shop_order.controller`

```java
@RestController
@RequestMapping("/order")
@RequiredArgsConstructor
@Slf4j
public class OrderController {
    final private RestTemplate restTemplate;
    final private OrderDao orderDao;
    final private DiscoveryClient discoveryClient;
    @GetMapping("/prod/{pid}")
    public OrderEntity orderEntity(@PathVariable("pid") Integer id) {
        ServiceInstance serviceInstance = discoveryClient.getInstances("shop-product").get(0);
        String url = serviceInstance.getHost() + ":" + serviceInstance.getPort();
        ProductEntity productById = restTemplate
                .getForObject("http://"+url+"/product/" + id, ProductEntity.class);
        log.info("商品信息查询结果" + productById);
        OrderEntity orderEntity = orderDao.findByPid(id);
        orderEntity.setOid(orderEntity.getOid());
        orderEntity.setUid(orderEntity.getUid());
        orderEntity.setUsername(orderEntity.getUsername());
        orderEntity.setPid(id);
        orderEntity.setPname(Objects.requireNonNull(productById).getPname());
        orderEntity.setPprice(Double.valueOf(productById.getPprice()));
        orderEntity.setNumber(orderEntity.getNumber());
        return orderEntity;
    }
}

```

 启动Nacos并启动子模块后访问相应接口：

<img src="https://s2.loli.net/2023/03/15/Hz98T2dxXCqV5nY.png" alt="image-20230315233347843" style="zoom: 67%;" />



## 实现微服务的负载均衡

### 负载均衡

`负载均衡（Load Balance，简称 LB）`是高并发、高可用系统必不可少的关键组件，目标是尽力将网络流量平均分发到多个服务器上，以提高系统整体的响应速度和可用性。

![image-20230316145946951](https://s2.loli.net/2023/03/16/ayZruvX6txMR81j.png)

> **高并发**：负载均衡通过算法调整负载，尽力均匀的分配应用集群中各节点的工作量，以此提高应用集群的并发处理能力（吞吐量）。
>
> **伸缩性**：添加或减少服务器数量，然后由负载均衡进行分发控制。这使得应用集群具备伸缩性。
>
> **高可用**：负载均衡器可以监控候选服务器，当服务器不可用时，自动跳过，将请求分发给可用的服务器。这使得应用集群具备高可用的特性。
>
> **安全防护**：有些负载均衡软件或硬件提供了安全性功能，如：黑白名单处理、防火墙，防 DDos 攻击等。

通俗的讲， 负载均衡就是将负载（工作任务，访问请求）进行分摊到多个操作单元（服务器,组件）上 进行执行。

### 负载均衡实现策略

- 1.轮询策略

  `轮询策略：RoundRobinRule`，按照一定的顺序依次调用服务实例。比如一共有 3 个服务，第一次调用服务 1，第二次调用服务 2，第三次调用服务3，依次类推。

- 2.权重策略

  `权重策略：WeightedResponseTimeRule`，根据每个服务提供者的响应时间分配一个权重，响应时间越长，权重越小，被选中的可能性也就越低。它的实现原理是，刚开始使用轮询策略并开启一个计时器，每一段时间收集一次所有服务提供者的平均响应时间，然后再给每个服务提供者附上一个权重，权重越高被选中的概率也越大。

- 随机策略

  `随机策略：RandomRule`，从服务提供者的列表中随机选择一个服务实例。此策略的配置设置如下：

- 4.最小连接数策略

  `最小连接数策略：BestAvailableRule`，也叫**最小并发数策略**，它是遍历服务提供者列表，选取连接数最小的⼀个服务实例。如果有相同的最小连接数，那么会调用轮询策略进行选取。

- 5.重试策略

  `重试策略：RetryRule`，按照轮询策略来获取服务，如果获取的服务实例为 null 或已经失效，则在指定的时间之内不断地进行重试来获取服务，如果超过指定时间依然没获取到服务实例则返回 null。

- 6.可用性敏感策略

  `可用敏感性策略：AvailabilityFilteringRule`，先过滤掉非健康的服务实例，然后再选择连接数较小的服务实例。

- 7.区域敏感策略

  `区域敏感策略：ZoneAvoidanceRule`，根据服务所在区域（zone）的性能和服务的可用性来选择服务实例，在没有区域的环境下，该策略和轮询策略类似。

### 修改`OrderController`

```java
@GetMapping("/prod/{pid}")
public OrderEntity orderEntity(@PathVariable("pid") Integer id) {
    // 通过服务名获取服务实例
    List<ServiceInstance> instances = discoveryClient.getInstances("shop-product");
    // 从服务实例中获取主机名和端口号
    ServiceInstance serviceInstance = instances.get(new Random().nextInt(instances.size()));
    // 拼接请求地址
    String url = serviceInstance.getUri().toString();
    log.info(url);
    ProductEntity productById = restTemplate
            .getForObject(url+"/product/" + id, ProductEntity.class);
    log.info("商品信息查询结果" + productById);
    ......
```

修改后启动各微服务，打开**nacos**发现`shop-product`服务成功实现负载均衡，健康实例数 2

![image-20230316144348643](https://s2.loli.net/2023/03/16/DsFq6o7l3iuIjOn.png)

多次调用`order/prod/` 接口，发现端口改变

![image-20230316145403911](https://s2.loli.net/2023/03/16/IWg4TkbE5yLSCJq.png)



## 使用Ribbon实现负载均衡

由于`spring-cloud-starter-alibaba-nacos-discovery`2.2.9RELEASE(大概是。。) 后去掉了`spring-cloud-starter-netflix-ribbon`，所以暂不撰写



## 使用OpenFeign实现服务调用

Feign 是 Netflix 开发的声明式、模板化的 HTTP 客户端，其灵感来自 Retrofit、JAXRS-2.0以及 WebSocket。Feign 可帮助我们更加便捷、优雅地调用 HTTP API。

Feign 可以做到使用 HTTP **请求远程服务时就像调用本地方法一样的体验**

- 父模块加入依赖

  ```groovy
  implementation 'org.springframework.cloud:spring-cloud-starter-openfeign:4.0.1'
  implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer:4.0.1'
  ```

- 在需要使用服务调用的服务启动类上打`@EnableFeignClients(basePackages=xx)`

  **basePackages**为被调用服务的接口所在包，可不添加，默认扫描当前模块

  ![image-20230411141613470](https://s2.loli.net/2023/04/11/EsaVLuoPW43IfTl.png)

- 新建Feign客户端`@FeignClient(name = "shop-product", path = "/product")`

  name：为被调用服务的名字

  path为被调用服务的Controller中的`@RequestMapping`

  ![image-20230411141825861](https://s2.loli.net/2023/04/11/kUn2MbFft3NRmBq.png)

- 修改调用服务的`OrderController`注入被调用的服务的客户端

  ![image-20230411141949535](https://s2.loli.net/2023/04/11/KtoB7CU2My5rnGX.png)

- 写上调用代码并运行

  ![image-20230317002532401](https://s2.loli.net/2023/03/17/9r5OY8hyDWdSQuN.png)

  调用成功！



## 服务容错

微服务架构中，一个应用往往由多个服务组成，服务之间相互依赖，关系错综复杂。

例如一个微服务系统中存在 A、B、C、D、E、F 等多个服务

![img](https://s2.loli.net/2023/03/29/jNsXpvg4YBSJuot.png)

当服务 E 发生故障或网络延迟时，会出现以下情况：

1. 即使其他所有服务都可用，由于服务 E 的不可用，那么用户请求 1、2、3 都会处于阻塞状态，等待服务 E 的响应。在高并发的场景下，会导致整个服务器的线程资源在短时间内迅速消耗殆尽。
2. 所有依赖于服务 E 的其他服务，例如服务 B、D 以及 F 也都会处于线程阻塞状态，等待服务 E 的响应，导致这些服务的不可用。
3. 所有依赖服务B、D 和 F 的服务，例如服务 A 和服务 C 也会处于线程阻塞状态，以等待服务 D 和服务 F 的响应，导致服务 A 和服务 C 也不可用。

当微服务系统的一个服务出现故障时，故障会沿着服务的调用链路在系统中疯狂蔓延，最终导致整个微服务系统的瘫痪，这就是“**雪崩效应**”。为了防止此类事件的发生，微服务架构引入了“熔断器”的一系列服务容错和保护机制。

### 容错思路

- 隔离

  它是指将系统按照一定的原则划分为若干个服务模块，各个模块之间相对独立，无强依赖。当有故 障发生时，能将问题和影响隔离在某个模块内部，而不扩散风险，不波及其它模块，不影响整体的 系统服务。常见的隔离方式有：线程池隔离和信号量隔离

- 超时

  在上游服务调用下游服务的时候，设置一个最大响应时间，如果超过这个时间，下游未作出反应， 就断开请求，释放掉线程

- 限流

  限流就是限制系统的输入和输出流量已达到保护系统的目的。为了保证系统的稳固运行,一旦达到 的需要限制的阈值,就需要限制流量并采取少量措施以完成限制流量的目的

- 熔断

  当下游服务因访问压力过大而响应变慢或失败，上游服务为了保护系统整 体的可用性，可以暂时切断对下游服务的调用。这种牺牲局部，保全整体的措施就叫做熔断。

  - 熔断关闭状态（Closed）：服务没有故障时，熔断器所处的状态，对调用方的调用不做任何限制 
  - 熔断开启状态（Open）： 后续对该服务接口的调用不再经过网络，直接执行本地的fallback方法 
  - 半熔断状态（Half-Open）： 尝试恢复服务调用，允许有限的流量调用该服务，并监控调用成功率。如果成功率达到预期，则说明服务已恢复，进入熔断关闭状态；如果成功率仍旧很低，则重新进入熔断关闭状态

- 降级

  降级其实就是为服务提供一个托底方案，一旦服务无法正常调用，就使用托底方案



### 集成Sentinel

Sentinel 分为两个部分:

-  核心库（Java 客户端）不依赖任何框架/库,能够运行于所有 Java 运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。 

-  控制台（Dashboard）基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等 应用容器。



1. 引入依赖

```groovy
implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-sentinel:2022.0.0.0-RC1'
```

2. 安装Sentinel控制台

   Sentinel 提供一个轻量级的控制台, 它提供机器发现、单机资源实时监控以及规则管理等功能

   [Releases · alibaba/Sentinel (github.com)](https://github.com/alibaba/Sentinel/releases)

3. 添加shell脚本到idea

​			![image-20230330144005172](https://s2.loli.net/2023/03/30/c4fs2jOt6PdX9Sq.png)

start.sh内容为

```bash
java -Dserver.port=8999 -Dcsp.sentinel.dashboard.server=localhost:8999 -Dcsp.sentinel.log.dir="./logs/csp/" -jar F:\codeingEnv\Java\sentinel\sentinel-dashboard-2.0.0-alpha-preview.jar
```



#### 修改需要容错的服务配置

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8999
```

访问对应服务的接口即可容错监控

#### 配置流控模式

- 直接（默认）：接口达到限流条件时，开启限流

​		对`/order/message1`配置，快速访问接口即可限流	![image-20230330153110432](https://s2.loli.net/2023/03/30/hemlOIzydB25QVY.png)

![image-20230330153323617](https://s2.loli.net/2023/03/30/9nGdiBQbktoOyCV.png)



- 关联：当关联的资源达到限流条件时，开启限流 [适合做应用让步]

1. 同样对`/order/message1`进行配置

​		![image-20230330153536439](https://s2.loli.net/2023/03/30/bziep9U7jSaf2Ql.png)

2. 使用压测工具对message2进行访问


![image-20230330153644901](https://s2.loli.net/2023/03/30/n4vViGx8qFDQeKw.png)
	
![image-20230330153711995](https://s2.loli.net/2023/03/30/Y34Tn2eDmBUVsM8.png)

3. 访问message1，成功进行限流

![image-20230330153852178](https://s2.loli.net/2023/03/30/dSP3kvYtJHK7q8I.png)



- 链路：当从某个接口过来的资源达到限流条件时，开启限流

> chainA、chainB两个接口都调用某一资源chain，chainA->chain、chainB->chain可以看成两个简单的链路，此时可以针对chain配置链路限流，比如限制chainA调用chain，而chainB调用chain则不受影响，它的功能有点类似于关联，而链路流控是**针对上级接口**，它的粒度更细。

1. 在`OrderService`接口添加`message`方法，并在方法上打上`@SentinelResource("message")`注解


![image-20230330154448624](https://s2.loli.net/2023/03/30/PxLQYwi5stepVWo.png)

2. 在`OrderController`中声明两个方法，都调用`OrderService`中的方法

​		![image-20230330190706859](https://s2.loli.net/2023/03/30/Qorkz7UFKbJ1BpS.png)

3. 修改服务的配置文件

```yaml
spring:
  cloud:
    sentinel:
      web-context-unify: false
```

4. sentinel添加流控规则

   对相应Service资源配置，入口资源为需要流控的接口

   ![image-20230330191133342](https://s2.loli.net/2023/03/30/YyazCg7bKjeLuGO.png)

5. 多次调用流控接口访问失败，但另外调用相应Service的接口不影响

   ![image-20230330191417082](https://s2.loli.net/2023/03/30/aLrQo459kdGWsCI.png)

<img src="https://s2.loli.net/2023/03/30/huLKJGzRFUO1l7x.png" alt="image-20230330191532751" style="zoom: 33%;" />

### Fallback

#### 对FeignClient的Fallback

修改Order的`ProductClient`，对`@FeignClient`注解添加`fallback`参数

![image-20230411141825861](https://s2.loli.net/2023/04/11/Znsh9cmTXPD3jf5.png)

新建`ProductClientFallback`类，实现`ProductClient`，并使用`@Component`注解

![image-20230411142623604](https://s2.loli.net/2023/04/11/SM72Ob4zerhK9Bg.png)

启动order服务，关闭product服务，fallback成功

![image-20230411142834510](https://s2.loli.net/2023/04/11/4p3EOksR6YdWCtF.png)



若遇到以下报错

```java
Error creating bean with name 'info.phj233.shop_order.client.ProductClient': FactoryBean threw exception on object creation

Cannot invoke "org.springframework.cloud.openfeign.FeignClientFactoryBean.getFallback()" because "feignClientFactoryBean" is null
```

 可对调用服务进行配置

```yaml
spring:  
  cloud:  
    openfeign:
      lazy-attributes-resolution: true
```



## 引入网关

### 基础网关

新建`shop-api`模块，可直接使用Spring生成器生成，记得勾选Gateway

![image-20230411151757601](https://s2.loli.net/2023/04/11/e5NfHStl2asnEKc.png)



修改配置，加入：

```yaml
server:
  port: 7000
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:   #路由,转发请求
        - id: product_route   #路由id，唯一标识
          uri: http://localhost:8081    #请求转发的目标地址
          order: 1   #路由的执行顺序，数字越小优先级越高
          predicates:   #断言,路由转发的条件
            - Path=/product/**
          filters:   # 过滤器,请求在传递过程中,可以对请求进行一些操作
            - StripPrefix=1   #转发请求之前,去掉1层前缀
```



运行api模块，访问相应路径成功

![image-20230411151934539](https://s2.loli.net/2023/04/11/pCZm9rv4cuqxQnG.png)



### 搭配nacos

![image-20230411155338818](https://s2.loli.net/2023/04/11/glUaJTL1idQ6EKV.png)

- 对网关模块引入依赖

  `implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-discovery:2022.0.0.0-RC1'`

  `implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'`

- 在api模块运行类上加入`@EnableDiscoveryClient`注解

- 修改配置文件，添加配置

  ```yaml
  spring:
    cloud:
      nacos:
        discovery:
          server-addr: 127.0.0.1:8848
      gateway:
        discovery:
          locator:
            enabled: true  #开启服务发现
        routes:   #路由,转发请求
          - id: product_route   #路由id，唯一标识
            uri: lb://shop-product      #请求转发的目标地址,lb是负载均衡的意思，即从注册中心获取服务地址
            order: 1   #路由的执行顺序，数字越小优先级越高
            predicates:   #断言,路由转发的条件
              - Path=/product_service/**
            filters:   # 过滤器,请求在传递过程中,可以对请求进行一些操作
              - StripPrefix=1   #转发请求之前,去掉1层前缀
  ```

- 访问地址，加载成功

  ![image-20230411153757923](https://s2.loli.net/2023/04/11/p4Vrfu9BotEzwx5.png)

#### 简写

去掉route的配置，使用服务在注册中心nacos的名字进行访问

![image-20230411154031712](https://s2.loli.net/2023/04/11/TQXAfK4HGVZyzu8.png)



### 关于网关

![image-20230411155032755](https://s2.loli.net/2023/04/11/nGoKsz9Nt4lRjhw.png)

传统的架构存在以下问题：

- 客户端多次请求不同的微服务，增加客户端代码或配置编写的复杂性
- 认证复杂，每个服务都需要独立认证。
- 存在跨域请求，在一定场景下处理相对复杂。

现可引入API网关解决：

​	系统的统一入口，它封装了应用程序的内部结构，为客户端提供统一服务，一些与业务本身功能无关的公共逻辑可以在这里实现，诸如认证、鉴权、监控、路由转发等等。 添加上API网关之后，系统的架构图变成了如下所示：

![image-20230411155321781](https://s2.loli.net/2023/04/11/bFGl3hiIvqSHn7O.png)

#### 基本概念

路由(Route) 是 gateway 中最基本的组件之一，表示一个具体的路由信息载体。

主要定义了下面的几个 信息:

- id:路由标识符，区别于其他 Route
- uri:路由指向的目的地 uri，即客户端请求最终被转发到的微服务
- order:用于多个 Route 之间的排序，数值越小排序越靠前，匹配优先级越高
- predicate:断言的作用是进行条件判断，只有断言都返回真，才会真正的执行路由
- fliter:过滤器用于修改请求和响应信息。

#### 执行流程

![image-20230411154703123](https://s2.loli.net/2023/04/11/Pg1CfBLydMOUIbl.png)

1. Gateway Client向Gateway Server发送请求
2. 请求首先会被HttpWebHandlerAdapter进行提取组装成网关上下文
3. 然后网关的上下文会传递到DispatcherHandler，它负责将请求分发给 RoutePredicateHandlerMapping 
4. RoutePredicateHandlerMapping负责路由查找，并根据路由断言判断路由是否可用 
5. 如果过断言成功，由FilteringWebHandler创建过滤器链并调用
6. 请求会一次经过PreFilter--微服务--PostFilter的方法，最终返回响应



## 消息中间件

### RabbitMQ

- 在需要使用中间件的两个服务引入依赖

  `implementation 'org.springframework.boot:spring-boot-starter-amqp'`

- 在中间件application.yml添加配置

  ![image-20230413084049760](https://s2.loli.net/2023/04/13/kzdEVbhQTRDaeWq.png)

  ```yaml
  spring:
    rabbitmq:
      host: 20.187.105.115
      port: 5672
      username: phj233
      password: phj123+-/
  ```

- 在`OrderController`注入`RabbitTemplate`,并发送消息

  ![image-20230413084401811](https://s2.loli.net/2023/04/13/LH1hW87ulSdaFrt.png)

​		routingKey为队列名，消费者接收方需要以此来接收消息

- 在user模块新建`UserConsumer`类

  ![image-20230413084638205](https://s2.loli.net/2023/04/13/nXw637iNoktVuDL.png)



开启服务，访问相应接口，成功

![image-20230413085004447](https://s2.loli.net/2023/04/13/qkH6RNdjghDru1I.png)

![image-20230413085036684](https://s2.loli.net/2023/04/13/TphD5Wob3Oxt6uJ.png)
