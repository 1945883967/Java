# 第三章 JDK并发包

## 3.1 多线程的团队协作：同步控制

同步控制是并发编程中必不可少的重要手段。synchronized是一种最简单的控制方法，它决定了一个线程是否可以访问临界资源。Object.wait()和Objec.notify()方法起到了线程等待和通知的作用。而**重入锁**——是synchronized、Object.wait()和Object.notify()方法的替代品(或者说是增强版)。

### 3.1.1 synchronized 的功能扩展：重入锁

重入锁可以完全代替synchronized关键字。在JDK5.0之前，重入锁的性能远远好于synchronized，但JDK1.6开始，JDK在synchronized上 做了大量的优化，使得两者的性能差距并不大。

重入锁使用**java.util.concurrent.locks.ReentrantLock**类来实现。

与synchronized相比，重入锁有着显示的操作过程。需手动指定何时加锁，何时是释放锁。因而，重入锁对逻辑控制的灵活性要要远远好于synchronized。（注：用重入锁加锁后，在退出临界区时，必须记得释放锁，否则其他线程就没有机会进入临界区）

之所以称为重入锁（从类名上看Re-Entrant-Lock翻译成重入锁非常贴切的），是因为这种锁是可以反复进入的，但这里的反复仅限于**同一个线程**。示例：

```java
lock.lock();
lock.lock();
try{
    i++;
}finally{
    lock.unlock();
    lock.unlock();
}
```

代码中一个线程连续两次获得同一把锁。（理解模糊p72）

除了使用上的灵活性外，重入锁还提供了一些高级功能：

* **中断响应**

  对于synchronized来说，如果一个线程在等待锁，那么结果只有两种，要么它获得这把锁继续执行，要么它保持等待。而使用重入锁，则提供另外一种可能，线程可以被中断。也就是在等待锁的过程中，程序可以根据需要取消对锁的请求。重入锁的**lockInterruptibly()**方法，可以对中断进行响应的锁申请动作，即在等待锁的过程中，可以响应中断（接受其他线程调用该线程的中断响应，该线程取消对锁的请求）。

* **锁申请等待限时**

  除了等待外部通知外，要避免死锁还有另外一种方式，限时等待。对于无法判断为什么一个线程迟迟拿不到锁，也许是因为死锁了，也许是因为产生了饥饿。可以给定一个等待时间，在给定时间内拿不到锁，让线程自动放弃。可以使用重入锁的**tryLock()**方法进行一次限时等待。

  ```java
  public boolean tryLock()
  public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException
  ```

  第一个方法：当前线程会尝试获得锁如果锁并未被其他线程占用，则申请锁成功，并立即返回true。如果锁被其他线程占用，则当前线程不会等待，而是立即返回false。

  第二个方法：接受两个参数，第一个表示等待时长，第二个表示计时单位，表示线程在这个锁请求中，在等待时间内还不能获取锁，则返回false。

* **公平锁**

  在大多数情况下，锁都是非公平的。

  公平锁的一大特点是不会产生饥饿现象，会按照时间的先后顺序，保证先到者先得，后到者后得。

  用synchronized关键字进行的锁控制是非公平的。

  重入锁允许我们对其公平性进行设置，可以通过构造函数

  ```java
  ReentrantLock(boolean fair) //创建一个具有给定公平策略的 ReentrantLock。
  ```

对重入锁的几个重要方法整理如下：

* **lock()**：获得锁，如果锁已经被占用在，则等待。
* **lockInterruptibly()**:获得锁，但优先响应中断。
* **tryLock()**:尝试获得锁，如果成功，返回true，失败返回false。该方法不等待，直接返回。
* **tryLock(long timeout, TimeUnit unit)**：给定时间内尝试获取锁。
* **unlock()**：释放锁。

从重入锁的实现来看，它主要集中在Java层面。在重入锁的实现中，主要包含三个要素：

第一，是原子状态。

第二，是等待队列。

第三，是阻塞原语park()和uppark()，用来挂起和恢复线程。

### 3.1.2 重入锁的好搭档：Condition条件

Condition对象和wait()和notify()方法的作用大致相同。 

* wait()和notify()方法是和synchronized关键字一起使用的。
* Condition对象是与重入锁相关联的。

通过使用ReentrantLock的实例调用方法

```java
public Condition newCondition() //返回用来与此 Lock 实例一起使用的 Condition 实例。 
```

生成一个与当前重入锁绑定的Condition实例，利用Condition对象，可以让线程在合适的时间等待，或者在某一特定的时刻得到通知，继续执行。

