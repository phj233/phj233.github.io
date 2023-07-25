---
title: JavaDoc 常用标签
date: 2023-05-09 17:39:00
categories:
- Program
tags:
- 笔记
- Java
- JavaDoc
---

## JavaDoc 常用标签

JavaDoc 是 Java 中自带的文档工具，用于生成 Java 代码的 API 文档。在 JavaDoc 注释中，可以使用各种标签来标记不同的注释信息，以下是一些常用的 JavaDoc 标签：

### @param

用于描述方法参数的说明。

```java
/**
 * 计算两个整数的和
 * @param a 加数
 * @param b 加数
 * @return 两个数的和
 */
public int add(int a, int b) {
    return a + b;
}
```

### @return

用于描述方法返回值的说明。

```java
/**
 * 计算两个整数的和
 * @param a 加数
 * @param b 加数
 * @return 两个数的和
 */
public int add(int a, int b) {
    return a + b;
}
```

### @throws

用于描述方法抛出的异常的说明。

```java
/**
 * 除法运算
 * @param a 被除数
 * @param b 除数
 * @return 商
 * @throws ArithmeticException 如果除数为零，则抛出此异常
 */
public int divide(int a, int b) throws ArithmeticException {
    if (b == 0) {
        throw new ArithmeticException("除数不能为零");
    }
    return a / b;
}
```

### @deprecated

用于标记不推荐使用的方法或类。

```java
/**
 * 此方法已过时，建议使用 {@link #add(int, int)} 方法代替
 * @deprecated 该方法已过时
 */
@Deprecated
public int sum(int a, int b) {
    return a + b;
}
```

### @see

用于指向相关的其他类、方法或文档。

```java
/**
 * 获取当前日期
 * @return 当前日期字符串，格式为 yyyy-MM-dd
 * @see java.util.Date
 */
public String getCurrentDate() {
    Date date = new Date();
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
    return sdf.format(date);
}
```

### @link

用于指向相关的其他类、方法或文档，并提供一个可点击的链接。

```java
/**
 * 计算两个整数的和
 * @param a 加数
 * @param b 加数
 * @return 两个数的和，具体实现见 {@link com.example.MathUtils#add(int, int)}
 */
public int add(int a, int b) {
    return MathUtils.add(a, b);
}
```

### @since

用于描述方法或类添加的版本信息。

```java
/**
 * 获取当前日期
 * @return 当前日期字符串，格式为 yyyy-MM-dd
 * @since 1.0.0
 */
public String getCurrentDate() {
    Date date = new Date();
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
    return sdf.format(date);
}
```

### @version

用于描述方法或类的版本信息。

```java
/**
 * 计算两个整数的和
 * @param a 加数
 * @param b 加数
 * @return 两个数的和
 * @version 1.0.0
 */
public int add(int a, int b) {
    return a + b;
}
```

### @author

用于描述方法或类的作者信息。

```java
/**
 * 计算两个整数的和
 * @param a 加数
 * @param b 加数
 * @return 两个数的和
 * @version 1.0.0
 * @author John
 */
public int add(int a, int b) {
    return a + b;
}

```

