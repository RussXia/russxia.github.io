---
layout: blog
title: "JDK9中String为何改为byte数组存储"
catalog: true
header-img: img/post-bg-2.jpg
subtitle: JDK9中String改为byte数组存储有何优点
date: 2024-07-22
tags: [2024,Java]
---
## JDK8中String的字符编码
JDK 8 中，String以`char[]`形式存储，字符串始终在内存中使用UTF-16编码，每个字符占用两个字节。

即使该字符是ASCII字符或Latin-1字符，这种情况也不例外。因此，可以确认在JDK 8中，String对象始终在内存中使用UTF-16编码。

你可以在输入、输出、转换过程中，将其指定为特定字符编码，如`UTF-8`。
```Java
String a = "A你🚗";
byte[] bytes = a.getBytes(UTF_8);
System.out.println(bytes.length);   // 1+3+4=8
```

## JDK9中String的字符编码
在JDK 9中开始以`byte[]`形式存储，因为JDK引入了`Compact Strings`机制，以优化内存使用。String类根据字符串的实际内容在以下两种编码之间选择：

+ Latin-1（ISO-8859-1）： 如果字符串中的所有字符都处在Latin-1范围内（U+0000到U+00FF），则使用单字节编码（Latin-1）。
+ UTF-16： 如果字符串包含超出Latin-1范围的字符（即U+0100及以上字符），则使用双字节编码（UTF-16）。

如果`Compact Strings`被disabled，字符串将一直以UTF-16字符编码存储在byte[]数组中。

可以通过JVM参数：`-XX:-CompactStrings`来禁用"Compact Strings"机制。

**总结** ：在JDK 8中，String类在内存中仅使用UTF-16编码。在JDK 9及更高版本中，String类在内存中支持两种编码：Latin-1和UTF-16。

## UTF-8和UTF-16的区别是什么
+ UTF-8
    + 可变长度编码: UTF-8使用1到4个字节来编码每个Unicode字符。
        + 1字节: 适用于基本ASCII字符（U+0000至U+007F）。
        + 2字节: 适用于拉丁字母及其他常见字符（U+0080至U+07FF）。
        + 3字节: 适用于多语言字符，包括汉字（U+0800至U+FFFF）。
        + 4字节: 适用于辅助平面字符（U+10000至U+10FFFF）。

+ UTF-16
    + 可变长度编码: UTF-16使用2或4个字节来编码每个Unicode字符。
        + 2字节: 用于基本多语言平面内的字符（U+0000至U+FFFF），即通常通过单个16位代码单元表示。
        + 4字节: 用于BMP外的字符（U+10000至U+10FFFF），需要使用一对代理对（Surrogate Pair），每对代理对由两个16位代码单元组成。

因为存储原理上的区别:
+ UTF-8
    + 对于纯ASCII文本更为高效，因为每个字符仅占1个字节。
    + 对于混合文本（包括ASCII和非ASCII字符），存储效率可变，但一般来说比UTF-16稍低，特别是当文本中包含大量非拉丁字符时。
+ UTF-16
    + 对于大量非ASCII字符（如汉字、日文假名等）的文本，UTF-16通常更为高效，因为大多数字符只占用2个字节。
    + 单纯ASCII文本的存储效率较低，每个字符占用2个字节。

**纯中文**
```Java
String originalString = "世";
// String originalString = "世界";
// String originalString = "世界你好";

// 将字符串转换为UTF-8编码的字节数组
byte[] utf8Bytes = originalString.getBytes(StandardCharsets.UTF_8);
// 将字符串转换为UTF-16编码的字节数组
byte[] utf16Bytes = originalString.getBytes(StandardCharsets.UTF_16);

System.out.println("Original: " + originalString);  
//3->6->12
System.out.println("UTF-8 bytes length: " + utf8Bytes.length); 
System.out.println();

System.out.println("UTF-16 bytes length: " + utf16Bytes.length);
//4->6->10
System.out.println();
```
**纯英文**
```Java
String originalString = "A";
// String originalString = "AB";
// String originalString = "ABCD";

// 将字符串转换为UTF-8编码的字节数组
byte[] utf8Bytes = originalString.getBytes(StandardCharsets.UTF_8);
// 将字符串转换为UTF-16编码的字节数组
byte[] utf16Bytes = originalString.getBytes(StandardCharsets.UTF_16);

System.out.println("Original: " + originalString);
//1->2->4
System.out.println("UTF-8 bytes length: " + utf8Bytes.length);
System.out.println();

//4->6->10
System.out.println("UTF-16 bytes length: " + utf16Bytes.length);
System.out.println();
```
**纯表情**
```Java
String originalString = "🚗";
// String originalString = "🚗😊";
// String originalString = "🚗😊😭🔥";

// 将字符串转换为UTF-8编码的字节数组
byte[] utf8Bytes = originalString.getBytes(StandardCharsets.UTF_8);
// 将字符串转换为UTF-16编码的字节数组
byte[] utf16Bytes = originalString.getBytes(StandardCharsets.UTF_16);

System.out.println("Original: " + originalString);
//4->8->16
System.out.println("UTF-8 bytes length: " + utf8Bytes.length);
System.out.println();

//4->6->10
System.out.println("UTF-16 bytes length: " + utf16Bytes.length);
System.out.println();
```


## 总结
+ JDK 8 中的 String 类：
    + 字符串始终在内存中使用UTF-16编码，每个字符占用两个字节。
+ JDK 9 及更高版本中的 String 类：
    + 使用"Compact Strings"机制，字符串可以在Latin-1或UTF-16编码之间动态选择，以提高内存使用效率。

