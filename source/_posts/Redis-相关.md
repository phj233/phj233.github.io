---
title: Redis 相关
date: 2023-04-27 15:09:59
categories:
- Program
tags:
- 笔记
- Redis
- 数据结构h'
s
---

## String 字符串

字符串是最基本的 Redis 数据类型, 它们通常用于缓存。

默认情况下，单个 Redis 字符串的最大大小为 512 MB。

### set

在redis存储字符串

```shell
set user phj233
```

#### ex

存储user:code 并从现在起 100 秒过期

```shell
set user:code 777777 ex 3600
```

### get

```shell
get user
return phj233
```

### incr

user:code 递增1

```shell
incr user:code
return 777778
```

### incrby

user:code 增加 指定值

```shell
incrby user:code 10
return 777787
```

## List 列表

列表是一个由字符串组成的有序集合，可以在列表的两端推入或弹出元素，也可以根据索引获取或修改元素。

### lpush

将一个或多个字符串插入到列表的左侧。

```
lpush mylist "hello"
```

### rpush

将一个或多个字符串插入到列表的右侧。

```
rpush mylist "world"
```

### lrange

获取列表中指定范围的元素。

```
lrange mylist 0 1
```

### llen

获取列表的长度

```shell
llen mylist
return 2
```

### lpop

从列表的左侧弹出一个元素，并返回它。

```shell
lpop mylist
```

### rpop

从列表的右侧弹出一个元素，并返回它。

```shell
rpop mylist
```

## Hash 表

结构化为key和value的记录类型

### hset

向hash表添加key value

```sql
hset user username phj233 password 123456
return: 2
```

### hmset

设置多个哈希键值对

```shell
hmset user age 18 gender "male"
```

### hget

获取hash表指定key的value

```sql
hget user username
return: "phj233"
```

### hmget

在 Redis 中，hmget 命令用于获取哈希表中多个键的值。

```shell
hmget user:info name email
return ["John Doe", "johndoe@example.com"]
```

### hgetall

获取指定表全部数据

```sql
hgetall user
return:
"password"
"123456"
"username"
"phj233"
```



### hincrby

对指定key的value增加指定整数

## Set 集合

 Set是唯一字符串的无序集合

- 跟踪唯一项目（例如，跟踪访问给定博客文章的所有唯一 IP 地址）。
- 表示关系（例如，具有给定角色的所有用户的集合）。
- 执行常见的集合操作，例如交集、并集和差分。

Redis  Set的最大大小为 2^32 - 1 （4，294，967，295） 个成员。



### sadd

向集合添加元素：

储存用户：

```sql
sadd user phj233
return: 1
```

储存用户的code：

```sql
sadd user:phj233:code 123456
```

example:

```sql
sadd user:phj233:num 465789
sadd user:phj233:num 123456
```



### smembers

获取集合中的所有元素

```shell
smembers myset
return ["apple"]
```

### sismember

检查value是否存在集合中

```sql
sismember user:phj233:code 123456
return: true
```



### sinter

查询两个集合交集

```sql
sinter user:phj233:code user:phj233:num
return: 123456
```



### scard

集合计数

```sql
scard user:phj233:num
return: 2
```



### srem

移出集合元素

```sql
srem user phj233
return: 1
```



## Sorted Set 有序集合

有序集合是一个无序的字符串集合，每个元素关联一个分数，可以根据分数进行排序。

### zadd

向有序集合添加一个或多个元素

```shell
zadd myzset 1 "apple"
```

### zrange

获取有序集合中指定范围内的元素

```shell
zadd myzset 2 "banana"
zadd myzset 3 "cherry"
zrange myzset 0 1
return ["apple", "banana"]
```

### zrevrange

获取有序集合中指定范围内的元素，按照分数从大到小排序

```shell
zrevrange myzset 0 1
return ["cherry", "banana"]
```

### zcard

获取有序集合的元素数量

```shell
zcard myzset
return 3
```

### zscore

