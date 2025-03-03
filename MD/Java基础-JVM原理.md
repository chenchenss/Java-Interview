## JVM原理
### Java内存区域的分配
JVM虚拟机内存模型实现规范：

![](https://github.com/xbox1994/Java-Interview/raw/master/images/JVM规范.png)


按线程是否共享分为以下区域：

所有线程共享的数据区：

1.	方法区（JVM规范中的一部分，不是实际的实现）: 存储每一个类的结构信息（运行时常量池、静态变量、方法数据、构造函数和普通方法的字节码、JIT编译后的代码)，没有要求使用垃圾回收因为回收效率太低。（运行时常量池：存放编译器生成的各种字面量和符号引用，在类加载后放到运行时常量池中）
2.	堆区: 最大的一块区域，是大部分类实例、对象、数组分配内存的区域，没有限制只能将对象分配在堆，所以出现逃逸分析的技术

每个线程都会有一块私有的数据区： 

1.	虚拟机栈: 虚拟机栈与线程同时创建，每个方法在执行时在其中创建一个栈帧，用于存储局部变量、操作数栈、动态链接、方法返回地址。正常调用完成后恢复调用者的局部变量表、操作数栈、递增程序计数器来跳过刚才执行的指令，或抛出异常不将返回值返回给调用者
2.	本地方法栈: 功能与虚拟机栈相同，为native方法服务
3.	pc寄存器: 任意时刻线程只会执行一个方法的代码，如果不是native的，就存放当前正在执行的字节码指令的地址，如果是native，则是undefined

以HotSpot虚拟机实现为例，Java8中内存区域如下：

