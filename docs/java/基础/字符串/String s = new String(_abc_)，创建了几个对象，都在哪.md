---
description: String s = new String("abc")，创建了几个对象，都在哪
title: String s = new String("abc")，创建了几个对象，都在哪
tag:
  - JAVA
sidebar: true
comment: true
recommend: 1
---
# String s = new String("abc")，创建了几个对象，都在哪

```java
  String s = new String("abc");
```

会涉及几个对象的创建？我们一步步拆解：

## 1. 字面量 "abc"
当类加载器加载到这行代码时，编译器已经把 "abc" 放到 字符串常量池（String Pool） 中。

如果常量池中已经有 "abc"，则不会再新建；如果没有，就在常量池里创建一个新的对象。

👉 情况 A： "abc" 第一次出现 → 在常量池中创建 1 个对象。

👉 情况 B： "abc" 已经存在 → 不会再创建。

## 2. new String("abc")
new 关键字一定会在 堆内存 (Heap) 中创建一个新的 String 对象。

这个对象的内容来自常量池中的 "abc"，内部依然持有那个 "abc" 的 char[]。

👉 无论如何，堆中一定会多 1 个对象。

## 3. 变量引用
s 只是一个引用，指向堆中新建的 String 对象，不算对象创建。

## 📌 总结
```java
String s = new String("abc");
```

常量池中：可能创建 1 个 "abc"（取决于之前是否已经有了）。

堆中：一定创建 1 个 String 对象。

因此：

![](https://cdn.nlark.com/yuque/0/2025/png/791535/1756301696227-d92edafb-a552-44cd-bfdb-15cb623b1cae.png)

**第一次出现时 → 共 2 个对象（常量池 "abc" + 堆中的 new String）。**

**如果 "abc" 已存在 → 共 1 个对象（堆中新建的 String）。**

