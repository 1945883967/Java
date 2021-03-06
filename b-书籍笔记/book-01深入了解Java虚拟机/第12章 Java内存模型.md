# 第12章 Java内存模型

## Java内存模型

Java内存模型来屏蔽掉各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的内存访 问效果。

### 1 主内存与工作内存

Java内存模型的主要目标识定义程序中各个变量的访问规则，则在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。

此处的变量（Variables）与Java编程中所说的 变量有所区别，它包括了实例字段、静态字段和构成数组对象的元素，但不包括局部变量与 方法参数，因为后者是线程私有的，不会被共享，自然就不会存在竞争问题。

注：如果局部变量是一个reference类型，它引用的对象在Java堆中可被各个线程共享，但是reference本身在Java栈的局部变量表中，它是线程私有的。

Java内存模型规定了所有的变量都存储在**主内存**（Main Memory）中（此处的主内存与 介绍物理硬件时的主内存名字一样，两者也可以互相类比，但此处仅是虚拟机内存的一部 分）。每条线程还有自己的**工作内存**（Working Memory，可与前面讲的处理器高速缓存类比），线程的工作内存中保存了被该线程使用到的变量的主内存副本拷贝（对象中的引用，对象中某个线程访问到的字段是有可能存在拷贝的，但不会有整个虚拟机实现把整个对象拷贝一次)，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的变量。 不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成，线程、主内存、工作内存三者的交互关系如图
![Snipaste_2019-08-14_16-57-27.jpg](https://i.loli.net/2019/08/14/nSRYPzeJpUfiA4Z.jpg)
### 2 内存间交互操作
关于主内存与工作内存之间具体的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步回主内存之类的实现细节，java 内存模型定义了以下8中操作来完整（虚拟机实现时必须保证下面提及的每一种操作都是原子的、不可再分的）：
* **Lock(锁定)：**作用于主内存的变量，它把一个变量标识为一条线程独占的状态。 
* **unlock(解锁)：**作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。 
* **read(读取)**：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内 存中，以便随后的load动作使用。 
* **load(载入)**：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工 作内存的变量副本中。 
* **use(使用)**：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引 擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作。 
* **assign(赋值)**：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内 存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。 
* **store(存储)**：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存 中，以便随后的write操作使用。 
* **write(写入)：**作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入 主内存的变量中。 

如果把一个变量从主内存复制到工作内存，就需要顺序地执行read和load操作，如果要把变量从工作内存同步回主内存，就要顺序地执行store和write操作。注意：这里只是说明顺序执行，但没有保证是连续的。

除此之外，Java内存模型还规定了在执行上述8种基本操作时必须满足如下规则：

* 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取了 但工作内存不接受，或者从工作内存发起回写了但主内存不接受的情况出现。 
* 不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该 变化同步回主内存。 
* 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步 回主内存中。
* 一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化 （load或assign）的变量，换句话说，就是对一个变量实施use、store操作之前，必须先执行 过了assign和load操作。 
* 一个变量在同一个时刻只允许一条线程对其进行lock操作，但lock操作可以被同一条线 程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。 
* 如果对一个变量执行lock操作，那将会清空工作内存中此变量的值，在执行引擎使用这 个变量前，需要重新执行load或assign操作初始化变量的值。 
* 如果一个变量事先没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许去 unlock一个被其他线程锁定住的变量。
*  对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操 作）。 

### 3.对于volatile型变量的特殊规则

关键字volatile可以说是Java虚拟机提供的最轻量级的同步机制。

当一个变量定义为volatile之后，它将具备两种特性：

* 保证此变量对所有线程的**可见性**，这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量不能做到这一点，普通变量的值在线程间传递均需要通过主内存类完成。

  volatile变量在各个线程的工作内存内也存在不一致的情况（但由于每次使用前都要刷新，执行引擎看不到不一致的情况，因此可以认为不存在一致性的问题）。由于**volatile变量只能保证可见性**，多个线程同时修改时，依然会发生线程安全问题，因为当两个线程同时修改时，一个先结束，则另外一个结束时发现操作栈顶的值就变成了过期的数据，所以会丢弃此次的计算。

  在不符合以下两条规则的运算场景中，任然要通过加锁（使用synchronized和java.util.concurrent中的原子类）来保证原子性

  * **运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。**
  * **变量不需要与其他的状态变量共同参与不变约束。**

* **禁止指令重排序优化**，普通的变量仅仅会保证在该方法 的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的 顺序与程序代码中的执行顺序一致。

Java 内存模型中对volatile变量定义的特殊规则：

* 每次使用volatile变量前都必须从主存刷新最新的值，用于保证能看见其他线程对该变量所做的修改后的值。
* 每次修改volatile变量后都必须立刻同步回主存中，用于保证其他线程可以看到自己对变量V所做的修改。
* volatile修饰的变量不会被指令排序优化，保证代码的执行顺序与程序的顺序相同。

#### 对于long和double型变量的特殊规则

允许虚拟机将没有被volatile修饰的64位数据的读写操作划分为两次32位的操作来进行，即允许虚拟机实现选择可以不保证64位数据类型的load、store、read和write这4个操作的原子性，这点就是所谓的long和double的**非原子性协定**。

Java内存模型虽然允许虚拟机不把long和double变量的读写实现成原子操作，但允许虚拟机选择把这些操作实现为具有原子性的操作，目前各种平台下的商用虚拟机几乎都选择把64位数据的读写操作作为原子操作来对待， 因此我们在编写代码时一般不需要把用到的long和double变量专门声明为volatile。

### 原子性、可见性与有序性

Java内存模型是围绕着在开发过程中如何处理原子性、可见性和有序性这三个特征来建立的。

**原子性（Atomicity）**：由Java内存模型来直接保证的原子性变量操作包括read、load、 assign、use、store和write，我们大致可以认为基本数据类型的访问读写是具备原子性的（例 外就是long和double的非原子性协定，读者只要知道这件事情就可以了，无须太过在意这些 几乎不会发生的例外情况）。 

**synchronized块 之间的操作也具备原子性**

**可见性（Visibility）**：可见性是指当一个线程修改了共享变量的值，其他线程能够立即 得知这个修改。Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是volatile变量都是如此。

普通变量与 volatile变量的区别是：volatile的特殊规则保证了新值能**立即同步到主内存**，以及每次使用前立即从主内存刷新。因此，可以说volatile保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。 

除了volatile之外，Java还有两个关键字能实现可见性，即synchronized和final:

**synchronized**:同步块的 可见性是由“对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、 write操作）”这条规则获得的。

synchronized对于同步方法，JVM采用`ACC_SYNCHRONIZED`标记符来实现同步。 对于同步代码块。JVM采用`monitorenter`、`monitorexit`两个指令来实现同步。

**final**:被final修饰的字段在构造器中一 旦初始化完成，并且构造器没有把“this”的引用传递出去（this引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到“初始化了一半”的对象），那在其他线程中就能看见final字段的值。

**有序性（Ordering）**：Java程序中天然的有序性可以总结为一句话：**如果在本线程内观察，所有的操作都是有 序的；如果在一个线程中观察另一个线程，所有的操作都是无序的**。前半句是指“线程内表现为串行的语义”（Within-Thread As-If-Serial Semantics），后半句是指“指令重排序”现象 和“工作内存与主内存同步延迟”现象。

java语言提供了volatile和synchronize两个关键字来保证线程之间操作的有序性，volatile关键字本身就包含了禁止指令重排序的语义，而synchronized则是保证持有同一个锁的两个同步块只能串行的进入。

### 先行发生原则

Java语言中有一个“先行发生”（happens-before）的原则，它是判断数据是否存在竞争、线程是否安全的主要依据，依靠这个原则，我们可以通过几条规则一揽子地解决并发环境下两个操作之间是否可能存在冲突的所有问题。 

下面是Java内存模型下一些“天然的”先行发生关系，这些先行发生关系无须任何同步器 协助就已经存在，可以在编码中直接使用。如果两个操作之间的关系不在此列，并且无法从 下列规则推导出来的话，它们就没有顺序性保障，虚拟机可以对它们随意地进行重排序。 

* 程序次序规则（Program Order Rule）：在一个线程内，按照程序代码顺序，书写在前面 的操作先行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序， 因为要考虑分支、循环等结构。 
* 管程锁定规则（Monitor Lock Rule）：一个unlock操作先行发生于后面对同一个锁的lock 操作。这里必须强调的是同一个锁，而“后面”是指时间上的先后顺序。 
* volatile变量规则（Volatile Variable Rule）：对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后顺序。 
* 线程启动规则（Thread Start Rule）：Thread对象的start（）方法先行发生于此线程的每一个动作。
*  线程终止规则（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的
    终止检测，我们可以通过Thread.join（）方法结束、Thread.isAlive（）的返回值等手段检测 到线程已经终止执行。
*  线程中断规则（Thread Interruption Rule）：对线程interrupt（）方法的调用先行发生于被 中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted（）方法检测到是否有 中断发生。 对象终结规则（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发 生于它的finalize（）方法的开始。 
* 传递性（Transitivity）：如果操作A先行发生于操作B，操作B先行发生于操作C，那就 可以得出操作A先行发生于操作C的结论。

Java语言无需任何同步手段保障就能成立的先行发生规则只有上面这些。

时间先后顺序与先行发生原则之间基本没有太大的关系，所以我们衡量并发安全问题的时候不要受到时间顺序的干扰，**一切必须以先行发生原则为准**。

## Java与线程

### 1.线程的实现

线程是比进程更轻量级的调度执行单位，线程的引入，可以把一个进程的资 源分配和执行调度分开，各个线程既可以共享进程资源（内存地址、文件I/O等），又可以 独立调度（线程是CPU调度的基本单位）。 

实现线程有三种方式：使用内核实现、使用用户线程实现和使用用户线程加轻量级进程混合实现。

**1.使用内核实现**

内核线程（Kernel-Level Thread,KLT）就是直接由操作系统内核（Kernel，下称内核）支 持的线程，这种线程由内核来完成线程切换，内核通过操纵调度器（Scheduler）对线程进行 调度，并负责将线程的任务映射到各个处理器上。每个内核线程可以视为内核的一个分身， 这样操作系统就有能力同时处理多件事情，支持多线程的内核就叫做多线程内核（MultiThreads Kernel）。 

**2.使用用户线程实现**

**3.使用用户线程加轻量级进程混合实现**、

线程除了依赖内核线程实现和完全由用户程序自己实现之外，还有一种将内核线程与用 户线程一起使用的实现方式。在这种混合实现下，既存在用户线程，也存在轻量级进程。用 户线程还是完全建立在用户空间中，因此用户线程的创建、切换、析构等操作依然廉价，并 且可以支持大规模的用户线程并发。而操作系统提供支持的轻量级进程则作为用户线程和内 核线程之间的桥梁，这样可以使用内核提供的线程调度功能及处理器映射，并且用户线程的 系统调用要通过轻量级线程来完成，大大降低了整个进程被完全阻塞的风险。在这种混合模 式中，用户线程与轻量级进程的数量比是不定的，即为N：M的关系。

**4.Java线程的实现**

Java线程在JDK 1.2之前，是基于称为“绿色线程”（Green Threads）的用户线程实现的， 而在JDK 1.2中，线程模型替换为基于操作系统原生线程模型来实现。

## 2.Java线程调度

线程调度是指系统为线程分配处理器使用权的过程，主要调度方式有两种，分别是**协同式**线程调度和抢占式线程调度。 

### 3.状态转换

在java语言中有5中线程状态，在任何一个时间点，一个线程只能有且只有其中的一种状态，这5种状态分别如下：

* 新建（New）：创建后尚未启动的线程处于这种状态。 
* 运行（Runable）：Runable包括了操作系统线程状态中的Running和Ready，也就是处于此 状态的线程有可能正在执行，也有可能正在等待着CPU为它分配执行时间。 
* 无限期等待（Waiting）：处于这种状态的线程不会被分配CPU执行时间，它们要等待被 其他线程显式地唤醒。以下方法会让线程陷入无限期的等待状态： 
  * 没有设置Timeout参数的Object.wait（）方法。 
  * 没有设置Timeout参数的Thread.join（）方法
  * LockSupport.park（）方法。 
* 限期等待（Timed Waiting）：处于这种状态的线程也不会被分配CPU执行时间，不过无 须等待被其他线程显式地唤醒，在一定时间之后它们会由系统自动唤醒。以下方法会让线程 进入限期等待状态： 
  * Thread.sleep（）方法。 
  * 设置了Timeout参数的Object.wait（）方法。 
  * 设置了Timeout参数的Thread.join（）方法。 
  * LockSupport.parkNanos（）方法。 
  * LockSupport.parkUntil（）方法
* 阻塞（Blocked）：线程被阻塞了，“阻塞状态”与“等待状态”的区别是：“阻塞状态”在等待着获取到一个排他锁，这个事件将在另外一个线程放弃这个锁的时候发生；而“等待状态”则是在等待一段时间，或者唤醒动作的发生。在程序等待进入同步区域的时候，线程将进入这种状态。 
* 结束（Terminated）：已终止线程的线程状态，线程已经结束执行.

上述5中状态在遇到特定事件发生的时候会互相转换，它们的转换关系如图：

![Snipaste_2019-08-14_21-20-50.jpg](https://i.loli.net/2019/08/14/lDWKaThmiFjbx7g.jpg)