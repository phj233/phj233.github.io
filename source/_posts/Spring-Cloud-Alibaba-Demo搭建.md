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



## 微服务注册于Nacos

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

![image-20230315233347843](https://s2.loli.net/2023/03/15/Hz98T2dxXCqV5nY.png)