![](https://github.com/xbox1994/Java-Interview/raw/master/images/JVM1.8.png)

与规范中的区别：

1.  直接内存：非Java标准，是JVM以外的本地内存，在Java4出现的NIO中，为了防止Java堆和Native堆之间往复的数据复制带来的性能损耗，此后NIO可以使用Native的方式直接在Native堆分配内存。JDK中有一种基于通道（Channel）和缓冲区（Buffer）的内存分配方式，将由C语言实现的native函数库分配在直接内存中，用存储在JVM堆中的DirectByteBuffer来引用。
2.	元数据区（方法区的实现）：Java7以及之前是使用的永久代来实现方法区，大小是在启动时固定的。Java8中用元空间替代了永久代，元空间并不在虚拟机中，而是使用本地内存，并且大小可以是自动增长的，这样减少了OOM的可能性。元空间存储JIT即时编译后的native代码，可能还存在短指针数据区CCS
3.	堆区: Java7之后运行时常量池从方法区移到这里，为Java8移除永久带的做好准备

### Java对象不都是分配在堆上
#### 逃逸分析
逃逸是指在某个方法之内创建的对象除了在方法体之内被引用之外，还在方法体之外被其它变量引用到；这样带来的后果是在该方法执行完毕之后，该方法中创建的对象将无法被GC回收。由于其被其它变量引用，由于无法回收，即称为逃逸。

逃逸分析技术可以分析出某个对象是否永远只在某个方法、线程的范围内，并没有“逃逸”出这个范围，逃逸分析的一个结果就是对于某些未逃逸对象可以直接在栈上分配提高对象分配回收效率，对象占用的空间会随栈帧的出栈而销毁。

### 类加载机制
#### 加载过程

1. 加载（获取来自任意来源的字节流并转换成运行时数据结构，生成Class对象）
2. 验证（验证字节流信息符合当前虚拟机的要求，防止被篡改过的字节码危害JVM安全）
3. 准备（为类变量分配内存并设置初始值）
4. 解析（将常量池的符号引用替换为直接引用，符号引用是用一组符号来描述所引用的目标，直接引用是指向目标的指针）
5. 初始化（执行类构造器、类变量赋值、静态语句块）

#### 类加载器
启动类加载器：用C++语言实现，是虚拟机自身的一部分，它负责将 <JAVA_HOME>/lib路径下的核心类库，无法被Java程序直接引用  
扩展类加载器：用Java语言实现，它负责加载<JAVA_HOME>/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类库，开发者可以直接使用  
系统类加载器：用Java语言实现，它负责加载系统类路径ClassPath指定路径下的类库，开发者可以直接使用
https://blog.csdn.net/t194978/article/details/125984471
#### 双亲委派
定义：如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载，这就是**双亲委派模式**。

优点：采用双亲委派模式的是好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系，通过这种层级关可以避免类的重复加载，当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次。其次防止恶意覆盖Java核心API。

三次大型破坏双亲委派模式的事件：

1. 在双亲委派模式出来之前，用户继承ClassLoader就是为了重写loadClass方法，但双亲委派模式需要这个方法，所以1.2之后添加了findClass供以后的用户重写
2. 如果基础类要调回用户的代码，如JNDI/JDBC需要调用ClassPath下的自己的代码来进行资源管理，Java团队添加了一个线程上下文加载器，如果该加载器没有被设置过，那么就默认是应用程序类加载器
3. 为了实现代码热替换，OSGi是为了实现自己的类加载逻辑，用平级查找的逻辑替换掉了向下传递的逻辑。但其实可以不破坏双亲委派逻辑而是自定义类加载器来达到代码热替换。比如[这篇文章](https://www.cnblogs.com/pfxiong/p/4070462.html)

### 内存分配（堆上的内存分配）
![](https://github.com/xbox1994/Java-Interview/raw/master/images/堆的内存分配.png)
#### 新生代
##### 进入条件
优先选择在新生代的Eden区被分配。
#### 老年代
##### 进入条件
1. 大对象，-XX:PretenureSizeThreshold 大于这个参数的对象直接在老年代分配，来避免新生代GC以及分配担保机制和Eden与Survivor之间的复制
2. 经过第一次Minor GC仍然存在，能被Survivor容纳，就会被移动到Survivor中，此时年龄为1，当年龄大于预设值就进入老年代  
3. 如果Survivor中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于等于该年龄的对象进入老年代  
4. 如果Survivor空间无法容纳新生代中Minor GC之后还存活的对象

### GC回收机制
#### 回收对象
不可达对象：通过一系列的GC Roots的对象作为起点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时则此对象是不可用的。  
GC Roots包括：虚拟机栈中引用的对象、方法区中类静态属性引用的对象、方法区中常量引用的对象、本地方法栈中JNI（Native方法）引用的对象。

彻底死亡条件：  
条件1：通过GC Roots作为起点的向下搜索形成引用链，没有搜到该对象，这是第一次标记。  
条件2：在finalize方法中没有逃脱回收（将自身被其他对象引用），这是第一次标记的清理。

#### 如何回收
新生代因为每次GC都有大批对象死去，只需要付出少量存活对象的复制成本且无碎片所以使用“复制算法”  
老年代因为存活率高、没有分配担保空间，所以使用“标记-清理”或者“标记-整理”算法

复制算法：将可用内存按容量划分为Eden、from survivor、to survivor，分配的时候使用Eden和一个survivor，Minor GC后将存活的对象复制到另一个survivor，然后将原来已使用的内存一次清理掉。这样没有内存碎片。  
标记-清除：首先标记出所有需要回收的对象，标记完成后统一回收被标记的对象。会产生大量碎片，导致无法分配大对象从而导致频繁GC。  
标记-整理：首先标记出所有需要回收的对象，让所有存活的对象向一端移动。

#### Minor GC条件
当Eden区空间不足以继续分配对象，发起Minor GC。

#### Full GC条件
1. 调用System.gc时，系统建议执行Full GC，但是不必然执行
2. 老年代空间不足（通过Minor GC后进入老年代的大小大于老年代的可用内存）
3. 方法区空间不足

## 垃圾收集器
### 串行收集器
串行收集器Serial是最古老的收集器，只使用一个线程去回收，可能会产生较长的停顿

新生代使用Serial收集器`复制`算法、老年代使用Serial Old`标记-整理`算法

参数：`-XX:+UseSerialGC`，默认开启`-XX:+UseSerialOldGC`

### 并行收集器
并行收集器Parallel关注**可控的吞吐量**，能精确地控制吞吐量与最大停顿时间是该收集器最大的特点，也是1.8的Server模式的默认收集器，使用多线程收集。ParNew垃圾收集器是Serial收集器的多线程版本。

新生代`复制`算法、老年代`标记-整理`算法

参数：`-XX:+UseParallelGC`，默认开启`-XX:+UseParallelOldGC`

### 并发收集器
并发收集器CMS是以**最短停顿时间**为目标的收集器。G1关注能在大内存的前提下精确控制**停顿时间**且垃圾回收效率高。

CMS针对老年代，有初始标记、并发标记、重新标记、并发清除四个过程，标记阶段会Stop The World，使用`标记-清除`算法，所以会产生内存碎片。

参数：`-XX:+UseConcMarkSweepGC`，默认开启`-XX:+UseParNewGC`

G1将堆划分为多个大小固定的独立区域，根据每次允许的收集时间优先回收垃圾最多的区域，使用`标记-整理`算法，是1.9的Server模式的默认收集器

参数：`-XX:+UseG1GC`

## Stop The World
Java中Stop-The-World机制简称STW，是在执行垃圾收集算法时，Java应用程序的其他所有线程都被挂起。Java中一种全局暂停现象，全局停顿，所有Java代码停止，native代码可以执行，但不能与JVM交互

STW总会发生，不管是新生代还是老年代，比如CMS在初始标记和重复标记阶段会停顿，G1在初始标记阶段也会停顿，所以并不是选择了一款停顿时间低的垃圾收集器就可以避免STW的，我们只能尽量去减少STW的时间。

那么为什么一定要STW？因为在定位堆中的对象时JVM会记录下对所有对象的引用，如果在定位对象过程中，有新的对象被分配或者刚记录下的对象突然变得无法访问，就会导致一些问题，比如部分对象无法被回收，更严重的是如果GC期间分配的一个GC Root对象引用了准备被回收的对象，那么该对象就会被错误地回收。

## [Java内存模型](https://mp.weixin.qq.com/s/ME_rVwhstQ7FGLPVcfpugQ)
定义：JMM是一种规范，目的是解决由于多线程通过共享内存进行通信时，存在的本地内存数据不一致、编译器会对代码指令重排序、处理器会对代码乱序执行等带来的问题。目的是保证并发编程场景中的原子性、可见性和有序性

实现：volatile、synchronized、final、concurrent包等。其实这些就是Java内存模型封装了底层的实现后提供给程序员使用的一些关键字

![](https://github.com/xbox1994/Java-Interview/raw/master/images/j12.jpg)

主内存：所有变量都保存在主内存中  
工作内存：每个线程的独立内存，保存了该线程使用到的变量的主内存副本拷贝，线程对变量的操作必须在工作内存中进行

每个线程都有自己的本地内存共享副本，如果A线程要更新主内存还要让B线程获取更新后的变量，那么需要：

1. 将本地内存A中更新共享变量
2. 将更新的共享变量刷新到主内存中
3. 线程B从主内存更新最新的共享变量

## [happens-before](https://www.cnblogs.com/chenssy/p/6393321.html)
我们无法就所有场景来规定某个线程修改的变量何时对其他线程可见，但是我们可以指定某些规则，这规则就是happens-before。特别关注在多线程之间的内存可见性。

它是判断数据是否存在竞争、线程是否安全的主要依据，依靠这个原则，我们解决在并发环境下两操作之间是否可能存在冲突的所有问题。

1. 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作；
2. 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作；
3. volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作；
4. 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C；
5. 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作；
6. 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生；
7. 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行；
8. 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始；

## JVM调优
前提：在进行GC优化之前，需要确认项目的架构和代码等已经没有优化空间

目的：优化JVM垃圾收集性能从而增大吞吐量或减少停顿时间，让应用在某个业务场景上发挥最大的价值。吞吐量是指应用程序线程用时占程序总用时的比例。暂停时间是应用程序线程让与GC线程执行而完全暂停的时间段

对于交互性web应用来说，一般都是减少停顿时间，所以有以下方法：

1. 如果应用存在大量的短期对象，应该选择较大的年轻代；如果存在相对较多的持久对象，老年代应该适当增大
2. 让大对象进入年老代。可以使用参数-XX:PetenureSizeThreshold 设置大对象直接进入年老代的阈值。当对象的大小超过这个值时，将直接在年老代分配
3. 设置对象进入年老代的年龄。如果对象每经过一次 GC 依然存活，则年龄再加 1。当对象年龄达到阈值时，就移入年老代，成为老年对象
4. 使用关注系统停顿的 CMS 回收器

基础：https://www.ibm.com/developerworks/cn/java/j-lo-jvm-optimize-experience/index.html

案例：https://www.wangtianyi.top/blog/2018/07/27/jvmdiao-you-ru-men-er-shi-zhan-diao-you-parallelshou-ji-qi/

欢迎光临[我的博客](http://www.wangtianyi.top/?utm_source=github&utm_medium=github)，发现更多技术资源~
