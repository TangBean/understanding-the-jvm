# 深入理解 Java 虚拟机阅读笔记

本 repo 为《深入理解 Java 虚拟机 第2版》的阅读笔记，并对全书内容按照自己的理解进行了一定程度的整理。《深入理解 Java 虚拟机 第2版》原书主要分为了五个部分，这里仅对前四个部分进行讲解，第五部分（高效并发）整合进了另一个 repo：[Java 并发编程实战的阅读笔记](https://github.com/TangBean/Java-Concurrency-in-Practice) 中。

其中前两部分：Java 内存管理机制和 Java 虚拟机程序执行需要重点掌握，并且内容也是比较多的。本 repo 将原书中有关虚拟机性能监控及故障处理的部分单抽了出来，组成了本 repo 的第三部分。第四部分对应于原书的第四部分，程序编译与代码优化，不过仅对 Java 的运行期优化，也就是 JIT 时进行的优化进行了总结，编译器优化部分尚未进行深入研究。

以下为本 repo 文章的目录：

## Content

### Java 内存管理机制

- [00-Java内存区域详解](./Ch1-Java内存管理机制/00-Java内存区域详解.md)
- [01-OOM异常](./Ch1-Java内存管理机制/01-OOM异常.md)
- [02-垃圾收集(GC)](./Ch1-Java内存管理机制/02-垃圾收集(GC).md)
- [03-Java内存分配策略](./Ch1-Java内存管理机制/03-Java内存分配策略.md)

### Java 虚拟机程序执行

- [00-Class文件的组成结构](./Ch2-Java虚拟机程序执行/00-Class文件的组成结构.md)
- [01-虚拟机的类加载机制](./Ch2-Java虚拟机程序执行/01-虚拟机的类加载机制.md)
- 02-虚拟机字节码执行引擎
	- [00-虚拟机栈栈帧结构](./Ch2-Java虚拟机程序执行/02-虚拟机字节码执行引擎_00-虚拟机栈栈帧结构.md)
	- [01-方法调用](./Ch2-Java虚拟机程序执行/02-虚拟机字节码执行引擎_01-方法调用.md)
	- [02-基于栈的字节码解释执行引擎](./Ch2-Java虚拟机程序执行/02-虚拟机字节码执行引擎_02-基于栈的字节码解释执行引擎.md)
- [附录0-实现Java类的热替换](./Ch2-Java虚拟机程序执行/附录0-实现Java类的热替换.md)

### 虚拟机性能监控及故障处理

- [00-常用虚拟机性能监控工具](./Ch3-虚拟机性能监控及故障处理/00-常用虚拟机性能监控工具.md)
- [01-虚拟机调优案例分析](./Ch3-虚拟机性能监控及故障处理/01-虚拟机调优案例分析.md)

### Java 程序运行优化

- [00-Java运行期优化](./Ch4-Java程序运行优化/00-Java运行期优化.md)



## “串一串” Java 虚拟机的知识点

本文将按照 Content 中给出的四个部分加上 Java 的内存模型部分进行说明，首先先来说说 Java 的内存管理机制。

### 说说 Java 的内存管理机制

和 C++ 相比，Java 的内存管理机制可谓是一大特色，程序员们不需要自己去写代码手动释放内存了，甚至你想自己干虚拟机都不给你干这个事情的机会（就是说，我们是没有办法自动触发 GC 的），虚拟机全权包办了 Java 的内存控制权力。这看起来挺美好的，不过也意味着，一旦虚拟机疏忽了（感觉不能赖虚拟机，毕竟虚拟机也不知道你能把程序写成那样啊……），发生了内存泄漏，问题都不好查，所以知道虚拟机到底是怎么管的内存就十分重要啦。

虚拟机对内存的管理，其实就是收拾哪些存放我们不会再用的对象的内存，把它们清了拿来放新的对象。所以它首先需要研究下以下几个问题：

- 这堆报废了的对象到底被放哪了？（Java 堆和方法区）
	- 5 个数据区域：[程序计数器](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/00-Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md#%E7%A8%8B%E5%BA%8F%E8%AE%A1%E6%95%B0%E5%99%A8)、[Java 虚拟机栈](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/00-Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md#java-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88)、[本地方法栈](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/00-Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md#%E6%9C%AC%E5%9C%B0%E6%96%B9%E6%B3%95%E6%A0%88)、[Java 堆](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/00-Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md#java-%E5%A0%86)、[方法区](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/00-Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md#%E6%96%B9%E6%B3%95%E5%8C%BA)。
- 这堆放报废对象的地方会不会内存泄漏？或者换一个洋气点的叫法，会不会 OOM？（[每个区的 OOM](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/01-OOM%E5%BC%82%E5%B8%B8.md#oom-%E5%BC%82%E5%B8%B8-outofmemoryerror)）
- 对象是咋被放到这些地方的？（[堆中对象的创建](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/00-Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md#%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%88%9B%E5%BB%BA%E9%81%87%E5%88%B0%E4%B8%80%E6%9D%A1-new-%E6%8C%87%E4%BB%A4%E6%97%B6)）
- 对象被安置好了之后虚拟机怎么再次找到它？（[堆中对象的访问](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/00-Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E8%AF%A6%E8%A7%A3.md#%E5%AF%B9%E8%B1%A1%E7%9A%84%E8%AE%BF%E9%97%AE)）

知道对象都放哪了，虚拟机就知道去哪里找报废的对象了，接下来就涉及到了 Java 的一大超级特色：垃圾收集（GC）了，垃圾收集，正如其名，就是把这些报废的对象给清了，腾出来地方放新对象，它主要关心以下几个事情：

- 哪些内存需要回收？
	- 放对象的地方需要垃圾回收：Java 堆和方法区。
- 什么时候回收？（[判断对象的生死](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E5%88%A4%E6%96%AD%E5%AF%B9%E8%B1%A1%E7%9A%84%E7%94%9F%E6%AD%BB)）
	- 判断对象报废了没的算法（重点）：[引用计数法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E7%AE%97%E6%B3%95) 和 [可达性分析法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E5%8F%AF%E8%BE%BE%E6%80%A7%E5%88%86%E6%9E%90%E7%AE%97%E6%B3%95%E4%B8%BB%E6%B5%81)。
- 如何回收？
  - GC 算法原理（[垃圾收集算法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E7%AE%97%E6%B3%95)）
  	- [基础：标记 - 清除算法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E5%9F%BA%E7%A1%80%E6%A0%87%E8%AE%B0---%E6%B8%85%E9%99%A4%E7%AE%97%E6%B3%95)
  	- [解决效率问题：复制算法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E8%A7%A3%E5%86%B3%E6%95%88%E7%8E%87%E9%97%AE%E9%A2%98%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95)
  	- [解决空间碎片问题：标记 - 整理算法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E8%A7%A3%E5%86%B3%E7%A9%BA%E9%97%B4%E7%A2%8E%E7%89%87%E9%97%AE%E9%A2%98%E6%A0%87%E8%AE%B0---%E6%95%B4%E7%90%86%E7%AE%97%E6%B3%95)
  	- [进化：分代收集算法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#%E8%BF%9B%E5%8C%96%E5%88%86%E4%BB%A3%E6%94%B6%E9%9B%86%E7%AE%97%E6%B3%95)
  - GC 算法的真正实现：
    - [7 个葫芦娃，哦不，垃圾收集器](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#7-%E4%B8%AA%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8)
    	- 新生代：Serial、ParNew、Parallel Scavenge
    	- 老年代：Serial Old、Parallel Old、CMS
    	- 全能：G1
    - [HotSpot 虚拟机如何高效实现 GC 算法](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/02-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86(GC).md#hotspot-%E4%B8%AD-gc-%E7%AE%97%E6%B3%95%E7%9A%84%E5%AE%9E%E7%8E%B0)

说完了对象是怎么被回收的，现在才算是把 Java 的内存管理机制需要用到的小零件给补全了。也就是说，Java 的内存管理流程应该是这样滴：

- 根据新对象是什么对象给对象找个地放
- 发现内存中没地放这个新对象了就进行 GC 清理出来点地方
- 真找不着地了就抛 OOM ……

虚拟机一般都用的是进化版的 GC 算法，也就是分代收集算法，也就是说，虚拟机 Java 堆中的内存是分为新生代和老年代的，那么给新对象找地方放的时候放哪呢？具体怎么放呢？放好了之后的对象会不会换个地呆呀？GC 什么时候进行？清理哪呢？……预知 Java 的内存管理机制的详情如何，请看：[Java 内存分配策略](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch1-Java%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6/03-Java%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%AD%96%E7%95%A5.md#java-%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%AD%96%E7%95%A5)。

到此为止，Java 的内存管理机制也就说的差不多了。现在，我们已经知道一个对象是如何在虚拟机的操控下，在内存中走一遭的了。可是首先，对象肯定是根据我们写的类创建的，那么我们写的类到底是如何变为内存中的对象的呢？而且，我们创建对象当然是为了执行它里面的方法呀，那么这个方法是怎么被执行的呢？想要回答这些问题，就需要我们研究一下 Java 虚拟机是如何执行我们的程序的了。

### 说说 Java 虚拟机程序执行

想要执行 Java 程序，必然要先将 Java 代码编译成字节码文件，也就是 Class 文件，这个编译的过程我们暂且不谈，主要说一下如果执行这个 Class 文件，所以首先我们要先来了解一下 Class 文件的组成结构。

在了解了组成结构之后，接下来需要考虑的事情是，我们该怎么把这个 .class 文件加载进内存，让它变成方法区（Java 8 后变为了 Metaspace 元空间）的一个 Class 对象。这一过程就叫做：类的加载。

类的加载说头可就多了，大家都喜欢揪着这问，其实主要就下面这 3 个过程：

- 类加载的时机：在程序第一次主动引用类的时候。
	- 什么是主动加载？
	- 什么是被动加载？
- 类加载的过程：加载 —— 验证 —— 准备 —— 解析 —— 初始化 —— 使用 —— 卸载
- 类加载器
	- 如何判断两个类 “相等”？
	- 类加载器的分类？
	- 什么双亲委派模型？
	- 什么是破坏双亲委派模型？
	- 如何自定义类加载器？

将类加载到内存之后，接下来就要考虑如何执行这个类中的方法了。我们知道 5 大内存区域中的虚拟机栈是服务与 Java 方法的内存模型，所以我们首先需要对虚拟机栈的栈帧结构有一定的了解，虚拟机栈的栈帧结构包括如下几个部分：

- 局部变量表
- 操作数栈 & 动态连接 & 方法返回地址

了解了方法执行时的内存模型，接下来就要考虑 Java 是如何根据我们写的代码知道我们到底想要调用哪个方法？即确定被调用的方法是哪一个，然后才是基于栈的解释执行（此时才真正的执行字节码，也就是方法才真正被执行了）。

- 方法调用字节码指令
- 方法调用
	- 解析调用
	- 分派调用
		- 静态分派（方法重载）
		- 动态分派（方法重写）
- 基于栈的解释执行

此外，还需要了解一下 Java 的动态类型语言支持。

### 说说虚拟机性能监控及故障处理



### 说说 JIT 优化

JIT，也就是即时编译，首先我们需要知道什么是 JIT？

之后我们需要了解 HotSpot 虚拟机内的即时编译器运作过程，即回答以下 5 个问题：

- 为什么要使用解释器与编译器并存的架构？
- 为什么虚拟机要实现两个不同的 JIT 编译器？
- 什么是虚拟机的分层编译？
- 如何判断热点代码，触发编译？
	- 什么是热点代码？
	- 什么是 “多次” 执行？
	- HotSpot 中每个方法的 2 个计数器
	- HotSpot 热点代码探测流程
- 热点代码编译的过程？

此外，我们知道 JIT 是要对代码的执行进行优化的，所以我们需要了解一下经典的优化技术：

- 公共子表达式消除【语言无关】
- 数组范围检查消除【语言相关】
- 方法内联【最重要】
- 逃逸分析【最前沿】

### 说说 Java 的内存模型

这部分内容主要与并发编程的内容相关，



## 项目推荐

对 JVM 相关知识的考查几乎成为 Java 面试的必备科目了，但是，就在简历上写个对 Java 虚拟机有一定了解，那极有可能被问到知识盲区呀！所以最好能在简历上就清晰明白的告诉人家我们都会啥，正如忘了在哪里看到的一个很有道理的话所言：简历就是我们准备面试的复习大纲。此时，倘若在简历上有那么一个项目可以用上 JVM 的相关的知识，那么在面试的时候，我们就可以基于这个项目开始我们的表演啦。

不过老实讲，Java 虚拟机相关的知识还真的不太好用在项目中，或者说不太好在项目中体现出来。这个问题我也想了好久，最后终于在看《深入理解 Java 虚拟机》第 9 章中的实战：自己动手实现远程执行功能时找到了答案，个人认为该实战中用到的通过修改字节码来替换 Java 代码中对于 System 类中方法的调用的技术酷极了！可是如果只是写这么一个小模块作为一个项目写在简历上又太小了点，所以，在有一天刷 LeetCode 时，突然灵光一闪，想到可以基于这个实战做一个在线 Java IDE，就有了这个项目：[基于 SpringBoot 的在线 Java IDE](https://github.com/TangBean/OnlineExecutor) 。

该项目基于 SpringBoot 实现了一个在线的 Java IDE，可以远程运行客户端发来的 Java 代码的 main 方法，并将程序的标准输出内容、运行时异常信息反馈给客户端，并且会对客户端发来的程序的执行时间进行限制。涉及了 Java 类文件的结构，Java 类加载器和 Java 类的热替换等 JVM 相关的技术，十分适合作为《深入理解 Java 虚拟机》这本书的一个实战内容，用来加深对该书内容的理解。