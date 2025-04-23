---
title: 《Java》JMM之“happens-before”
date: 2020-05-12 12:19:31
categories:
  - [ java, jmm]
---

	这是Java内存模型（JMM）系列的第四篇文章，主要介绍的是happens-before特性。

# 一、happens-before
通过这个概念来阐述操作之间的内存可见性。

<!-- more -->

# 二、JSR-133 对 happens-before 原则的定义
通过这个概念来阐述操作之间的内存可见性。如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须存在happens-before 关系。
两个操作之间存在 happens-before 关系，并不意味着 Java 平台的具体实现必须要按照 happens-before 关系指定的顺序来执行。如果重排序之后的执行结果，与按 happens-before 关系来执行的结果一致，那么 JMM 也允许这样的重排序。
```java
int userNum = getUserNum();   // 1
int teacherNum = getTeacherNum();   // 2
int totalNum = userNum + teacherNum;  // 3
```

```dtd
1 happens-before 2
2 happens-before 3
1 happens-before 3
```
虽然 1 happens-before 2，但对 1 和 2 进行重排序不会影响代码的执行结果，所以 JMM 是允许编译器和处理器执行这种重排序的。但 1 和 2 必须是在 3 执行之前，也就是说 1,2 happens-before 3 。



# 三、规则如下
- **程序顺序规则：** 一个线程的每个操作，happens-before于该线程中的任意后续操作
- **监视器锁规则：** 一个监视器的解锁，happens-before于该监视器的加锁
- **volatile变量规则：** 一个volatile的写，happens-before对该变量的读
- **传递性：** A happens-before B，B happens-before C ，那么A happens-before C
