# 第三章 垃圾回收与内存分配策略
---
## 概述
GC garbage collection 解决三个问题  
what when how

回收问题针对“堆“和”方法区“

## 对象判生死

### 1 引用计数算法

#### 算法原理 
每当一个引用指向对象，则count++，否则count--。如果count为0则代表对象已死。

#### 问题
很难解决对象之间相互引用的问题

### 2 可达性分析算法

#### 算法原理
通过一系列“GC ROOTS”对象作为起始点，向下搜索，搜索走过的路径称为“引用链”。当一个对象没有引用链与GC ROOTS相连，则说明此对象不可达。

#### JAVA中的GC ROOTS
1. 虚拟机栈(栈帧中的本地变量表)中引用的对象
2. 方法区中类静态属性引用的对象
3. 方法区中常量引用的对象
4. 本地方法栈中JNI（NATIVE方法）应用的对象
 
### 3 引用分类
1. **STRONG REFERENCE 强引用**		Object o = new Object 永远不会回收掉被引用的对象
2. **SOFT REFERENCE 软引用**		系统将要发生OOM前，将把这些对象列入范围进行二次回收
3. **WEAK REFERENCE 弱引用**		只能生存到下次回收
4. **PHANTOM REFERENCE 虚引用**	对对象的生存时间无影响，也无法通过虚引用获取对象实例。虚引用的目的是能在对象在被回收时收到一条系统通知

### 4 两次标记与finalize

对象不可达后，需要两次标记，才会真正被回收:  
1. 可达性分析发现不可达，则标记，进行第一次筛选，判断是否有必要执行finalize方法:如果**对象没有覆盖finalize方法或者finalize方法已经被调用过**，则没有必要执行；如果有必要执行，则将对象放入F-Queue队列中，虚拟机创建一个低优先级的Finalizer线程去执行。在finalize中如果对象被重新引用，则可以防止被回收。
2. 再次标记，则真正回收。

finalize方法是历史遗留产物，想法类似析构函数，但并无作用。**不建议使用**

### 5 回收方法区

永久代垃圾主要回收两部分： **废弃常量**和**无用的类**  
回收都是可能发生，不是必然发生

#### 废弃常量
没有任何对象引用该常量，并且也无任何对象引用到这个字面量

#### 无用的类
1. 该类的实例已经被回收
2. 加载该类的ClassLoader已经被回收
3. 该类对应的Class对象没有在任何地方被引用，并且无法通过反射访问该类的方法

## 垃圾回收算法

### 1 标记-清除算法（Mark-Sweep）
最基础的收集算法，其他算法是对其的改进。
#### 算法问题
1. **效率问题** 标记和清除效率都不高
2. **空间问题** 标记清除后产生大量内存碎片，分配大块内存时无法使用，提前触发gc

### 2 复制算法（copying）

#### 算法原理
将内存分为两份，每次gc将一份上的存活对象复制到另一份上，保持连续，并清空掉当前这一份；两份交替使用。新分配内存也无需考虑内存碎片的情况。但是代价太高，浪费一半内存。

#### 算法应用
HotSpot将内存分为Eden和两个Survivor，比例为8：1：1。每次新生代容量占90%，只有10%被浪费。
当survivor的空间不足时，需要依赖老年代Older进行分配担保(Handle Promotion)

### 3 标记-整理算法（Mark-Compact）

#### 算法原理
标记并清除后，将存货的内存向一端移动、整理，从而避免了内存碎片的问题。

#### 算法意义
对于老年代的内存，使用复制算法，由于内存存活率很高，会导致频繁复制，并且使用分配担保

### 4 分代收集算法(Generation Collection)
现在虚拟机主流回收算法
### 算法原理
把JAVA堆分为新生代和老年代，对于新生代使用Copying算法，对于老年代使用Mark-Compact算法

## HotSpot算法实现（待回看）

### 1 枚举根节点

1. 寻找GC ROOTS，如果通过遍历方式查找，很费时间
2. 可达性分析必须在确保一致性的快照内进行，以保证准确性，因此需要STOP THE WORLD
3. 主流JVM通过准确式GC，不需要遍历就，在加载类时就将引用的偏移信息记录下来，方便使用

### 2 安全点

在何时进行阻塞，调起GC？ 安全点
gc线程置flag = true，线程主动查看flag并主动阻塞

### 3 安全区域

