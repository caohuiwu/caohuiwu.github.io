---
title: 《Java》JMM之“final”
date: 2020-05-13 12:19:31
categories:
  - [ java, jmm]
---

<style>
.my-code {
   color: orange;
}
.orange {
   color: orange
}
.red {
   color: red
}
code {
   color: #0ABF5B;
}
</style>

	这是Java内存模型（JMM）系列的第五篇文章，主要介绍的是final特性。

# 一、final

<!-- more -->
`final`
在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序
实现
JMM 禁止编译器把 final 域的写重排序到构造函数之外
编译器会在 final 域的写之后，构造函数 return 之前，插入一个 StoreStore 屏障。这个屏障禁止处理器把 final 域的写重排序
到构造函数之外
初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序
实现
在一个线程中，初次读对象引用与初次读该对象包含的 final 域，JMM 禁止处理器重排序这两个操作（注意，这个规则仅仅针
对处理器）。编译器会在读 final 域操作的前面插入一个 LoadLoad 屏障