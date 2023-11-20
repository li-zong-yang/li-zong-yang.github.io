# Java基础

> 第七人格

## String为什么是不可变的？

String 被声明为 final，因此它不可被继承。

在 Java 8 中，String 内部使用 char 数组存储数据。value 数组也被声明为 final，意味着 value 数组初始化之后就不能再引用其它数组。并且 String 内部没有改变 value 数组的方法，因此可以保证 String 不可变。

**如果你非要修改他，可以通过反射的方式修改其value，达到修改String的目的。**

不可变的好处

1. 可以缓存 hash 值因为 String 的 hash 值经常被使用，例如 String 用做 HashMap 的 key。不可变的特性可以使得 hash 值也不可变，因此只需要进行一次计算。 
2. String Pool 的需要如果一个 String 对象已经被创建过了，那么就会从 String Pool 中取得引用。只有 String 是不可变的，才可能使用 String Pool。 
3. 安全性String 经常作为参数，String 不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果 String 是可变的，那么在网络连接过程中，String 被改变，改变 String 对象的那一方以为现在连接的是其它主机，而实际情况却不一定是。
4.  线程安全String 不可变性天生具备线程安全，可以在多个线程中安全地使用。

