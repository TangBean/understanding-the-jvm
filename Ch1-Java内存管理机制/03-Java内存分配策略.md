# Java 内存分配策略

![内存分配与回收策略.png](./pic/内存分配与回收策略.png)



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

我们知道，新生代采用的是复制算法清理内存，每一次 Minor GC，虚拟机会将 Eden 区和其中一块 Survivor 区的存活对象复制到另一块 Survivor 区，但**当出现大量对象在一次 Minor GC 后仍然存活的情况时，Survivor 区可能容纳不下这么多对象，此时，就需要老年代进行分配担保，即将 Survivor 无法容纳的对象直接进入老年代。**

这么做有一个前提，就是老年代得装得下这么多对象。可是在一次 GC 操作前，虚拟机并不知道到底会有多少对象存活，所以空间分配担保有这样一个判断流程：

- 发生 Minor GC 前，虚拟机先检查老年代的最大可用连续空间是否大于新生代所有对象的总空间；
	- 如果大于，Minor GC 一定是安全的；
	- 如果小于，虚拟机会查看 HandlePromotionFailure 参数，看看是否允许担保失败；
		- 允许失败：尝试着进行一次 Minor GC；
		- 不允许失败：进行一次 Full GC；
- 不过 JDK 6 Update 24 后，HandlePromotionFailure 参数就没有用了，规则变为只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行 Minor GC，否则将进行 Full GC。