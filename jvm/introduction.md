# 说明及简介

> 本系列文档主要参考[《Java虚拟机规范》][jvm_spec_8]和周志明老师的[《深入理解Java虚拟机（第3版）》][deep_in_jvm_3]。

## 说明

1. 本系列文档中所提及的《Java虚拟机规范》在默认情况下均是专指 [The Java Virtual Machine Specification SE8 Edition][jvm_spec_8]，例外情况会另行说明。
1. 本系列文档中所提及的《Java语言规范》在默认情况下均是专指 [The Java Language Specification SE8 Edition][jls]，例外情况会另行说明。
1. 本系列文档中的代码示例默认情况下均基于OpenJDK 8，例外情况会另行说明。

## 简介

Java虚拟机是Java语言平台的基石，有自己的指令集，并在运行时对多个内存区域进行操作。但它和Java语言关系其实不大，其他基于Java虚拟机的编程语言也可以运行在Java虚拟机之上，如：Groovy、Kotlin、Scala等。

Java虚拟机也是一套抽象的规范，它并不专指某一个具体的Java虚拟机实现（如HotSpot等），但是它规定了Java虚拟机实现应该遵循怎样的结构及规范。了解Java虚拟机规范对我们深入理解Java语言有很大帮助。

通常来讲，一个Java虚拟机实现通常需要包含：

- 编译子系统：负责将Java源码编译成字节码（如javac编译器），并在运行时将字节码编译成本地机器码（如JIT编译器），这两部分的编译工作通常是分开实现的
- 执行子系统：负责执行字节码指令
- 自动内存管理子系统：负责内存的分配及回收

本系列文档将大致按照：编译 -> class文件 -> 类加载 -> 执行 -> 垃圾回收，这样的顺序来对Java虚拟机作相应的介绍。

### Java虚拟机简史

在Java发展的历史上出现过很多不同的虚拟机实现，这里做一下简单介绍。

#### Classic VM 和 Exact VM

Sun发布的JDK1.0中所带的虚拟机实现就是Classic VM，这款虚拟机采用纯解释执行的方式来执行Java代码，因此执行效率和C/C++程序有很大差距。

为了解决Classic VM的各种问题，提升运行效率，在JDK1.2时，Sun的虚拟机团队曾在Solaris平台上发布过一款Exact VM，它因采用准确式内存管理而得名，具备了现代高性能虚拟机的雏形，在技术上相对Classic VM先进许多，但是并没有能够真正实现商用。

#### HotSpot VM

大名鼎鼎的HotSpot虚拟机最初并非由Sun公司所开发，而是来源于一家被Sun收购的小公司。HotSpot VM因其热点代码探测技术而得名，它可以通过执行计数器找出最具有编译价值的代码，然后通知即时编译器以方法为单位进行编译。

#### BEA JRockit VM 和 IBM J9 VM

除了Sun/Oracle公司研发的Java虚拟机，还有其他大型商业公司也有自己的Java虚拟机实现，其中比较出色的是BEA System公司的JRockit虚拟机与IBM公司的IBM J9虚拟机。

BEA将JRockit发展为一款专门为服务器硬件和服务端应用场景高度优化的虚拟机，其内部不包含解释器实现，全部代码都靠即时编译器编译后执行。

而IBM的J9虚拟机的定位和HotSpot虚拟机比较接近，它是一款全面考虑服务端、桌面应用，嵌入式设备的多用途虚拟机。J9虚拟机的职责分离和模块化做得比HotSpot更优秀。

#### Graal VM

Graal VM是一款跨语言的全栈虚拟机，除了支持Java、Groovy、Kotlin、Scala等基于Java虚拟机的语言，还支持C、C++、Rust等基于LLVM的语言，同时还支持JavaScript、Python、Ruby等语言。Graal VM可以无额外开销地混合使用这些编程语言，支持在不同语言中混用对方的接口和对象。

从Java角度来看，Graal VM可以直接把Java应用编译成本地机器码，体积更小，执行效率更高。基于Graal VM的种种优秀特性，Twitter已经大规模使用它替代HotSpot VM来支撑Java服务。

除了上述介绍的虚拟机实现之外，其实还有一些其他的虚拟机实现，感兴趣的读者可以查看[《深入理解Java虚拟机（第3版）》][deep_in_jvm_3]这本书的1.4章节。

[jvm_spec_8]: https://docs.oracle.com/javase/specs/jvms/se8/html/index.html
[deep_in_jvm_3]: https://book.douban.com/subject/34907497/
[jls]: https://docs.oracle.com/javase/specs/jls/se8/html/index.html
