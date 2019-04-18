# Java 内存分配策略

![内存分配与回收策略.png](./pic/内存分配与回收策略.png)

<!-- TOC -->

- [Java 内存分配策略](#java-内存分配策略)
    - [优先在 Eden 区分配](#优先在-eden-区分配)
    - [大对象直接进入老年代](#大对象直接进入老年代)
    - [长期存活的对象将进入老年代](#长期存活的对象将进入老年代)
    - [空间分配担保](#空间分配担保)

<!-- /TOC -->

> **新生代和老年代的 GC 操作**
>
> - 新生代 GC 操作：Minor GC
> 	- 发生的非常频繁，速度较块。
> - 老年代 GC 操作：Full GC / Major GC
> 	- 经常伴随着至少一次的 Minor GC；
> 	- 速度一般比 Minor GC 慢上 10 倍以上。



## 优先在 Eden 区分配

- Eden 空间不够将会触发一次 Minor GC；
- 虚拟机参数：
	- `-Xmx`：Java 堆的最大值；
	- `-Xms`：Java 堆的最小值；
	- `-Xmn`：新生代大小；
	- `-XX:SurvivorRatio=8`：Eden 区 / Survivor 区 = 8 : 1



## 大对象直接进入老年代

- **大对象定义：** 需要大量连续内存空间的 Java 对象。例如那种很长的字符串或者数组。
- **设置对象直接进入老年代大小限制：**
	- `-XX:PretenureSizeThreshold`：单位是字节；
		- 只对 Serial 和 ParNew 两款收集器有效。
	- **目的：** 因为新生代采用的是复制算法收集垃圾，大对象直接进入老年代可以避免在 Eden 区和 Survivor 区发生大量的内存复制。



## 长期存活的对象将进入老年代

- **固定对象年龄判定：** 虚拟机给每个对象定义一个年龄计数器，对象每在 Survivor 中熬过一次 Minor GC，年龄 +1，达到 `-XX:MaxTenuringThreshold` 设定值后，会被晋升到老年代，`-XX:MaxTenuringThreshold` 默认为 15；
- **动态对象年龄判定：** Survivor 中有相同年龄的对象的空间总和大于 Survivor 空间的一半，那么，年龄大于或等于该年龄的对象直接晋升到老年代。



## 空间分配担保

我们知道，新生代采用的是复制算法清理内存，每一次 Minor GC，虚拟机会将 Eden 区和其中一块 Survivor 区的存活对象复制到另一块 Survivor 区，但 **当出现大量对象在一次 Minor GC 后仍然存活的情况时，Survivor 区可能容纳不下这么多对象，此时，就需要老年代进行分配担保，即将 Survivor 无法容纳的对象直接进入老年代。**

这么做有一个前提，就是老年代得装得下这么多对象。可是在一次 GC 操作前，虚拟机并不知道到底会有多少对象存活，所以空间分配担保有这样一个判断流程：

- 发生 Minor GC 前，虚拟机先检查老年代的最大可用连续空间是否大于新生代所有对象的总空间；
	- 如果大于，Minor GC 一定是安全的；
	- 如果小于，虚拟机会查看 HandlePromotionFailure 参数，看看是否允许担保失败；
		- 允许失败：尝试着进行一次 Minor GC；
		- 不允许失败：进行一次 Full GC；
- 不过 JDK 6 Update 24 后，HandlePromotionFailure 参数就没有用了，规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行 Minor GC，否则将进行 Full GC。



## Metaspace 元空间与 PermGen 永久代

Java 8 彻底将永久代 (PermGen) 移除出了 HotSpot JVM，将其原有的数据迁移至 Java Heap 或 Metaspace。

**移除 PermGen 的原因：**

- PermGen 内存经常会溢出，引发恼人的 java.lang.OutOfMemoryError: PermGen，因此 JVM 的开发者希望这一块内存可以更灵活地被管理，不要再经常出现这样的 OOM；
- 移除 PermGen 可以促进 HotSpot JVM 与 JRockit VM 的融合，因为 JRockit 没有永久代。

**移除 PermGen 后，方法区和字符串常量的位置：**

- 方法区：移至 Metaspace；
- 字符串常量：移至 Java Heap。

**Metaspace 的位置：** 本地堆内存(native heap)。

**Metaspace 的优点：** 永久代 OOM 问题将不复存在，因为默认的类的元数据分配只受本地内存大小的限制，也就是说本地内存剩余多少，理论上 Metaspace 就可以有多大；

**JVM参数：**

- `-XX:MetaspaceSize`：分配给类元数据空间（以字节计）的初始大小，为估计值。MetaspaceSize的值设置的过大会延长垃圾回收时间。垃圾回收过后，引起下一次垃圾回收的类元数据空间的大小可能会变大。
- `-XX:MaxMetaspaceSize`：分配给类元数据空间的最大值，超过此值就会触发Full GC，取决于系统内存的大小。JVM会动态地改变此值。
- `-XX:MinMetaspaceFreeRatio`：一次GC以后，为了避免增加元数据空间的大小，空闲的类元数据的容量的最小比例，不够就会导致垃圾回收。
- `-XX:MaxMetaspaceFreeRatio`：一次GC以后，为了避免增加元数据空间的大小，空闲的类元数据的容量的最大比例，不够就会导致垃圾回收。