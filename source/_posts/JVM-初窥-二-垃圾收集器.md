---
title: JVM 初窥(二) 垃圾回收
date: 2018-06-10 14:10:57
tags: [JVM, Java虚拟机]
---
说起 Java 和 C++ 语言的不同之处，不得不提一下垃圾回收技术。
由于垃圾回收器的存在，在编写 Java 代码时，几乎不用去考虑对象的销毁问题，垃圾回收器会在合适的时间，自动回收不再存活的对象。
所以，垃圾回收器在工作时，需要思考以下三个问题。
> 1.哪些内存需要回收？
> 2.什么时候回收？
> 3.如何回收？


## 哪些内存需要回收
这个问题放到 Java 堆中，则变成了“哪些对象需要回收”。垃圾回收器回收的是 Java 对象，它需要确定堆中哪些对象还存活，而哪些对象已经死去（不可能再被人和途径使用的对象）。死去的对象是垃圾回收器重点关注的对象。

### 1.引用计数法
简单而言，引用计算法就是给对象添加一个引用计数器，每当该对象被其他地方引用，则给引用计数器加 1，当不再引用时，则引用计数器减 1，当计数器为 0 时，则表示该对象不会再被使用。
那虚拟机中采用的是这种算法吗，读者可以先自行思考一下。
我们来看一下如下这段代码：
```java
/**
 * 采用引用计数法的GC
 *
 * @author hinoah
 * @date 2018/6/10 下午4:18
 */
public class GcObject {

    public Object instance;

    public static void main(String[] args) {
        GcObject o1 = new GcObject();
        GcObject o2 = new GcObject();
        o1.instance = o2;
        o2.instance = o1;

        o1 = null;
        o2 = null;

        System.gc();
    }
}
```
试问，在20行进行 gc 时，o1、o2对象会被回收吗？
答案是，会被回收掉。
按照引用计数法的原理，对象o1、o2读存有对方的引用，这时并不会判定o1、o2为不可用的对象，所以不会回收。但实际上这两个对象已经不再可被访问已经不可能再被访问到，所以说这俩对象被回收掉是更合理的做法。
由于引用计数器无法解决（或者说很难解决）对象之间循环引用的问题，所以主流 JVM 虚拟机都未采用这种算法作为垃圾回收算法。

### 2.可达性分析算法
JVM 虚拟机真正采用的垃圾回收算法。基本思路是通过一些列“GC Roots”的对象作为起点，从这些对象向下搜索，搜索走过的路径被称为引用链。当从“GC Roots”没有任何引用链指向某个对象时，则证明该对象不可用，可以被回收。如下图所示，1、2、3、4对象被包含在 GC Roots 引用链中，而 5、6对象虽互有关联，但他们都不在 GC Roots 的引用链中，所以他们都会被判定为可回收的对象。

![可达性分析算法](http://p8e6biisu.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-06-10%20%E4%B8%8B%E5%8D%884.53.23.png)

##### 可作为GC Roots的对象包含以下几种：
- 虚拟机栈中引用的对象
- 方法区中类静态属性和常量引用的对象
- 本地方法栈中引用的对象 

## 什么时候回收
虚拟机的垃圾回收是一个典型的后台守护（Daemon）进程，可以理解为一种优先级较低的线程，该线程等待操作系统分配时间片运行。垃圾回收贯穿于虚拟机的整个运行期，随虚拟机的启动或停止而存亡。
也可以通过调用 System.gc() 主动执行垃圾回收方法。
```java
System.gc();
```
## 如何回收
垃圾回收比较常见的策略有如下几种

### 1.标记-清除算法
最基础的一种回收算法。首先根据可达性分析标记处所有需要回收的对象，然后再统一回收所有被标记的对象。
大致过程如下图所示：
![标记-清除算法](http://p8e6biisu.bkt.clouddn.com/%E6%A0%87%E8%AE%B0%E6%B8%85%E9%99%A4%E7%AE%97%E6%B3%95.jpeg)
这种算法有一些不足，一是标记和清除两个过程的效率问题，都比较低；二是空间问题，标记清除后会产生大量不连续的内存碎片，内存碎片太多会造成后面需要分配一个大对象时找不到足够的连续内存空间，不得不触发另一次垃圾回收动作。
### 2.复制算法
复制算法将内存空间分为大小相等的两块，每次只使用其中的一块。当该块内存空间消耗殆尽时，将该块还存活的对象一次性复制到另一半内存块中，并将原先已经满的内存块里的对象一次性清楚掉。这解决了标记算法的效率问题，同时也就不会有内存碎片的问题。但是这种方式的代价就是可用内存降为原先的一半，这种方式还是非常浪费内存的。
![复制算法](http://p8e6biisu.bkt.clouddn.com/%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95.jpeg)
但在大多数情况下，对象的生命周期都是非常短暂的，所以我们没有必要把一半的内存空间都作为备用空间，而是将内存划分为一块较大的 Eden 空间和两块较小的 Survivor 空间（S1 和 S2），每次只在 Eden 和 S1 中分配对象，在进行垃圾回收时，将存活的对象一次性复制至 S2 区，然后清理 Eden 和 S1 的内存空间。在绝大多数情况下，S2 都足够容纳 Eden 和 S1 存活下来的对象，但这并不是绝对的。在 S2 内存不足时，再去向老年代申请新的内存。
在 HotSpot 虚拟机中，Eden 和 Survivor 的默认比例为 8:1，这样只有十分之一的内存会被浪费。

### 3.标记-整理算法
复制算法中，老年代为内存的复制提供了分配担保，在 S2 不足时可以由老年代来继续存储存活的对象。但复制算法不太适合老年代，这是为什么呢？老年代的对象存活率相对比较高，如果这时进行复制效率不会特别高，需要复制的对象也相对年轻代更多；同时，老年代以外再没有多余的空间可以进行分配担保，除非浪费一半的空间用作两个块之间的复制。
老年代一般采用**标记-整理算法**，其中标记与标记-清除算法的过程相同，但清除时不是简单的清除掉被标记的对象，而是将所有存活的对象向一端移动，这样就不会存在内存碎片。
![标记-整理算法](http://p8e6biisu.bkt.clouddn.com/%E6%A0%87%E8%AE%B0%E6%95%B4%E7%90%86%E7%AE%97%E6%B3%95.jpeg)
### 4.分代回收算法
分代就是将内存区域按照对象的存货周期进行划分划分，一般分为年轻代和年老代，然后在不同的分代中按照不同方式进行垃圾回收。例如在年轻代中，大多数对象生命周期很短，只存活少量对象，则采用复制算法，效率高，需要多余的空间较少；在年老代，对象不容易消亡，没有额外的空间用于复制，则一般采用标记-整理算法进行垃圾回收。   

### 附录：HotSpot 的垃圾收集器简要介绍
##### Serial 收集器
单线程，会 Stop The World
##### ParNew 收集器
Serial 的多线程版本（仍然会Stop The World），可配合 CMS 收集器使用
##### Parallel Scanvenge 收集器
新生代收集器，使用复制算法，多线程并行，吞吐量优先
##### Serial Old 收集器
Serial 的年老代版本，采用标记-整理算法，单线程
##### Parallel Old 收集器
Parallel Scanvenge 的年老代版本，多线程
##### CMS 收集器
基于标记-清除算法，并发标记、并发清除，降低系统停顿时间
##### G1 收集器
并行并发，分代收集，空间整合，可预测的停顿



