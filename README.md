# 《深入理解 Java 虚拟机》阅读笔记

本 repo 为《深入理解 Java 虚拟机 第2版》的阅读笔记，并对全书内容按照自己的理解进行了一定程度的整理。《深入理解 Java 虚拟机 第2版》原书主要分为了五个部分，这里仅对前四个部分进行讲解，第五部分（高效并发）整合进了另一个 repo：[Java 并发编程实战的阅读笔记](https://github.com/TangBean/Java-Concurrency-in-Practice) 中。

其中前两部分：Java 内存管理机制和 Java 虚拟机程序执行需要重点掌握，并且内容也是比较多的。本 repo 将原书中有关虚拟机性能监控及故障处理的部分单抽了出来，组成了本 repo 的第三部分。第四部分对应于原书的第四部分，程序编译与代码优化，不过仅对 Java 的运行期优化，也就是 JIT 时进行的优化进行了总结，编译器优化部分尚未进行深入研究。

**阅读方法：** 本 repo 的 README.md 从头读到尾就是一个虚拟机大部分知识点的框架，就像一颗搜索树一样，我们想要了解哪一部分知识，就从根节点开始搜索，直到找到我们想要了解的知识所在的叶节点或者子树。小伙伴们可以通过 README.md 回忆 JVM 相关的知识，遇到想不起来的点就点开相应的链接查看。这样像考试一样的学习方式，可以加深印象，记忆效果将远远好于盯着文字硬背。

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
- [01-JVM常见参数设置](./Ch3-虚拟机性能监控及故障处理/01-JVM常见参数设置.md)
- [02-虚拟机调优案例分析](./Ch3-虚拟机性能监控及故障处理/02-虚拟机调优案例分析.md)

### Java 程序运行优化

- [00-Java运行期优化](./Ch4-Java程序运行优化/00-Java运行期优化.md)



## “串一串” Java 虚拟机的知识点

本文将按照 Content 中给出的四个部分加上 Java 的内存模型部分进行说明，首先先来说说 Java 的内存管理机制。

- [说说 Java 的内存管理机制](#说说-java-的内存管理机制)
- [说说 Java 虚拟机程序执行](#说说-java-虚拟机程序执行)
- [说说虚拟机性能监控及故障处理](#说说虚拟机性能监控及故障处理)
- [说说 JIT 优化](#说说-jit-优化)
- [说说 Java 的内存模型 (JMM)](#说说-java-的内存模型-jmm)

### 说说 Java 的内存管理机制

和 C++ 相比，Java 的内存管理机制可谓是一大特色，程序员们不需要自己去写代码手动释放内存了，甚至你想自己干虚拟机都不给你干这个事情的机会（就是说，我们是没有办法自动触发 GC 的），虚拟机全权包办了 Java 的内存控制权力。这看起来挺美好的，不过也意味着，一旦虚拟机疏忽了（感觉不能赖虚拟机，毕竟虚拟机也不知道你能把程序写成那样啊……），发生了内存泄漏，问题都不好查，所以知道虚拟机到底是怎么管的内存就十分重要啦。

虚拟机对内存的管理，其实就是收拾那些存放我们不会再用的对象的内存，把它们清了拿来放新的对象。所以它首先需要研究下以下几个问题：

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

想要执行 Java 程序，必然要先将 Java 代码编译成字节码文件，也就是 Class 文件，这个编译的过程我们暂且不谈，主要说一下如何执行这个 Class 文件，所以首先我们要先来了解一下 [Class 文件的组成结构](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/00-Class%E6%96%87%E4%BB%B6%E7%9A%84%E7%BB%84%E6%88%90%E7%BB%93%E6%9E%84.md#class-%E6%96%87%E4%BB%B6%E7%9A%84%E7%BB%84%E6%88%90%E7%BB%93%E6%9E%84)。

在了解了组成结构之后，接下来需要考虑的事情是，我们该怎么把这个 .class 文件加载进内存，让它变成方法区（Java 8 后变为了 Metaspace 元空间）的一个 Class 对象呢？（[类的加载](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6)）。

虚拟机的类加载机制说头可就多了，大家都喜欢揪着这问，其实主要就下面这 3 个过程：

- [类加载的时机](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E7%B1%BB%E5%8A%A0%E8%BD%BD%E7%9A%84%E6%97%B6%E6%9C%BA)：在程序第一次主动引用类的时候。
  - 什么是主动引用和被动引用？
  - 什么是显式加载和隐式加载？
- [类的生命周期](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E7%B1%BB%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)：[加载](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E5%8A%A0%E8%BD%BD) —— [验证](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E9%AA%8C%E8%AF%81) —— [准备](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E5%87%86%E5%A4%87) —— [解析](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E8%A7%A3%E6%9E%90) —— [初始化](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E5%88%9D%E5%A7%8B%E5%8C%96) —— 使用 —— 卸载
- [类加载器](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8)
  - [如何判断两个类 “相等”？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E5%A6%82%E4%BD%95%E5%88%A4%E6%96%AD%E4%B8%A4%E4%B8%AA%E7%B1%BB-%E7%9B%B8%E7%AD%89)
  - [类加载器的分类？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E5%88%86%E7%B1%BB)
  - [什么双亲委派模型？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/01-%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md#%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B)
  - 破坏双亲委派模型？
  	- [实现 Java 类的热替换](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/%E9%99%84%E5%BD%950-%E5%AE%9E%E7%8E%B0Java%E7%B1%BB%E7%9A%84%E7%83%AD%E6%9B%BF%E6%8D%A2.md#%E5%AE%9E%E7%8E%B0-java-%E7%B1%BB%E7%9A%84%E7%83%AD%E6%9B%BF%E6%8D%A2)
  - 如何自定义类加载器？
  	- 需要保留双亲委派模型：`extends ClassLoader`，重写 `findClass()`
  	- 破坏双亲委派模型：直接重写 `loadClass()`

将类加载到内存之后，接下来就要考虑如何执行这个类中的方法了。我们知道 5 大内存区域中的 Java 虚拟机栈是服务与 Java 方法的内存模型，那么我们首先应该了解一下 [虚拟机栈的栈帧到底是怎样的结构](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_00-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88%E6%A0%88%E5%B8%A7%E7%BB%93%E6%9E%84.md#%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88%E6%A0%88%E5%B8%A7%E7%BB%93%E6%9E%84)，虚拟机栈的栈帧结构包括如下几个部分：

- [局部变量表](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_00-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88%E6%A0%88%E5%B8%A7%E7%BB%93%E6%9E%84.md#%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F%E8%A1%A8)（重要）
- [操作数栈](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_00-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88%E6%A0%88%E5%B8%A7%E7%BB%93%E6%9E%84.md#%E6%93%8D%E4%BD%9C%E6%95%B0%E6%A0%88) & [动态连接](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_00-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88%E6%A0%88%E5%B8%A7%E7%BB%93%E6%9E%84.md#%E5%8A%A8%E6%80%81%E8%BF%9E%E6%8E%A5) & [方法返回地址](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_00-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88%E6%A0%88%E5%B8%A7%E7%BB%93%E6%9E%84.md#%E6%96%B9%E6%B3%95%E8%BF%94%E5%9B%9E%E5%9C%B0%E5%9D%80)

了解了辅助方法执行的 Java 虚拟机栈的结构后，接下来就要考虑 Java 类中方法的调用了。就像将大象放进冰箱，方法的调用也不是上来就直接执行方法的，而是分为以下两个步骤：

- [方法调用](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_01-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.md#%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8)：确定被调用的方法是哪一个
- [基于栈的解释执行](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_02-%E5%9F%BA%E4%BA%8E%E6%A0%88%E7%9A%84%E5%AD%97%E8%8A%82%E7%A0%81%E8%A7%A3%E9%87%8A%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E.md#%E5%9F%BA%E4%BA%8E%E6%A0%88%E7%9A%84%E5%AD%97%E8%8A%82%E7%A0%81%E8%A7%A3%E9%87%8A%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E)：真正的执行方法的字节码

为什么还要加一个方法调用的步骤呢？因为一切方法调用都是在 Class 文件中以常量池中的符号引用存储的，这就导致了不是我们想要执行哪个方法就能立刻执行的，因为我们首先需要根据这个符号引用（其实就一字符串）找到我们想要执行的方法，而这一过程就叫做方法调用。当找到这个方法之后，我们才会开始执行这个方法，也就是基于栈的解释执行。

想要调用一个方法，我们先来看一下虚拟机中有哪些指令可以进行方法调用：[方法调用字节码指令](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_01-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.md#%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8%E5%AD%97%E8%8A%82%E7%A0%81%E6%8C%87%E4%BB%A4)。

这些字节码会触发不同的方法调用，总体来说，有以下几种：

- [解析调用](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_01-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.md#%E8%A7%A3%E6%9E%90%E8%B0%83%E7%94%A8)
- [分派调用](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_01-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.md#%E5%88%86%E6%B4%BE%E8%B0%83%E7%94%A8)（没有在解析调用中将符号引用转化为直接引用的方法就只能靠分派调用了）
  - [静态分派（方法重载）](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_01-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.md#%E9%9D%99%E6%80%81%E5%88%86%E6%B4%BE%E6%96%B9%E6%B3%95%E9%87%8D%E8%BD%BD)
  - [动态分派（方法重写）](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_01-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.md#%E5%8A%A8%E6%80%81%E5%88%86%E6%B4%BE%E6%96%B9%E6%B3%95%E9%87%8D%E5%86%99)

确定了要调用的方法具体是哪一个了之后，就可开始基于栈的解释执行了，这个时候，方法才真正的被执行。

此外，还需要了解一下 [Java 的动态类型语言支持](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch2-Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%A8%8B%E5%BA%8F%E6%89%A7%E8%A1%8C/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AD%97%E8%8A%82%E7%A0%81%E6%89%A7%E8%A1%8C%E5%BC%95%E6%93%8E_01-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8.md#%E5%8A%A8%E6%80%81%E7%B1%BB%E5%9E%8B%E8%AF%AD%E8%A8%80%E6%94%AF%E6%8C%81)。

### 说说虚拟机性能监控及故障处理

常用的 JDK 命令行工具：[JDK 命令行工具](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch3-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%80%A7%E8%83%BD%E7%9B%91%E6%8E%A7%E5%8F%8A%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86/00-%E5%B8%B8%E7%94%A8%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%80%A7%E8%83%BD%E7%9B%91%E6%8E%A7%E5%B7%A5%E5%85%B7.md#jdk-%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%B7%A5%E5%85%B7)。

JVM 常见的参数设置已经设置经验可见：[JVM 常见参数设置](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch3-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%80%A7%E8%83%BD%E7%9B%91%E6%8E%A7%E5%8F%8A%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86/01-JVM%E5%B8%B8%E8%A7%81%E5%8F%82%E6%95%B0%E8%AE%BE%E7%BD%AE.md#jvm-%E5%B8%B8%E8%A7%81%E5%8F%82%E6%95%B0%E8%AE%BE%E7%BD%AE)。

虚拟机调优案例分析可见：[虚拟机调优案例分析](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch3-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%80%A7%E8%83%BD%E7%9B%91%E6%8E%A7%E5%8F%8A%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86/02-%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%B0%83%E4%BC%98%E6%A1%88%E4%BE%8B%E5%88%86%E6%9E%90.md#%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%B0%83%E4%BC%98%E6%A1%88%E4%BE%8B%E5%88%86%E6%9E%90)。

### 说说 JIT 优化

JIT (Just In Time)，也就是即时编译，首先我们需要知道 [什么是 JIT？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E5%8D%B3%E6%97%B6%E7%BC%96%E8%AF%91)

然后，对于 [HotSpot 虚拟机内的即时编译器运作过程](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#hotspot-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%86%85%E7%9A%84%E5%8D%B3%E6%97%B6%E7%BC%96%E8%AF%91%E5%99%A8%E8%BF%90%E4%BD%9C%E8%BF%87%E7%A8%8B)，我们可以通过以下 5 个问题来研究它：

- [为什么要使用解释器与编译器并存的架构？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E4%BD%BF%E7%94%A8%E8%A7%A3%E9%87%8A%E5%99%A8%E4%B8%8E%E7%BC%96%E8%AF%91%E5%99%A8%E5%B9%B6%E5%AD%98%E7%9A%84%E6%9E%B6%E6%9E%84)
- [为什么虚拟机要实现两个不同的 JIT 编译器？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E4%B8%BA%E4%BB%80%E4%B9%88%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%A6%81%E5%AE%9E%E7%8E%B0%E4%B8%A4%E4%B8%AA%E4%B8%8D%E5%90%8C%E7%9A%84-jit-%E7%BC%96%E8%AF%91%E5%99%A8)
- [什么是虚拟机的分层编译？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E4%BB%80%E4%B9%88%E6%98%AF%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E5%88%86%E5%B1%82%E7%BC%96%E8%AF%91)
- [如何判断热点代码，触发编译？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E5%A6%82%E4%BD%95%E5%88%A4%E6%96%AD%E7%83%AD%E7%82%B9%E4%BB%A3%E7%A0%81%E8%A7%A6%E5%8F%91%E7%BC%96%E8%AF%91)
  - [什么是热点代码？(两种)](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E4%BB%80%E4%B9%88%E6%98%AF%E7%83%AD%E7%82%B9%E4%BB%A3%E7%A0%81)
  - [什么是 “多次” 执行？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E4%BB%80%E4%B9%88%E6%98%AF-%E5%A4%9A%E6%AC%A1-%E6%89%A7%E8%A1%8C)
  - HotSpot 采用的是基于计数器的热点探测方法，并且为了对两种热点代码进行探测，[每个方法有 2 个计数器](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#hotspot-%E4%B8%AD%E6%AF%8F%E4%B8%AA%E6%96%B9%E6%B3%95%E7%9A%84-2-%E4%B8%AA%E8%AE%A1%E6%95%B0%E5%99%A8)：
  	- 方法调用计数器
  	- 回边计数器
  - [HotSpot 热点代码探测流程](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#hotspot-%E7%83%AD%E7%82%B9%E4%BB%A3%E7%A0%81%E6%8E%A2%E6%B5%8B%E6%B5%81%E7%A8%8B)
- [热点代码编译的过程？](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E7%83%AD%E7%82%B9%E4%BB%A3%E7%A0%81%E7%BC%96%E8%AF%91%E7%9A%84%E8%BF%87%E7%A8%8B)

此外，JIT 并不是简单的将热点代码编译成机器码就收工的，它还会对代码的执行进行优化，主要有以下几种经典的优化技术：

- [公共子表达式消除【语言无关】](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E5%85%AC%E5%85%B1%E5%AD%90%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B6%88%E9%99%A4%E8%AF%AD%E8%A8%80%E6%97%A0%E5%85%B3)
- [数组范围检查消除【语言相关】](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E6%95%B0%E7%BB%84%E8%8C%83%E5%9B%B4%E6%A3%80%E6%9F%A5%E6%B6%88%E9%99%A4%E8%AF%AD%E8%A8%80%E7%9B%B8%E5%85%B3)
- [方法内联【最重要】](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E6%96%B9%E6%B3%95%E5%86%85%E8%81%94%E6%9C%80%E9%87%8D%E8%A6%81)
- [逃逸分析【最前沿】](https://github.com/TangBean/understanding-the-jvm/blob/master/Ch4-Java%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E4%BC%98%E5%8C%96/00-Java%E8%BF%90%E8%A1%8C%E6%9C%9F%E4%BC%98%E5%8C%96.md#%E9%80%83%E9%80%B8%E5%88%86%E6%9E%90%E6%9C%80%E5%89%8D%E6%B2%BF)

### 说说 Java 的内存模型 (JMM)

这部分内容主要与并发编程的内容相关，所以详细介绍会跳到另一个 repo：[Java-Concurrency-in-Practice](https://github.com/TangBean/Java-Concurrency-in-Practice)。

Java 的内存模型主要就是研究一个变量的值是怎么在主内存、线程的工作内存和 Java 线程（执行引擎）之间倒腾的。就是说虽然 Java 内存模型规定了所有变量都存储在主内存中，但是每个线程都有一个自己的工作内存，里面存着从主内存拷贝来的变量副本，Java 线程要对变量进行修改，都是先在自己的工作内存中进行，然后再把变化同步回主内存中去。

这样做是由于计算机的存储设备和处理器的运算速度有着几个数量级的差距，所以需要在主内存和 Java 线程间加入一个工作内存作为缓冲，但这也同时会导致主内存和工作内存间的缓存一致性问题，所以当两个工作内存中关于同一个变量的值发生冲突时，需要一定的访问规则来确定主内存以怎样的顺序同步这个变量，也就是说该听哪个工作内存的。而 Java 的内存模型的主要目标就是定义这个规则，即虚拟机如何将变量存储到内存或是从内存中取出的。

简单的来讲，就是掌握 [Java 内存模型中的 8 个原子操作](https://github.com/TangBean/Java-Concurrency-in-Practice/blob/master/Ch0-Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/00-Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md#java-%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E4%B8%AD%E7%9A%84-8-%E4%B8%AA%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C)，并且知道 [Java 内存间是如何通过这 8 个操作进行变量传递的](https://github.com/TangBean/Java-Concurrency-in-Practice/blob/master/Ch0-Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/00-Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md#java-%E5%86%85%E5%AD%98%E9%97%B4%E7%9A%84%E6%93%8D%E4%BD%9C)。

其实 Java 的内存模型就是围绕着在并发的过程中如何处理 [原子性](https://github.com/TangBean/Java-Concurrency-in-Practice/blob/master/Ch0-Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/00-Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md#%E5%8E%9F%E5%AD%90%E6%80%A7)、[可见性](https://github.com/TangBean/Java-Concurrency-in-Practice/blob/master/Ch0-Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/00-Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md#%E5%8F%AF%E8%A7%81%E6%80%A7)、[有序性](https://github.com/TangBean/Java-Concurrency-in-Practice/blob/master/Ch0-Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/00-Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md#%E6%9C%89%E5%BA%8F%E6%80%A7) 这 3 个特征建立的。同时 Java 除了可以依靠 volatile 和 synchronized 来保证有序性外，它自己本身还有一个 [Happens-Before 原则](https://github.com/TangBean/Java-Concurrency-in-Practice/blob/master/Ch0-Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%9F%BA%E7%A1%80/00-Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md#happens-before-%E8%A7%84%E5%88%99)，依靠这个原则，我们就可以判断并发环境下的两个操作是否可能存在冲突了。



## 项目推荐

对 JVM 相关知识的考查几乎成为 Java 面试的必备科目了，但是，就在简历上写个对 Java 虚拟机有一定了解，那极有可能被问到知识盲区呀！所以最好能在简历上就清晰明白的告诉人家我们都会啥，正如忘了在哪里看到的一个很有道理的话所言：简历就是我们准备面试的复习大纲。此时，倘若在简历上有那么一个项目可以用上 JVM 的相关的知识，那么在面试的时候，我们就可以基于这个项目开始我们的表演啦。

不过老实讲，Java 虚拟机相关的知识还真的不太好用在项目中，或者说不太好在项目中体现出来。这个问题我也想了好久，最后终于在看《深入理解 Java 虚拟机》第 9 章中的实战：自己动手实现远程执行功能时找到了答案，个人认为该实战中用到的通过修改字节码来替换 Java 代码中对于 System 类中方法的调用的技术酷极了！可是如果只是写这么一个小模块作为一个项目写在简历上又太小了点，所以，在有一天刷 LeetCode 时，突然灵光一闪，想到可以基于这个实战做一个在线 Java IDE，就有了这个项目：[基于 SpringBoot 的在线 Java IDE](https://github.com/TangBean/OnlineExecutor) 。

该项目基于 SpringBoot 实现了一个在线的 Java IDE，可以远程运行客户端发来的 Java 代码的 main 方法，并将程序的标准输出内容、运行时异常信息反馈给客户端，并且会对客户端发来的程序的执行时间进行限制。涉及了 Java 类文件的结构，Java 类加载器和 Java 类的热替换等 JVM 相关的技术，十分适合作为《深入理解 Java 虚拟机》这本书的一个实战内容，用来加深对该书内容的理解。