Condition接口提供的基本方法如下：
![Snipaste_2019-06-15_21-38-37.jpg](https://i.loli.net/2019/06/15/5d04f4ef3dd0071672.jpg)

### 3.1.3 允许多个线程同时访问：信号量（Semaphore）

信号量为多线程协作提供了更为强大的控制方法。广义上说，信号量是对锁的扩展。无论内部锁synchronized还是重入锁ReentrantLock，一次只允许一个线程访问一个资源，而信号量却可以指定多个线程，同时访问某一资源。信号量主要提供的构造函数：

```java
public Semaphore(int permits)//创建具有给定的许可数和非公平的公平设置的 Semaphore。
public Semaphore(int permits,boolean fair)//创建具有给定的许可数和给定的公平设置的Semaphore。 
```

信号量的主要逻辑方法：
![Snipaste_2019-06-16_12-19-08.jpg](https://i.loli.net/2019/06/16/5d05c38b128b155597.jpg)
![Snipaste_2019-06-16_12-19-29.jpg](https://i.loli.net/2019/06/16/5d05c38e7e8d547816.jpg)

### 3.1.4 ReadWriteLock读写锁

ReadWriteLock时JDK5提供的读写分离锁。读写锁可以有效地帮助减少锁竞争，以提升系统性能。读写锁允许多个线程同时读。虽然读与读之前不会发生数据安全问题，但写写操作和读写操作间依然是需要相互等到和持有锁的，因此读写锁有以下访问约束：

* 读-读不互斥：读读之间不阻塞。
* 读-写互斥：读阻塞写，写也会阻塞读。
* 写-写互斥：歇歇阻塞。

当系统中读操作次数远远大于写操作时，读写锁就可以发挥做大的功效，提升系统的性能。

### 3.1.5 倒计时器：CountDownLatch

倒计时器常用来控制线程等待，可以让某一个线程等待直到倒计时结束，再开始执行。

CountDownLatch的构造函数就收一个整数作为参数，即当前这个计数器的计数个数。

```java
public CountDownLatch(int count)
```

用大白话理解倒计时器：就是倒计数器上的所有线程（检查任务）都完成后,对应的目标线程才可以继续执行。

示意图：
![Snipaste_2019-06-19_09-21-30.jpg](https://i.loli.net/2019/06/19/5d098e7ca41af98851.jpg)

### 3.1.6 循环栅栏：CyclicBarrier

CyclicBarrier是另外一种多线程并发控制使用工具。和CountDownLatch非常相似，它也可以实现线程间的计数等待，但它的功能比CountDownLatck更加强大。

CyclicBarrier可以立即为循环栅栏。栅栏就是一种障碍物。Java并发中用来阻止线程继续执行，要求线程在栅栏处等待。前面的Cyclic意味循环，也就是说这个计数器可以重复使用。比如，假设我们将计数器设置为10，那么凑齐第一批10个线程后，计数器归零，然后接着凑齐下一批10个线程，这就是循环栅栏内在的含义。

比CountDownLatch强大的是，CyclicBarrier可以接受一个参数为barrierAction（当计数器一次技术完成后，系统会执行的动作），如下构造函数：(其中参数parties表示计数总数，也就是参与的线程数)

```java
public CyclicBarrier(int parties, Runnable barrierAction)
```

### 3.1.7线程的阻塞工具类：LockSupport

LockSupport是一个非常方便使用的线程阻塞工具，可以在线程内部任意位置阻塞。

LockSupport的静态方法park()可以阻塞当期线程，类似的还有parknanos()、parkUntil()等方法。它可以实现一个线程等待。

LockSupport使用信号量的机制，它为每一个线程准备了一个许可，如果许可可用，那么park()函数会立即返回，并且消费这个许可（也就是让许可变为不可用），如果许可不可用，就回阻塞。而unpack()则使得一个许可变为可用（但是和信号量不同的是，许可不能累加，你不能拥有超过一个许可，它永远只有一个）。

## 3.2 线程复用：线程池

线程的使用必须掌握一个度，在有限的范围内，增加线程的数量可以明显提高线程的吞吐量，但一旦超出这个范围，大量的线程只会拖垮应用系统。因此，在生产环境中使用线程，必须对其加以控制和管理。盲目的使用大量线程对系统性能是有伤害的。

### 3.2.1 什么是线程池

为了避免系统频繁的创建和销毁线程。可以让创建的线程进行复用。

线程池与数据库连接池类似。在线程池中，总有几个活跃的线程。当你需要使用线程时，可以从池子中随便拿出一个空闲线程，完成工作时，并不急着关闭线程，而是将这个线程退回到池子，方便其他人使用。简而言之：使用线程池后，创建线程变成了从线程池中获取线程，关闭线程变成了向池子中归还线程。

### 3.2.2 不要重复发明轮子：JDK对线程池的支持

为了能更好的控制多线程，JDK提供了一套Executor框架，帮助开发人员有效地进行线程控制，其本质就是一个线程池。它的核心成员如图：
![Snipaste_2019-06-19_10-08-49.jpg](https://i.loli.net/2019/06/19/5d099938cca7243437.jpg)

以上成员均在Java.util.concurrent包中，是JDK并发包的核心类。其中ThreadPoolExecutor
![Snipaste_2019-06-19_10-14-40.jpg](https://i.loli.net/2019/06/19/5d099a9d58e4a28562.jpg)

### 3.2.3 刨根问底：核心线程池的内部实现

对于核心的几个线程池，**newFixedThreadPool()**方法、 **newSingleThreadExecutor**()方法还是 

**newCachedThreadPool**()方法，其内部均使用**ThreadPoolExecutor**实现。具体实现如下：

![Snipaste_2019-06-19_11-12-20.jpg](https://i.loli.net/2019/06/19/5d09a821a5ade89571.jpg)

![Snipaste_2019-06-19_11-21-01.jpg](https://i.loli.net/2019/06/19/5d09aa71bf7d161158.jpg)
![Snipaste_2019-06-19_11-21-55.jpg](https://i.loli.net/2019/06/19/5d09aa72763cd22338.jpg)
![Snipaste_2019-06-19_11-22-08.jpg](https://i.loli.net/2019/06/19/5d09aa731bf6262325.jpg)
![Snipaste_2019-06-19_11-27-12.jpg](https://i.loli.net/2019/06/19/5d09ab9997e2175380.jpg)

### 3.2.4 超负载了怎么办：拒绝策略

ThreadPoolEcevutor的最后一个参数指定了拒绝策略。也就是任务数量超过系统实际承载能力时如何处理。拒绝策略可以说时系统超负荷运行的补救措施，通常由于压力太大而引起的，也就是线程池中的线程已经用完了，无法继续为新任务服务，同时，等待队列中也已经排满了，再也塞不下新任务了。这是，可以拒绝策略可以合理处理这个问题。

JDK内置提供了四种拒绝策略，如图：

![Snipaste_2019-06-19_11-35-31.jpg](https://i.loli.net/2019/06/19/5d09ad8d643c495299.jpg)

* **AbortPolicy策略**：该策略会直接抛出异常，阻止系统正常工作。
* **CallerRunsPolicy策略**：只要线程池未关闭，该策略直接在调用者线程中，运行当前被丢弃的任务。显然这样做不会真的丢弃任务，但是任务提交线程的性能极有可能会急剧下降。
* **DiscardOldestPolicy策略**：该策略会丢弃最老的一个请求，也就是即将被执行的一个任务，并尝试再次提交前。
* **DiscardOldestPolicy策略**：该策略默默的丢弃无法处理的任务，不予任何处理。

以上内置的策略均实现**RejectedExecutionHandler**接口，若以上策略任然无法满足实际应用需要，完全可以自己扩展RejectedExecutionHandler接口。RejectedExecutionHandler的定义如下：

```java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

其中r为执行任务，executor为当前的线程池。

### 3.2.5 自定义线程创建：ThreadFactory

线程池中的线程从哪来？ ===>>>**ThreadFacotry**

**ThreadFactory**是一个接口，它只有一个方法，用来创建线程。

```java
public interface ThreadFactory {
    Thread newThread(Runnable r);  
}
```

当线程池需要新建线程时，就会调用这个方法。

### 3.2.6 我的应用我做主：扩展线程池

**ThreadPoolExecutor**也是一个可以扩展的线程池。它提供了beforeExecutor()、afterExecutor()和terminated()三个接口对线程池进行控制。

在默认的**ThreadPoolExecutor**实现中，提供了空的beforeExecute()和afterExecute()实现。在实际应用中，可以对其进行扩展来实现对线程池运行状态的跟踪，输出一些有用的调试信息。

### 3.2.7 合理的选择：优化线程池线程数量

线程池的大小对系统的性能有一定的影响。过大和过小的线程数量都无法发挥最优的系统性能，但线程池的大小的确定也不需要太精确，因为只要避免极大和极小两种情况，线程池的大小对系统性能并不会影响太大。

一般来说线程池的大小要考虑CPU的数量、内存大小等因素。在《Java Concurrency in Practice》一书中给出了一个估算线程池大小的经验公式：
![Snipaste_2019-06-20_09-37-16.jpg](https://i.loli.net/2019/06/20/5d0ae3644663828778.jpg)

在Java中，可以通过：

```java
Runtime.getRuntime().availableProcessors()
```

取得可用COU数量。

### 3.2.8 堆栈去哪里了：在线程池中寻找堆栈

线程池的submit()和execute()方法的区别是execute()可以线程池讨回异常堆栈。

以下来两种方式等价：

* pools.execute(new DivTask(100,i));

* Future re = pools.submit(new DivTask(100,i));

  re.get();

### 3.2.9 分而治之：Fork/Join框架

“分而治之”一直是一个非常有效地处理大量数据的方法。

Fork一词的原始含义是吃饭用的叉子，也是分叉的意思。在Linux平台中，函数fork()用来创建子进程，使的系统可以多一个执行分支。在Java中也沿用了类似的命名方式。

join的含义表示等待。也就是使用frok()后系统多了一个执行分支（线程），所以需要等待这个执行分支完毕，才能得到最终的结果，因此join()就表示等待。

在jdk中for()方法并不急着开启线程，而是交给ForkJoinPool线程池处理，以节省系统资源。
![Snipaste_2019-06-20_10-07-18.jpg](https://i.loli.net/2019/06/20/5d0aea626b3c284781.jpg)

由于线程池的优化，提交的任务和线程数量并不是一对一的关系。在绝大多数情况下，一个物理线程实际上是需要处理多个逻辑任务的。因此，每个线程必然需要拥有一个任务队列。因此，在实际执行过程中，可能遇到这么一种情况：线程A已经把自己的任务都执行完成了，而线程B还有一堆任务等着处理，此时，线程A就会“帮助”线程B，从线程B的任务队列中拿一个任务过来处理，尽可能的达到平衡。如图：
![Snipaste_2019-06-20_10-11-12.jpg](https://i.loli.net/2019/06/20/5d0aebec2ae3716044.jpg)

注意：当线程帮助别人时，总是从任务队列的底部拿数据，而线程执行自己的任务时，则是从相反的顶部开始拿。因此这种行为十分有利于避免数据竞争。

**ForkJoinPool**的一个重要接口：

![Snipaste_2019-06-20_10-30-41.jpg](https://i.loli.net/2019/06/20/5d0aefe5a9dc477169.jpg)
![Snipaste_2019-06-20_10-30-51.jpg](https://i.loli.net/2019/06/20/5d0aefe61835554618.jpg)

## 3.3 不要重复发明轮子：JDK的并发容器

### 3.3.1 并发集简介

JDK提供的答这些大部分容器在java.util.concurrent包中。

* **ConcurrentHashMap**:这是一个高效的并发HashMap。可以理解为一个线程安全的HashMap。
* **CopyOnWriteArrayList**：这是一个List，和ArrayList是一个家族的，在读多写少的场合，这个List的性能非常好，远远好于Vector.
* **ConcurrentLinkedQueue**:高效的并发队列，使用链表实现。可以看做一个线程安全的LinkedList。
* **BlockingQueue**:这是一个接口，JDK内部通过链表、数组等方式实现这个接口。表示阻塞队列，非常适合于作为数据共享的通道。
* **ConcurrentSkipListMap**:跳表的实现。这是一个Map，使用跳表的数据结构进行快速查找。

除了以上并发包专有的数据结构外，java.util下的Vector是线程安全的（虽然性能和上述专用工具没得比），另外Conllection工具类可以帮助我们将任意集合包装成线程安全的集合。

### 3.3.2 线程安全的HashMap
![Snipaste_2019-06-20_10-55-36.jpg](https://i.loli.net/2019/06/20/5d0af5b440f5840195.jpg)

一个更加专业的并发HashMap是ConcurrentHashMap。它位于java.util.concurrent包内。它专门为并发进行了性能优化，因此更加适合多线程的场合。

### 3.3.3 有关List的线程安全

在Java中，ArrayList和Vector都是使用数组作为其内部实现的。两者最大的不同在于Vector是线程安全的，而ArrayList不是。此外，LinkedList使用链表的数据结构实现了List。但是很不幸，LinkedList并不是线程安全的，不过可以使用Collections.synchronizedList()方(返回指定列表支持的同步（线程安全的）列表)法包装任意List，如下：

```java
public static List<String> l = Collections.synchronizedList(new LinkedList<String>());
```

