# 前言

volatile 不能保证原子性

### 问题

### 一. i++ 的原子问题

i++ 实际上是分为三个步骤的"读 - 改 - 写"

```java
int i = 10;
i = i++;
// 最终 i = 10
```

###### 底层原理:

```java
// i++ 进行的操作
int tmep = i;
int i = i + 1;
int i = temp;
```



# 二. 原子变量

在 JDK 1.5之后, `java.util.concureent.atomic` 包下提供了常用的原子变量

### 特点

```
1. volatile 保证内存可见性
2. CAS (compare and swap) 算法保证数据的原子性
```

