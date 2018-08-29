---
title: 关于java
urlname: about_java
date: 2017-09-25 16:25:28
categories: 
 - java
 - jvm
 - 《深入理解java虚拟机》
tags:
---

## java体系

> java不仅仅是一门编程语言，还是由一系列计算机软件和规范形成的技术体系。
这个体系提供了完整的用于软件开发和跨平台部署的支持环境。

**Sun官方定义的java技术体系包括以下内容**：  
1. java程序设计语言
2. 各硬件平台上的java虚拟机（jvm）
3. Class文件格式
4. java api类库
5. 来自商业机构和开源社区的第三方类库

以上1、2、4点统称为jdk，jdk是支持java程序开发的最小环境.  
java api中的java se api子集和java虚拟机统称为jre，jre是java程序运行的最小环境.

java除了一门优秀的编程语言外，还有许多不可忽视的优点：
- 摆脱硬件平台束缚，实现跨平台
- 内存管理，避免绝大部分内存泄露和指针越界问题
- 热点代码检测和运行时编译及优化
- 完善的应用程序编程接口及丰富、优秀的第三方类库

java技术体系按照所服务的领域来划分，可以分为四大模块：`java card` `java se` `java me` `java ee`

## jdk发展史

java语言前身：oak
- 1995年，oak更名为java，同时sun正式提出"write once, run anywhere"的口号，标识着java正式诞生
- 1996年，jdk1.0发布，并提供了一个纯解释性的java虚拟机（Sun Classic VM）
- 1997年，jdk1.l发布，代表技术有：jdbc, rmi, java beans, .jar文件等
- 1998年，jdk1.2发布，正式把java技术体系划分为3个方向：java se, java me, java ee，代表技术较多，如ejb, java pluin-in, swing等。java虚拟机中第一次内置了jit编译器，且附带了HotSpot作为备选
- 2000年，jdk1.3发布，将HotSpot作为默认虚拟机，并提供jndi作为平台级服务，从此后Sun公司基本保持两年一个主版本的更新速度
- 2002年，jdk1.4发布，标志着java正式走向成熟，代表技术有：正则表达式、异常链、nio、日志类、xml解析器等等
- 2004年，jdk1.5(jdk5)发布，主要是对java语言的易用性方面做了改进
- 2006年，jdk1.6(jdk6)发布，除了改进之外，Sun正式将java开源（GPL协议），并建立了OpenJDK组织对其进行管理
jdk1.7(jdk7)开发，Sun公司由于各种原因，最终无法按计划完成。2009年Oracle公司将其收购后，对jdk1.7的内容进行大幅度裁剪，并把剩余内容延迟到jdk1.8中。jdk1.7于2011年正式发布
- 2013年，jdk1.8(jdk8)发布，加入了lambda表达式及函数式接口等支持


## 部分jvm介绍
- Sun Classic VMr
    - Classic VM是Sun公司发布的第一款商用java虚拟机，作为默认虚拟机与jdk1.0绑定发布，技术较为原始
    - 在jdk1.2之前都是唯一存在的虚拟机
    - 只能使用纯解释器方式来运行java程序
    - 如果要使用JIT编译器，必须外挂

- Exact VM
    - 为了解决Classic VM存在的各种问题而开发，在jdk1.2期间发布
    - 具备了现代高性能虚拟机的原型
    - 在商用只存在了短暂的时间后，即被更为优秀的HotSpot取代

- Sun HotSpot VM
    - 是SunJDK及OpenJDK中所带的虚拟机，也是目前使用最广的虚拟机
    - 最初是由一家名为"Longview Teconologies"的小公司设计，在1997年该公司被Sun收购
    - 其即有Sun之前两款虚拟机的优点，也有许多自身的技术优势如热点代码探测能力

- Sun Mobile-Embedded VM/Meta-Circular VM
    - Sun公司针对移动和嵌入式市场的虚拟机

- BEA JRockit
    - 一款专门为服务端硬件和服务端应用场景而专门优化的虚拟机
    - 不关注应用程序的启动速度，因而其内部不含解析器的实现，全部代码靠JIT编译运行
    - 除此之外，其垃圾收集器和MissionControl等服务套件的实现也一直在众多虚拟机中保持领先的水平

- IBM J9 VM
- Azul VM/BEA Liquid VM
- Apache Harmony/Google Android Dalvik VM
- Microsoft JVM