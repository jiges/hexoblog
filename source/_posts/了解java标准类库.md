---
title: 了解java标准类库
date: 2017-11-15 09:45:22
tags: [java]
categories: java源码阅读
---
> 参考jdk8标准

在阅读`java`源码之前，首先要了解`java`各个类库有什么样的作用，各个类库之前又有什么样的关联。
# java.applet包
包括`Applet`类和几个接口，用于创建`Java`小程序，处理小程序与浏览器之间的相互作用，包括声音图像等多媒体的处理。扩展包`javax.swing`增加了`JApplet`，该类派生自`Applet`，是其扩展版。`applet`现在主流浏览器几乎不用，可以忽略。
 <!-- more -->
# java.awt包
该包包括许多Java早期的图形界面设计相关的类与接口，包括定义字体、颜色、集合绘图和图像显示等。
# java.beans 包
包含与开发 `beans` 有关的类，即基于 `JavaBeans` 架构的组件。 `java`反射就用到了此包。
# java.io包
该包包括处理字节数据流(`InputStream`, `OutputStream`)，字符数据流(`Reader`, `Writer`)，文件操作类(`File`)等，实现各种不同的输入输出功能。
# java.lang包
是提供利用 `Java` 编程语言进行程序设计的基础类。编程用到该包中的类时，无需使用`import`语句引入他们，由编译器自动引入。
该类包含最重要的类`Object`、数据类型的包装类 (`Boolean`,`Character`, `Double`, `Float`, `Integer`, `Long` 等)、数学类 (`Math`)、系统类(`System`)和运行时类(`Runtime`)、字符串类 `String`, `StringBuffer`)、异常处理类(`Throwable`, `Exception`,`Error`)、线程类 (`Thread`, `Runnable`接口)、类操作类(`Class`)、`annotation`包等。
# java.math包
用来处理任意精度的整形或浮点型数据的。`BigDecimal`和`BigInteger`等。
# java.net包
用于网络通信，实现网络功能。
# java.nio包
`Java NIO`是`java 1.4`之后新出的一套`IO`接口，这里的的新是相对于原有标准的`Java IO`和`Java Networking`接口。`NIO`提供了一种完全不同的操作方式。
* `Java NIO: Channels and Buffers`
标准的`IO`编程接口是面向字节流和字符流的。而`NIO`是面向通道和缓冲区的，数据总是从通道中读到`buffer`缓冲区内，或者从`buffer`写入到通道中。
* `Java NIO: Non-blocking IO`
`Java NIO`使我们可以进行非阻塞`IO`操作。比如说，单线程中从通道读取数据到`buffer`，同时可以继续做别的事情，当数据读取到`buffer`中后，线程再继续处理数据。写数据也是一样的。
* `Java NIO: Selectors`
`NIO`中有一个“`slectors`”的概念。`selector`可以检测多个通道的事件状态（例如：链接打开，数据到达）这样单线程就可以操作多个通道的数据。

# java.rmi包
提供远程方法调用接口。
# java.security包
为安全框架提供类和接口。
# java.sql包
用于数据库操作的一些类。
# java.text包
提供以与自然语言无关的方式来处理文本、日期、数字和消息的类和接口。典型的`DateFormat`就在该包内。
# java.time包
时间处理包。
# java.util包
Java提供一些常用的实用功能类，数据结构的`Java`实现类。日期时间类（`Date`，`Calender`）、随机数类（`Random`）、向量类（`Vector`）、堆栈类（`Stack`）、散列表类（`Hashtable`）、`Java`集合框架。
# javax包
`javax`是`java`的扩展包,如`j2ee` 中的类库，包括`servlet`，`jsp`，`ejb`，数据库相关的一些东西，`xml`等。