获取有序集合中指定元素的分数

```shell
zscore myzset "cherry"
return 3
```

### zrem

从有序集合中删除一个或多个元素

```shell
zrem myzset "cherry"
```

## Bitmaps 位图

位图是一种紧凑的数据结构，用于存储二进制位的值，可以进行一些位运算操作。

### setbit

设置位图中指定位置的值

```shell
setbit mybitmap 0 1
```

### getbit

获取位图中指定位置的值

```shell
getbit mybitmap 0
return 1
```

### bitcount

获取位图中值为 1 的二进制位的数量

```shell
setbit mybitmap 1 1
setbit mybitmap 3 1
bitcount mybitmap
return 2
```

### bitop

对多个位图进行位运算操作

```shell
setbit mybitmap2 0 1
bitop AND myresult mybitmap mybitmap2
getbit myresult 0
return 0
```

## HyperLogLog 基数统计

HyperLogLog 是一种概率性的数据结构，用于估计一个集合中不同元素的数量。HyperLogLog 以完美的精度换取高效的空间利用率。

### pfadd

向 HyperLogLog 中添加一个或多个元素

```shell
pfadd myloglog "apple"
pfadd myloglog "banana"
```

### pfcount

获取 HyperLogLog 中估计的元素数量

```shell
pfcount myloglog
return 2
```

### pfmerge

合并两个集合

```she
PFADD hll1 foo bar zap a
PFADD hll2 a b c foo
PFMERGE hll3 hll1 hll2
return ok
```



## Geo 地理空间

Geo 是一种用于存储地理空间信息的数据结构，可以对地理空间信息进行查询和计算。

### geoadd

向 Geo 中添加一个或多个地理空间信息

```shell
geoadd mygeo 116.48105 39.996794 "Beijing"
```

### geopos

获取指定地理空间信息的经纬度坐标

```shell
geopos mygeo "Beijing"
return [[116.48105, 39.996794]]
```

### geodist

计算两个地理空间信息之间的距离

```shell
geoadd mygeo 116.417524, 39.928216 "Peking University"
geodist mygeo "Beijing" "Peking University"
return 4239.2289
```

### georadius

获取指定范围内的地理空间信息

```shell
georadius mygeo 116.45635 39.91803 10 km
return ["Peking University"]
```

## Stream 消息流

Stream 是一种消息流数据结构，用于存储消息并支持消费者组的消费。

### xadd

向 Stream 中添加一条消息

```shell
xadd mystream * name "phj233" age 18
```

### xread

从 Stream 中读取消息

```shell
xread COUNT 1 STREAMS mystream 0-0
return [["mystream", [["1-0", {"name":"phj233", "age":"18"}]]]]
```

### xgroup

创建或管理消费者组

```shell
xgroup CREATE mystream mygroup 0-0
```

### xreadgroup

从 Stream 中以消费者组的方式读取消息

```shell
xreadgroup GROUP mygroup myconsumer COUNT
```

### xack

确认消费者组已经成功消费了一条或多条消息

```shell
xack mystream mygroup 1-0
```

### xdel

从 Stream 中删除一条或多条消息

```shell
xdel mystream 1-0
```

##  Bitfields

Bitfields 是 Redis 3.2 引入的新数据类型，它可以看作是对 Redis 字符串类型的扩展，支持对字符串中的某些位进行读写操作。

### bitfield

对 Redis 字符串中的位进行读写操作

```shell
bitfield mykey SET u4 0 15 GET u4 0
return [0, 0]
```

### bitcount

统计 Redis 字符串中值为 1 的位数

```shell
bitcount mykey
return 0
```

### bitop

对多个 Redis 字符串进行位运算，并将结果存储到新的 Redis 字符串中

```shell
set key1 "101010"
set key2 "111000"
bitop AND destkey key1 key2
return 6
```

### bitpos

查找 Redis 字符串中值为 1 的位的位置

```shell
set mykey "\xff\xf0\x00"
bitpos mykey 1
return 12
```
