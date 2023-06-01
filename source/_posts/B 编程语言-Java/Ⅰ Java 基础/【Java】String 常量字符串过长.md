---
title: 【Java】String 常量字符串过长
date: 2023-06-01
categories: 
- 编程语言-Java
- Java 基础
---

## 1. 参考博客

图片转 BASE64 编码： https://c.runoob.com/front-end/59/
图片与base64之间的互相转换「Java」： https://blog.csdn.net/weixin_41608476/article/details/79354592

## 2. 问题描述

在开发一个接口时间，对方接口需要图片的 BASE64 编码。

> Base64 是一种基于 64 个可打印字符来表示二进制数据的表示方法。
> 
> Base64 常用于在通常处理文本数据的场合，表示、传输、存储一些二进制数据，包括 MIME 的电子邮件及 XML 的一些复杂数据。
> 图片的 BASE64 编码就是可以将一幅图片数据编码成一串字符串，使用该字符串代替图片地址，从而不需要使用图片的 URL 地址。

当 BASE64 编码的图片 赋值给 String 后，构建失败，提醒常量字符串过长。

## 3. 探究 String

基础知识：[[../../../1 想法/【Java】2 数据类型#2.3 字符串 String]]
基础知识：[[../../B. 操作系统-Linux/计算机基础知识/数据单位]]

在 Java8 之前，String 内部使用 char 类型数据存储数据， Java9 开始，该用 byte 类型数组来存储数据。所以 String 的最大存储就是数组能存储的最大数量。

数组有两种限制。一是规范隐含的限制。Java 数组的 length 必须是非负的 int，所以它的理论最大值就是 java.lang.Integer.MAX_VALUE = 2^31-1 = 2147483647 位。二是具体的实现带来的限制。这会使得实际的JVM不一定能支持上面说的理论上的最大length。

String 当是一个字符数组。字符是 2字节 16 位 的基本类型，那么一个 String 的最大长度是是 2 G，占用内存是 4GB。

**问题**

我的字符串肯定没有超过最大限制，但是字符串肯定超过了 2^16。「在IDEA中，字符串长度超过 65535，进行打印，IDEA会提示 `java: 常量字符串过长`」

> 为什么会提醒我的字符串错过了 65535
> 引入文章： https://blog.csdn.net/jayzym/article/details/103716868

**编译期**

当我们使用字符串字面量直接定义 String 的时候，是会把字符串在常量池中存储一份的。那么上面提到的 65534 其实是常量池的限制。
常量池中的每一种数据项也有自己的类型。Java 中的 UTF-8 编码的 Unicode 字符串在常量池中以 CONSTANT_Utf8 类型表示。
CONSTANTUtf8info 是一个 CONSTANTUtf8 类型的常量池数据项，它存储的是一个常量字符串。常量池中的所有字面量几乎都是通过 CONSTANTUtf8info 描述的。CONSTANTUtf8_info的定义如下：
```markdown
CONSTANT_Utf8_info {
	u1 tag;     
	u2 length;     
	u1 bytes[length];
}
```

我们使用字面量定义的字符串在 class 文件中，是使用 CONSTANTUtf8info 存储的，而 CONSTANTUtf8info 中有 u2 length; 表明了该类型存储数据的长度 u2 是无符号的 16 位整数，因此理论上允许的的最大长度是 2^16=65536。而 java class 文件是使用一种变体 UTF-8 格式来存放字符的，null 值使用两个字节来表示，因此只剩下 65536－ 2 ＝ 65534 个字节。

关于这一点，在the class file format spec中也有明确说明：
```vbnet
The length of field and method names, field and method descriptors, and other constant string values is limited to 65535 characters by the 16-bit unsigned length item of the CONSTANTUtf8info structure (§4.4.7). Note that the limit is on the number of bytes in the encoding and not on the number of encoded characters. UTF-8 encodes some characters using two or three bytes. Thus, strings incorporating multibyte characters are further constrained.
```

也就是说，在Java中，所有需要保存在常量池中的数据，长度最大不能超过 65535，这当然也包括字符串的定义咯。

**运行期**

运行期的最大长度就是 Integer.MAX_VALUE ，这个值约等于 4G。

## 4. 解决方案

1. 使用 StringBuilder 或者 StringBuffer。
2. 切换 Java 编译器，将 `javac` 切换为 `eclipse` 「Eclipse 编译器允许运行没有真正正确编译的代码」



