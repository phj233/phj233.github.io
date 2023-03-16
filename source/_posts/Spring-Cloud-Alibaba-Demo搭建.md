title: Spring_Cloud_Alibaba-Demo搭建
date: 2023-03-15 19:43:36
categories:
- Program
tags:
- 笔记
- SpringCloud-Alibaba
- necos
- SpringCloud

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

- 在需要使用服务调用的服务启动类上打`@EnableFeignClients(basePackages = "info.phj233.shop_product.service")`

  **basePackages**为被调用服务的接口

  ![image-20230317001555711](https://s2.loli.net/2023/03/17/eVaZiQU8c21GYrw.png)

- 在被调用接口上打上`@FeignClient(name = "shop-product", path = "/product")`

  name：为被调用服务的名字

  path为被调用服务的Controller中的`@RequestMapping`

  ![image-20230317001920203](https://s2.loli.net/2023/03/17/Xh7JIOogspPiAML.png)

- 修改调用服务的`OrderController`注入被调用的服务接口![image-20230317002408413](https://s2.loli.net/2023/03/17/xXsOHFRQ4vAcTKP.png)

- 写上调用代码并运行

  ![image-20230317002532401](https://s2.loli.net/2023/03/17/9r5OY8hyDWdSQuN.png)

  调用成功！

