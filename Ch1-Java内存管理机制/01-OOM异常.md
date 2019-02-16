# OOM 异常 (OutOfMemoryError)

<!-- TOC -->

- [OOM 异常 (OutOfMemoryError)](#oom-%E5%BC%82%E5%B8%B8-outofmemoryerror)
	- [Java 堆溢出](#java-%E5%A0%86%E6%BA%A2%E5%87%BA)
	- [Java 虚拟机栈和本地方法栈溢出](#java-%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%A0%88%E5%92%8C%E6%9C%AC%E5%9C%B0%E6%96%B9%E6%B3%95%E6%A0%88%E6%BA%A2%E5%87%BA)
	- [方法区和运行时常量池溢出](#%E6%96%B9%E6%B3%95%E5%8C%BA%E5%92%8C%E8%BF%90%E8%A1%8C%E6%97%B6%E5%B8%B8%E9%87%8F%E6%B1%A0%E6%BA%A2%E5%87%BA)
	- [直接内存溢出](#%E7%9B%B4%E6%8E%A5%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA)

<!-- /TOC -->

## Java 堆溢出

- 出现标志：`java.lang.OutOfMemoryError: Java heap space`
- 解决方法：
	- 先通过内存映像分析工具分析 Dump 出来的堆转储快照，确认内存中的对象是否是必要的，即分清楚是出现了内存泄漏还是内存溢出；
	- 如果是内存泄漏，通过工具查看泄漏对象到 GC Root 的引用链，定位出泄漏的位置；
	- 如果不存在泄漏，检查虚拟机堆参数（-Xmx 和 -Xms）是否可以调大，检查代码中是否有哪些对象的生命周期过长，尝试减少程序运行期的内存消耗。
- 虚拟机参数：
	- `-XX:HeapDumpOnOutOfMemoryError`：让虚拟机在出现内存泄漏异常时 Dump 出当前的内存堆转储快照用于事后分析。



## Java 虚拟机栈和本地方法栈溢出

- 单线程下，栈帧过大、虚拟机容量过小都不会导致 OutOfMemoryError，只会导致 StackOverflowError（栈会比内存先爆掉），一般多线程才会出现 OutOfMemoryError，因为线程本身要占用内存；
- 如果是多线程导致的 OutOfMemoryError，在不能减少线程数或更换 64 位虚拟机的情况，只能通过减少最大堆和减少栈容量来换取更多的线程；
	- 这个调节思路和 Java 堆出现 OOM 正好相反，Java 堆出现 OOM 要调大堆内存的设置值，而栈出现 OOM 反而要调小。



## 方法区和运行时常量池溢出

- 测试思路：产生大量的类去填满方法区，直到溢出；
- 在经常动态生成大量 Class 的应用中，如 Spring 框架（使用 CGLib 字节码技术），方法区溢出是一种常见的内存溢出，要特别注意类的回收状况。



## 直接内存溢出

- 出现特征：Heap Dump 文件中看不见明显异常，程序中直接或间接用了 NIO；
- 虚拟机参数：`-XX:MaxDirectMemorySize`，如果不指定，则和 `-Xmx` 一样。