如果线程进入安全区域，则阻塞自己，确定自己完成gc后离开安全区域。

## 垃圾回收器

### 1 Serial收集器
复制算法
#### 使用场景
Client模式的虚拟机

### 2 ParNew收集器
多线程版的Serial，分线程收集新生代的内存，可以与CMS配合使用，复制算法
多核情况下表现好，单核双核表现不一定比Serial好

#### 使用场景
Server模式的虚拟机

	tips
	- 并发Concurrent
	- 并行Parallel

	- 解释一：并行是指两个或者多个事件在同一时刻发生；而并发是指两个或多个事件在同一时间间隔发生。
	- 解释二：并行是在不同实体上的多个事件，并发是在同一实体上的多个事件。
	- 解释三：在一台处理器上“同时”处理多个任务，在多台处理器上同时处理多个任务。如hadoop分布式集群

### 3 Parallel Scavenge收集器
复制算法，并行收集，与ParNew区别： 不关注gc stw时间，而是关注吞吐量，因此适合无用户交互的计算任务
不可以与CMS配合使用
### 4 Serial Old收集器
Serial的老年代版本，标记-整理算法

### 5 Parallel Old收集器
Parallel Scavenge的老年代版本，标记-整理算法

### 6 CMS收集器
Concurrent Mark Sweep 以获取最短回收停顿时间为目标的收集器

标记清除算法，分为四步：  

	1. 初始标记
	2. 并发标记
	3. 重新标记
	4. 并发清除

其中，初始标记和重新标记需要STW，但是1和3相对于2和4时间很短

**初始标记**只是标记下能与GC ROOTS直接关联的对象  
**并发标记**是进行GC ROOTS TRACING的过程  
**重新标记**是为了**修正****并发标记期间**因为程序运行导致标记产生变动的对象的**标记**

#### 缺点
1. 并发的回收线程占用CPU
2. 4阶段清除时，产生的垃圾为“浮动垃圾”，需要在下个GC才能处理，因此GC需要提前，不能再堆满了才GC
3. 标记清除算法，导致有内存碎片

### 7 G1收集器（待回看）
Garbage First， 目标替代CMS 

#### 特点
1. 并行与并发
2. 分代收集
3. 空间整合
4. 可预测的停顿

### 8 GC日志

#### GC种类
- GC
- FULL GC
 
#### GC位置
- DefNew : Default New Generation
- Tenured
- Perm
- ParNew : Parallel New Generation
- PSYoungGen : Parallel Scavenge Young Generation

## 内存分配和回收策略

### 1 对象优先在Eden区分配
如果没有足够空间，则发起一次Minor GC

	-XX:+PrintGCDetail参数 可以将GC日志打印出来
	
	-Xmn2g：设置年轻代大小为2G。整个JVM内存大小=年轻代大小 + 年老代大小 + 持久代大小。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。
	
	-Xss128k：设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。
	
	-XX:SurvivorRadio=8 表示Eden和一个Survivor的比例为8：1

### 2 大对象直接进入老年代
大对象是指需要大量连续内存空间的对象，常常为数组、长字符串等

	-XX:PretenureSizeThreshold 设置一个大小，大于该值得对象直接分配在老年代，避免Eden和Survivor区之间发生大量的内存复制

### 3 长期存活的对象将进入老年代
对象年龄计数器，每经过一次Minor GC，对象年龄加一，当超过15（默认）时，从Survivor区移到tenure老年代  

	-XX:MaxTenuringThreshold 默认为15，age大于此值即可进入老年代 e.g. 此值设置为1，需要在第二次GC才能进入老年代

### 4 动态对象年龄判定
如果Survivor区中相同年龄的对象大小总和大于Survivor空间的一半，则大于等于该年龄的对象直接进入老年代，无需等到MaxTenuringThreshold才进入

### 5 空间分配担保
新生代使用复制收集算法，当大对象在Minor GC 后无法回收进Survivor中时，需要由Tenure来分配担保。因此为了完成这个担保，需要在Minor GC前确保Tenure中有足够的空间来容纳GC后未被回收的部分。
未被回收的部分有两种估算方式：

1. 新生代对象的总大小
2. 新生代历次晋升的平均大小

因此，在JDK6 UPDATE24后的版本中，在Minor GC前，进行判断：  
**（Tenure_available > New_all） || (Tenure_available > New_Average_Promotion)**  
条件成立，则进行Minor GC ，否则进行Full GC




