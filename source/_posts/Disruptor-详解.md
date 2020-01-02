---
title: Disruptor——详解
date: 2020-01-02 17:48:12
tags:
- Disruptor
- 高效队列
- 伪共享
- CAS
- 环形队列
- DSL
- MESI 协议
categories:
- 后端
- Java
---

# 资料一（基础概念+简单demo）

**背景**

高并发是指通过设计保证系统能够同时并行处理很多请求。虽然我在工作中经常听到高并发，QPS之类的术语。其实，我对高并发也是一知半解，知道Java里面可以用Lock，Synchronized，ArrayBlockingQueue之类的来进行高并发的处理。我个人觉得，高并发领域更多的依靠的是经验的累积。今天想跟大家分享的是一个高性能的并发框架Disruptor。

**Disruptor概述**

Disruptor是一个异步并发处理框架。是由LMAX公司开发的一款高效的无锁内存队列。它使用无锁的方式实现了一个环形队列，非常适合于实现生产者和消费者模式，比如事件和消息的发布。

Disruptor最大特点是高性能，其LMAX架构可以获得每秒6百万订单，用1微秒的延迟获得吞吐量为100K+。

**一个官网的简单的demo**

1.maven依赖
maven引入Disruptor的jar包，Disruptor版本为3.2.1

```xml
<dependency> 
  <groupId>com.lmax</groupId>  
  <artifactId>disruptor</artifactId>  
  <version>3.2.1</version> 
</dependency>
```

2.创建数据实体类LongEvent

```java
//代表数据的类
public class LongEvent {
    private long value;

    public void set(long value) {
        this.value = value;
    }
}
```

3.创建工厂类LongEventFactory

```java
//产生LongEvent的工厂类，它会在Disruptor系统初始化时，构造所有的缓冲区中的对象实例（预先分配空间）
public class LongEventFactory implements EventFactory<LongEvent> {
    public LongEvent newInstance() {
        return new LongEvent();
    }
}
```

4.创建消费者类LongEventHandler

```java
//消费者实现为WorkHandler接口，是Disruptor框架中的类
public class LongEventHandler implements EventHandler<LongEvent> {
    //onEvent()方法是框架的回调用法
    public void onEvent(LongEvent event, long sequence, boolean endOfBatch) {
        System.out.println("Event: " + event);
    }
}
```

5.创建生产者类LongEventProducer

```java
//消费者实现为WorkHandler接口，是Disruptor框架中的类
public class LongEventProducer {
    //环形缓冲区,装载生产好的数据；
    private final RingBuffer<LongEvent> ringBuffer;

    public LongEventProducer(RingBuffer<LongEvent> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    //将数据推入到缓冲区的方法：将数据装载到ringBuffer
    public void onData(ByteBuffer bb) {
        long sequence = ringBuffer.next();

        // Grab the next sequence //获取下一个可用的序列号
        try {
            LongEvent event = ringBuffer.get(sequence);

            // Get the entry in the Disruptor //通过序列号获取空闲可用的LongEvent

            // for the sequence
            event.set(bb.getLong(0));

            // Fill with data //设置数值
        }
        finally {
            ringBuffer.publish(sequence);

            //数据发布，只有发布后的数据才会真正被消费者看见
        }
    }
}
```

6.创建测试类 LongEventMain

```java
public class LongEventMain {

    public static void main(String[] args) throws Exception {
        // 创建线程池
        Executor executor = Executors.newCachedThreadPool();
        // 事件工厂
        LongEventFactory factory = new LongEventFactory();
        // ringBuffer 的缓冲区的大小是1024
        int bufferSize = 1024;
        // 创建一个disruptor, ProducerType.MULTI:创建一个环形缓冲区支持多事件发布到一个环形缓冲区
        Disruptor<LongEvent> disruptor = new Disruptor<>(factory, bufferSize, executor, ProducerType.MULTI, new BlockingWaitStrategy());
        // 创建一个消费者
        disruptor.handleEventsWith(new LongEventHandler());
        // 启动并初始化disruptor
        disruptor.start();
        // 获取已经初始化好的ringBuffer
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        // 生产数据
        LongEventProducer producer = new LongEventProducer(ringBuffer);
        ByteBuffer bb = ByteBuffer.allocate(8);
        for (long l = 0; true; l++) {
            bb.putLong(0, l);
            producer.onData(bb);
            Thread.sleep(1000);
        }

    }

}
```

7.demo结果输出

{% asset_img image-20191218180407076.png %}

**Disruptor的一些核心介绍**

**1.RingBuffer**

RingBuffer是其核心，生产者向RingBuffer中写入元素，消费者从RingBuffer中消费元素。

随着你不停地填充这个buffer（可能也会有相应的读取），这个序号会一直增长，直到绕过这个环。

槽的个数是2的N次方更有利于基于二进制的计算机进行计算。（注：2的N次方换成二进制就是1000，100，10，1这样的数字， sequence & （array length－1） = array index，比如一共有8槽，3&（8－1）=3，HashMap就是用这个方式来定位数组元素的，这种方式比取模的速度更快。）

会预先分配内存,可以做到完全的内存复用。在系统的运行过程中，不会有新的空间需要分配或者老的空间需要回收。因此，可以大大减少系统分配空间以及回收空间的额外开销。

关于RingBuffer可以直观的看一下下面的这幅图片（网上copy的），表示取到编号为4的数据。

{% asset_img image-20191218180458585.png %}

**2.消费者等待策略**

BlockingWaitStrategy：这是默认的策略。使用锁和条件进行数据的监控和线程的唤醒。因为涉及到线程的切换，是最节省CPU，但在高并发下性能表现最糟糕的一种等待策略。

SleepingWaitStrategy:会自旋等待数据，如果不成功，才让出cpu，最终进行线程休眠，以确保不占用太多的CPU数据，因此可能产生比较高的平均延时。比较适合对延时要求不高的场合，好处是对生产者线程的影响最小。典型的应用场景是异步日志。

YieldingWaitStrategy:用于低延时的场合。消费者线程不断循环监控缓冲区变化，在循环内部，会使用Thread.yield()让出cpu给别的线程执行时间。

BusySpinWaitStrategy:开启的是一个死循环监控，消费者线程会尽最大努力监控缓冲区变化，因此，CPU负担比较大

**3.Disruptor的应用场景**

Disruptor号称能够在一个线程里每秒处理 6 百万订单,实际上我也没有测试过。Disruptor实际上内部是使用环形队列来实现的，所以一般来说，在消费者和生产者的场景中都可以考虑使用Disruptor。比如像日志处理之类的。实际上，我个人觉得Disruptor就像是Java里面的ArrayBlockingQueue的替代者，因为Disruptor可以提供更高的并发度和吞吐量。从下面这幅官网的图就可以直观的感受到Disruptor和ArrayBlockingQueue之间的效率的对比。(注意这是一个对数对数尺度，不是线性的。)

{% asset_img 640.webp %}

# 资料二（Disruptor为什么这么快？）

**Padding Cache Line，体验高速缓存的威力**

我们先来看看 Disruptor 里面一段神奇的代码。这段代码里，Disruptor 在 RingBufferPad 这个类里面定义了 p1，p2 一直到 p7 这样 7 个 long 类型的变量。

```java
public abstract class RingBufferPad {
    protected long p1, p2, p3, p4, p5, p6, p7;
}
```

{% asset_img 640-1576721121761.webp %}

面临这样一个情况，Disruptor 里发明了一个神奇的代码技巧，这个技巧就是缓存行填充。Disruptor 在 RingBufferFields 里面定义的变量的前后，分别定义了 7 个 long 类型的变量。前面的 7 个来自继承的 RingBufferPad 类，后面的 7 个则是直接定义在 RingBuffer 类里面。这 14 个变量没有任何实际的用途。我们既不会去读他们，也不会去写他们。

而 RingBufferFields 里面定义的这些变量都是 final 的，第一次写入之后不会再进行修改。所以，一旦它被加载到 CPU Cache 之后，只要被频繁地读取访问，就不会再被换出 Cache 了。这也就意味着，对于这个值的读取速度，会是一直是 CPU Cache 的访问速度，而不是内存的访问速度。

**使用 RingBuffer，利用缓存和分支预测**

其实这个利用 CPU Cache 的性能的思路，贯穿了整个 Disruptor。Disruptor 整个框架，其实就是一个高速的生产者 - 消费者模型（Producer-Consumer）下的队列。生产者不停地往队列里面生产新的需要处理的任务，而消费者不停地从队列里面处理掉这些任务。

{% asset_img 640-1576721233410.webp %}

如果你熟悉算法和数据结构，那你应该非常清楚，如果要实现一个队列，最合适的数据结构应该是链表。我们只要维护好链表的头和尾，就能很容易实现一个队列。生产者只要不断地往链表的尾部不断插入新的节点，而消费者只需要不断从头部取出最老的节点进行处理就好了。我们可以很容易实现生产者 - 消费者模型。实际上，Java 自己的基础库里面就有 LinkedBlockingQueue 这样的队列库，可以直接用在生产者 - 消费者模式上。

{% asset_img 640-1576721247878.webp %}

不过，Disruptor 里面并没有用 LinkedBlockingQueue，而是使用了一个 RingBuffer 这样的数据结构，这个 RingBuffer 的底层实现则是一个固定长度的数组。比起链表形式的实现，数组的数据在内存里面会存在空间局部性。

就像上面我们看到的，数组的连续多个元素会一并加载到 CPU Cache 里面来，所以访问遍历的速度会更快。而链表里面各个节点的数据，多半不会出现在相邻的内存空间，自然也就享受不到整个 Cache Line 加载后数据连续从高速缓存里面被访问到的优势。

除此之外，数据的遍历访问还有一个很大的优势，就是 CPU 层面的分支预测会很准确。这可以使得我们更有效地利用了 CPU 里面的多级流水线，我们的程序就会跑得更快。

**总结延伸**

好了，不知道讲完这些，你有没有体会到 Disruptor 这个框架的神奇之处呢？

CPU 从内存加载数据到 CPU Cache 里面的时候，不是一个变量一个变量加载的，而是加载固定长度的 Cache Line。如果是加载数组里面的数据，那么 CPU 就会加载到数组里面连续的多个数据。所以，数组的遍历很容易享受到 CPU Cache 那风驰电掣的速度带来的红利。

对于类里面定义的单独的变量，就不容易享受到 CPU Cache 红利了。因为这些字段虽然在内存层面会分配到一起，但是实际应用的时候往往没有什么关联。于是，就会出现多个 CPU Core 访问的情况下，数据频繁在 CPU Cache 和内存里面来来回回的情况。而 Disruptor 很取巧地在需要频繁高速访问的变量，也就是 RingBufferFields 里面的 indexMask 这些字段前后，各定义了 7 个没有任何作用和读写请求的 long 类型的变量。

这样，无论在内存的什么位置上，这些变量所在的 Cache Line 都不会有任何写更新的请求。我们就可以始终在 Cache Line 里面读到它的值，而不需要从内存里面去读取数据，也就大大加速了 Disruptor 的性能。

这样的思路，其实渗透在 Disruptor 这个开源框架的方方面面。作为一个生产者 - 消费者模型，Disruptor 并没有选择使用链表来实现一个队列，而是使用了 RingBuffer。RingBuffer 底层的数据结构则是一个固定长度的数组。这个数组不仅让我们更容易用好 CPU Cache，对 CPU 执行过程中的分支预测也非常有利。更准确的分支预测，可以使得我们更好地利用好 CPU 的流水线，让代码跑得更快。

# 资料三（Disruptor无锁框架为啥这么快）

**1.1 CPU缓存**

在现代计算机当中，CPU是大脑，最终都是由它来执行所有的运算。而内存(RAM)则是血液，存放着运行的数据；但是，由于CPU和内存之间的工作频率不同，CPU如果直接去访问内存的话，系统性能将会受到很大的影响，所以在CPU和内存之间加入了三级缓存，分别是L1、L2、L3。

当CPU执行运算时，它首先会去L1缓存中查找数据，找到则返回；如果L1中不存在，则去L2中查找，找到即返回；如果L2中不存在，则去L3中查找，查到即返回。如果三级缓存中都不存在，最终会去内存中查找。对于CPU来说，走得越远，就越消耗时间，拖累性能。

{% asset_img 640-1576721576044.webp %}

在三级缓存中，越靠近CPU的缓存，速度越快，容量也越小，所以L1缓存是最快的，当然制作的成本也是最高的，其次是L2、L3。

CPU频率，就是CPU运算时的工作的频率（1秒内发生的同步脉冲数）的简称，单位是Hz。主频由过去MHZ发展到了当前的GHZ（1GHZ=10^3MHZ=10^6KHZ= 10^9HZ）。

内存频率和CPU频率一样，习惯上被用来表示内存的速度，内存频率是以MHz（兆赫）为单位来计量的。目前较为主流的内存频率1066MHz、1333MHz、1600MHz的DDR3内存，2133MHz、2400MHz、2666MHz、2800MHz、3000MHz、3200MHz的DDR4内存。

{% asset_img 640-1576721594437.webp %}

可以看得出，如果CPU直接访问内存，是一件相当耗时的操作。

**1.2 缓存行**

当数据被加载到三级缓存中，它是以缓存行的形式存在的，不是一个单独的项，也不是单独的指针。

在CPU缓存中，数据是以缓存行(cache line)为单位进行存储的，每个缓存行的大小一般为32—256个字节，常用CPU中缓存行的大小是64字节；CPU每次从内存中读取数据的时候，会将相邻的数据也一并读取到缓存中，填充整个缓存行；

{% asset_img 640-1576721623615.webp %}

可想而知，当我们遍历数组的时候，CPU遍历第一个元素时，与之相邻的元素也会被加载到了缓存中，对于后续的遍历来说，CPU在缓存中找到了对应的数据，不需要再去内存中查找，效率得到了巨大的提升；

但是，在多线程环境中，也会出现伪共享的情况，造成程序性能的降低，堪称无形的性能杀手。

**1.2.1 缓存命中**

通过具体的例子，来阐述缓存命中和未命中之间的效率：

测试代码：

```java
public class CacheHit {

    //二维数组：
    private static long[][] longs;

    //一维数组长度：
    private static int length = 1024*1024;

    public static void main(String [] args) throws InterruptedException {
        //创建二维数组,并赋值：
        longs = new long[length][];
        for(int x = 0 ;x < length;x++){
            longs[x] = new long[6];
            for(int y = 0 ;y<6;y++){
                longs[x][y] = 1L;
            }
        }
        cacheHit();
         cacheMiss();
    }
    //缓存命中：
    private static void cacheHit() {
        long sum = 0L;
        long start = System.nanoTime();
        for(int x=0; x < length; x++){
            for(int y=0;y<6;y++){
                sum += longs[x][y];
            }
        }
        System.out.println("命中耗时："+(System.nanoTime() - start));
    }
    //缓存未命中：
    private static void cacheMiss() {
        long sum = 0L;
        long start = System.nanoTime();
        for(int x=0;x < 6;x++){
            for(int y=0;y < length;y++){
                sum += longs[y][x];
            }
        }
        System.out.println("未命中耗时："+(System.nanoTime() - start));
    }
}
```

测试结果：

```java
未命中耗时：43684518
命中耗时：19244507
```

在Java中，一个long类型是8字节，而一个缓存行是64字节，因此一个缓存行可以存放8个long类型。但是，在内存中的布局中，对象不仅包含了实例数据(long类型变量)，还包含了对象头。对象头在32位系统上占用8字节，而64位系统上占用16字节。

所以，在上面的例子中，笔者向二维数组中填充了6个元素，占用了48字节。

在cacheHit()的例子中，当第一次遍历的时候，获取longs\[0\]\[0\]，而longs\[0\]\[0\]—longs\[0\]\[5\]也同时被加载到了缓存行中，接下来获取longs\[0\]\[1\]，已存在缓存行中，直接从缓存中获取数据，不用再去内存中查找，以此类推；

在cacheMiss()的例子中，当第一次遍历的时候，也是获取longs\[0\]\[0\]的数据，longs\[0\]\[0\]—longs\[0\]\[5\]也被加载到了缓存行中，接下来获取long\[1\]\[0\]，不存在缓存行中，去内存中查找，以此类推；

以上的例子可以充分说明缓存在命中和未命中的情况下，性能之间的差距。

**1.2.2 伪共享**

由于CPU加载机制，某个数据被加载的同时，其相邻的数据也会被加载到CPU当中。在得到CPU免费加载的同时，也产生了不好的情况；俗话说得好，凡事都有利有弊。

在我们的java程序中，当多个线程修改两个独立变量的时候，如果这两个变量存在于一个缓存行中，那么就有很大的概率产生伪共享。

这是为什么呢？

现如今，CPU都是多核处理器，一般为2核或者4核，当我们程序运行时，启动了多个线程。例如：核心1启动了1个线程，核心2启动了1个线程，这2个线程分别要修改不同的变量，其中核心1的线程要修改x变量，而核心2的线程要修改y变量，但是x、y变量在内存中是相邻的数据，他们被加载到了同一个缓存行当中，核心1的缓存行有x、y，核心2的缓存行也有x、y。

那么，只要有一个核心中的线程修改了变量，另一个核心的缓存行就会失效，导致数据需要被重新到内存中读取，无意中影响了系统的性能，这就是伪共享。

{% asset_img 640-1576721831140.webp %}

cpu的伪共享问题本质是：几个在内存中相邻的数据，被CPU的不同核心加载在同一个缓存行当中，数据被修改后，由于数据存在同一个缓存行当中，进而导致缓存行失效，引起缓存命中降低。

{% asset_img 640-1576721844773.webp %}

代码例子：

```java
public class FalseShare implements Runnable{

    //线程数、数组大小：
    public static int NUM_THREADS = 4; // change

    //数组迭代的次数：
    public final static long ITERATIONS = 500L * 1000L * 1000L;

    //线程需要处理的数组元素角标：
    private final int handleArrayIndex;

    //操作数组：
    private static VolatileLong[] longs = new VolatileLong[NUM_THREADS];

    //对数组的元素进行赋值：
    static{
        for (int i = 0; i < longs.length; i++) {
            longs[i] = new VolatileLong();
        }
    }

    public FalseShare(final int handleArrayIndex) {
        this.handleArrayIndex = handleArrayIndex;
    }

    //启动线程，每一个线程操作一个数组的元素，一一对应：
    public static void main(final String[] args) throws Exception {
        //程序睡眠必须加上：
        Thread.sleep(10000);

        final long start = System.nanoTime();

        Thread[] threads = new Thread[NUM_THREADS];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(new FalseShare(i));
        }
        for (Thread t : threads) {
            t.start();
        }
        for (Thread t : threads) {
            t.join();
        }
        System.out.println(System.nanoTime() - start);
    }

    //对数组的元素进行操作：
    public void run() {
        long i = ITERATIONS;
        while (0 != --i) {
            longs[handleArrayIndex].value = i;
        }
    }

    //数组元素：
    public final static class VolatileLong {
        public volatile long value = 0L;
        public long p1, p2, p3, p4, p5; //代码1
        public int p6;//代码1
    }
}
```

测试结果：（纳秒）

```java
未注释代码1：19830512      18472356     19993249    19841462
注释代码1：  21141471      25611265     19939633    29976847
```

通过测试结果，可以看出，在注释掉代码后，性能明显下降。让我们来阐述下原因：

通过代码，我们可以看出来，程序模拟的情况就是每一个线程操作数组中的一个元素，例如：线程1操作longs[0]，线程2操作longs[1]，线程3操作longs[2]…以此类推；之前说过，CPU缓存中是以缓存行为单位来进行存储的，一个缓存行大小为64字节。在程序中VolatileLong对象，正好满足64字节，为什么这么说？

在Java程序中，对象在内存中的分布：对象头（Header），实例数据（Instance Data），对齐填充（Padding）；

其中，对象头在32位系统上占用8字节，64位系统上占用16字节；实例数据也就是我们平常是所用到的基本类型和引用类型；对齐填充是对象在内存区域内的补充，jvm要求对象在内存区域的大小必须是8的整数倍，所以当对象头+实例数据的和不是8的整数倍时，就需要用到对齐填充，少多少就填充多少无效数据；

综上所述，VolatileLong=对象头(12字节)+value(8字节)+p1-p5(40字节)+p6(4字节) = 64字节，正好填充满整个缓存行；

当我们没有注释掉代码的时候，数组的各个元素将分布在不同的缓存行当中；而当注释掉代码的时候，数组的元素有很大的几率分布在同一个缓存行当中；当不同线程操作元素的时候，就会产生冲突，产生伪共享，影响系统性能；

经过上面的叙述，你大概对伪共享有了一定的了解，但是你会不会有这样的疑问？为什么其中1个核心缓存行的数据被修改了，其余核心中的缓存行就失效了？是什么机制产生了这样的情况？

以下，我们就来简单的介绍CPU的一致性协议MESI，就是这个协议保证了Cache的一致性；

**1.2.3 MESI协议**

多核理器中，每个核心都有自己的cache，内存中的数据可以同时处于不同的cache中，若各个核心独立修改自己的cache，就会出现不一致问题。为了解决一致性问题，MESI协议被引入。

MESI（Modified Exclusive Shared Or Invalid）是一种广泛使用的支持写回策略的缓存一致性协议，该协议最早被应用在Intel奔腾系列的CPU中。

其实，MESI协议就是规定了缓存行的4种状态，以及这4种状态之间的流转，以来保证不同核心中缓存的一致；每种状态在缓存行中用2个bit位来进行描述，分别是修改态（M）、独享态（E）、共享态（S）、无效态（I）；

{% asset_img 640-1576721907495.webp %}

- E(Exclusive)：x变量只存在于core1中；

{% asset_img 640-1576721931430.webp %}

- S(Shared):x变量存在于core1 core2 core3中

{% asset_img 640-1576721944728.webp %}

- M(Modified)：core1修改了x变量，core2 core3的缓存行被置为无效状态

{% asset_img 640-1576721956301.webp %}

在CPU中，每个核心不但控制着自己缓存行的读写操作，而且还监听这其他核心中缓存行的读写操作；每个缓存行的状态受到本核心和其他核心的双重影响；

下面，我们就阐述下这4中状态的流转：

{% asset_img 640-1576721967959.webp %}

```
(1)I--本地读请求：CPU读取变量x，如果其他核中的缓存有变量x，且缓存行的状态为M，则将该核心的变量x更新到内存，本核心的再从内存中读取取数据，加载到缓存行中，两个核心的缓存行状态都变成S；如果其他核心的缓存行状态为S或者E，本核心从内存中杜取数据，之后所有核心中的包含变量x的缓存行状态都变成S。

(2)I--本地读请求：CPU读取变量x，如果其他核中的缓存没有变量x，则本核心从内存中读取变量x，存入本核心的缓存行当中，该缓存行状态变成E；

(3)I--本地写请求：CPU读取写入变量x，如果其他核中没有此变量，则从内存中读取，在本核心中修改，此缓存行状态变为M；如果其他缓存行中有变量x，并且状态为M,则需要先将其他核心中的变量x写回内存，本核心再从内存中读取；如果其他缓存行中有变量x，并且状态为E/S，则将其他核心中的缓存行状态置为I，本核心在从内存中读取变量x，之后将本核心的缓存行置为M；

注意，一个缓存除在Invalid状态外都可以满足CPU的读请求，一个invalid的缓存行必须从主存中读取（变成S或者 E状态）来满足该CPU的读请求。

(4)S--远程写请求：多个核心共享变量X，其他核心将变量x修改，本核心中的缓存行不能使用，状态变为I；

(5)S--本地读请求：多个核心共享变量X，本核心读取本缓存中的变量x，状态不变；

(6)S--远程读请求：多个核心共享变量X，其他核心要读取变量X，从主内存中读取变量x，状态置为S，本核心状态S不变；

(7)S--本地写请求：多个核心共享变量X，本核心修改本缓存行中的变量x，必须先将其他核心中所拥有变量x的缓存行状态变成I，本核心缓存行状态置为M；该操作通常使用RequestFor Ownership (RFO)广播的方式来完成；

(8)E--远程读请求：只有本核心拥有变量x，其他核心也要读取变量x,从内存中读取变量x，并将所有拥有变量x的缓存行置为S状态；

(9)E--本地读请求：只有本核心拥有变量x，本核心需要读取变量x，读取本地缓存行中的变量x即可，状态不变依旧为E；

(10)E--远程写请求：只有本核心拥有变量x，其他核心需要修改变量x，其他核心从内存中读取变量x，进行修改，状态变成M，而本核心中缓存行变为状态I；

(11)E--本地写请求：只有本核心拥有变量x，本核心修改本缓存行中的变量x，状态置为M；

(12)M--本地写请求：只有本核心中拥有变量x，本核心进行修改x操作，缓存行状态不变；

(13)M--本地读请求：只有本核心中拥有变量x，本核心进行读取x操作，缓存行状态不变；

(14)M--远程读请求：只有本核心中拥有变量x，其他核心需要读取变量x,先将本核心中的变量x写回到内存中，在将本缓存行状态置为S，其他核心拥有变量x的缓存行状态也变为S；

(15)M--远程写请求：只有本核心中拥有变量x，其他和核心需要修改变量x，先将本核心中的变量x写回内存，再将本核心中缓存行置为I。其他核心的在从缓存行中读取变量x，修改后置为M；
```

以上就是MESI协议的状态流转；如果对状态流转还有疑问的话，还可以结合以下图例进行学习：

{% asset_img 640-1576722004805.webp %}

**1.3 CAS**

那么，CAS是什么呢？

在Java中，多线程之间如何保证数据的一致性？想必大部分都会异口同声地说出锁—-synchronized锁。在JDK1.5之前，的确是使用synchronized锁来保证数据的一致性。但是，synchronized锁是一种比较重的锁，俗称悲观锁。在较多线程的竞争下，加锁、释放锁会对系统性能产生很大的影响，而且一个线程持有锁，会导致其他线程的挂起，直至锁的释放。

那么，有没有比较轻的锁呢，答案是有的！与之相对应的是乐观锁！乐观锁虽然名称中带有锁，但实际在代码中是不加锁的，乐观锁大多实现体现在数据库sql层面，通常是的做法是：为数据增加一个版本标识，在表中增加一个 “version” 字段来实现。读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一。此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据。

```sql
update XXX_TABLE SET MONEY = 100 AND VERSION = 11 WHERE ID = 1 AND VERSION = 10;
```

这就是乐观锁！

上面说到了数据库层面的乐观锁，那么代码层面有没有类似的实现？答案是，有的！那就是我们本小节的主角—CAS；

CAS是一个CPU级别的指令，翻译为Compare And Swap比较并交换；

CAS是对内存中共享数据操作的一种指令，该指令就是用乐观锁实现的方式，对共享数据做原子的读写操作。原子本意是“不能被进一步分割的最小粒子”，而原子操作意为”不可被中断的一个或一系列操作”。原子变量能够保证原子性的操作，意思是某个任务在执行过程中，要么全部成功，要么全部失败回滚，恢复到执行之前的初态，不存在初态和成功之间的中间状态。

CAS有3个操作数，内存中的值V，预期内存中的值A，要修改成的值B。当内存值V和预期值相同时，就将内存值V修改为B，否则什么都不做。

例如：

```java
public class CasTest implements Runnable{

    private int memoryValue = 1;

    private int expectValue;

    private int updateValue;

    public CasTest(int expectValue,int updateValue){
        this.expectValue = expectValue;
        this.updateValue = updateValue;
    }

    public void run() {
        if(memoryValue==expectValue){
            this.memoryValue = updateValue;
            System.out.println("修改成功");
        }else {
            System.out.println("修改失败");
        }
    }

    public static void main(String[] agrs) throws InterruptedException {
        CasTest casTest1 = new CasTest(1,2);
        Thread t1 = new Thread(casTest1);
        t1.start();

        Thread t2= new Thread(casTest1);
        t2.start();

        t1.join();
        t2.join();
    }
}
```

在Java中，主要使用了Unsafe类来实现CAS操作，利用JNI来完成CPU指令的调用。JNI：java native interface为java本地调用，也就是说允许java调用其他计算机语言（例如：C、C++等）；

在java.util.concurrent.atomic包下(AtomicInteger为例)：

```java
public class AtomicInteger extends Number implements java.io.Serializable {

    private static final long serialVersionUID = 6214790243416807050L;

    private static final Unsafe unsafe = Unsafe.getUnsafe();

    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
}
```

实际最终调用了sun.misc.Unsafe类：

```java
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

可以看到Unsafe的compareAndSwapInt方法，使用了native修饰符，是一个本地方法调用，最终由C++代码来操作CPU。至于具体实现，有兴趣的朋友可以去参考openJDK中Unsafe类；

与synchronized锁相比较而言，CAS最大的优势就是非阻塞，在代码层面，多线程情况下不阻塞其他线程的执行，从而达到既保证数据的安全，又提高了系统的性能。

**1.4 Disruptor中的运用**

上面，说了分别说了CAS、缓存行、伪共享。接下来，就来看看再Disruptor中是如何使用的！

在多生产者的环境下，更新下一个可用的序列号地方，我们使用CAS（Compare And Swap）操作。

{% asset_img 640-1576722127388.webp %}

Disruptor中多生产者情况下，获取下一个可用序列号的实现:

```java
public final class MultiProducerSequencer extends AbstractSequencer{
    @Override
    public long next(int n){
        if (n < 1){
            throw new IllegalArgumentException("n must be > 0");
        }
        long current;
        long next;
        do{
            current = cursor.get();
            next = current + n;
            long wrapPoint = next - bufferSize;
            long cachedGatingSequence = gatingSequenceCache.get();
            if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current){
                long gatingSequence = Util.getMinimumSequence(gatingSequences, current);

                if (wrapPoint > gatingSequence){
                    waitStrategy.signalAllWhenBlocking();
                    LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                    continue;
                }
                gatingSequenceCache.set(gatingSequence);

            //对current,next进行compareAndSet，cursor就是序列号对象：
            } else if (cursor.compareAndSet(current, next)){
                break;
            }
        }while (true);
        return next;
    }
}
```

Disruptor通过缓存行填充的方式来解决伪共享：

```java
class LhsPadding{
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding{
    protected volatile long value;
}

class RhsPadding extends Value{
    protected long p9, p10, p11, p12, p13, p14, p15;
}

public class Sequence extends RhsPadding{}
```

Sequence是Disruptor中序列号对象，value是对象具体的序列值，通过上面的方式，value不会与其他需要操作的变量存在同一个缓存行中。

# 资料四（Disruptor之概览）

**概述**

“多核危机”驱动了并发编程的复兴，然后并发编程和一般的系统相比，复杂性有个很大梯度的上升。多线程开发很大困难在于：多个线程间存在依赖关系时，如何进行协调。依赖一方面是执行顺序的依赖，如某个线程执行需要依赖其他线程执行或其它线程的某些阶段执行结果，Java为我们提供的解决方案是：wait/notify、lock/condition、join、yield、Semaphore、CountDownLatch、CyclicBarrier以及JDK7新增的一个Phaser等；数据依赖主要是多个线程对同一资源并发修改导致的数据状态不一致问题，Java中主要依靠Lock和CAS两种方案，也就是我们熟知的悲观锁、乐观锁。

然而，当你在并发编程方面慢慢有些经验并开始在项目中使用时，你会发现仅仅依赖JDK提供的上面所说开发工具类是远远不够的， JDK提供的工具类都只能解决一个个功能“点”的问题。并发编程复杂性一个体现就是：多个顺序执行流在多核CPU中同时并行执行与我们已经习惯的单个数据顺序流执行的方式产生了很大的冲突。

好比：现在你开车从A地到B地去，传统的开发模式就像从A地到B地之间只存在一条公路，你只需要延着这个公路一直开下去就可以达到B地；假如经过多年发展，现在A地到B地横起有10条公路，纵起有10条公路，它们之间相互交叉形成错综复杂的公路网，你再开车从A地到B地就会存在太多的选择，可能从东南西北任何方向出发最终都能到达B地。这就体现了并发编程和传统编程复杂性的对比：传统编程由于只存在一个顺序执行流，可以很好的预判程序的执行流程；而并发编程存在太多的顺序执行流导致很难准确的预判出它们真正的执行流程，一旦出现问题也很难排查，就好比上面的例子第二种情况，你很难预判你开车的真正路线，而且可能存在每次路线都不一样情况。

我认为一个并发编程项目好坏其中一个关键核心就是：项目的整体结构是否清晰。很简单的一个例子，调用notify()方法唤醒挂起在指定对象上的休眠线程，如果没有一个清晰简单的架构设计，可能会导致在该对象上进行休眠的对象散落到系统中各处代码上，很难把控具体唤醒的是哪个线程从而与你的业务逻辑发生偏差导致bug的出现。当然，项目结构清晰在传统编程中也是非常看重的，只有结构清晰的架构才会让人易于理解，同时和他人沟通探讨时方便描述，但是在并发编程中这点尤为重要，因为并发编程的复杂性更高，没有一个清晰的结构设计，你可能经过大量测试修改暂时做出了一个看似没有bug的项目，但是后期需求变更或者是其他人来维护这个项目时，很难下手导致后期会引入大量的bug，而且不利于项目功能的扩展。

常用的并发编程使用的模型有并行模型、流水线模型、生产者/消费者模型、Actor模型等，采用模型设计一方面是因为这些模型都是大牛们经过长时间实际生产经验的积累总结出的并发编程方面一些好的解决方案；另一方面，采用模型设计可以解决相关人员之间沟通信息不对等问题，降低沟通学习成本。

并行模型是JDK8中Stream所采用的实现并发编程的方式，并行模型非常简单，就是为每个任务分配一个线程直到该任务执行结束，示意图如下：

{% asset_img 640-1576722373615.webp %}

并行模型太过简单导致对任务的精细化控制不足，一个任务可能会被分解为多个阶段，而每个阶段的子任务特性可能差别很大，这时并行模型就无能为力了。并行模型只适合于CPU密集型且任务中不含IO阻塞等情况的任务。这时，就演进出流水线模型，示意图如下：

{% asset_img 640-1576722417189.webp %}

流水线模型在实际的并发编程中使用比较常见，我们所说的Pipeline设计模型、Netty框架等都是这一思想的体现。

生产者/消费者模型在并发编程中也是使用频度非常高的一个模型，生产者/消费者模型可以很容易地将生产和消费进行解耦，优化系统整体结构，并且由于存在缓冲区，可以缓解两端性能不匹配的问题。

Actor模型其典型代表就是Akka，基于Akka可以轻松实现一个分布式异步数据处理集群系统，非常强大，后期我们有机会可以再深入讨论下Akka。

好了，说了这么多，终于要开始正题：Disruptor，官方宣传基于该框架构建的系统单线程可以支撑每秒处理600万订单，此框架真乃惊为天人。Disruptor在生产者/消费者模型上获得尽量高的吞吐量和尽量低的延迟，其目标就是在性能优化方面做到极致。国内国外都存在大量的知名项目在广泛使用，比如我们所熟知的strom底层就依赖Disruptor的实现，其在并发、缓存区、生产者/消费者模型、事务处理等方面都存在一些性能优秀的方案，因此是非常值得深入研究的。

**生产者/消费者模型**

生产者/消费者模型在编程中使用频度非常高的一个模型，生产者/消费者模型可以很容易地将生产和消费进行解耦，优化系统整体结构，并且由于存在缓冲区，可以缓解两端性能不匹配的问题。生产者/消费者和我们所熟悉的设计模式中的观察者模型很相似，生产者类似于被观察者，消费者类似于观察者，被观察者的任何变动都以事件的方式通知到观察者；同理，生产者生产的数据都要传递给消费者最终都要被消费者处理。

一般项目开发中，我们可以使用JDK提供的阻塞队列BlockingQueue很简单的实现一个生产者/消费者模型，其中生产者线程负责提交需求，消费者线程负责处理任务，二者之间通过共享内存缓冲区进行通信。

{% asset_img 640-1576722441766.webp %}

BlockingQueue实现类主要有两个：ArrayBlockingQueue和LinkedBlockingQueue，底层实现一个是基于数组的，一个是基于链表的，这种实现方式的差异导致了它们使用场景的不一样。在生产者/消费者模型中的缓存设计上肯定优先使用ArrayBlockingQueue，但是查看ArrayBlockingQueue底层源码会发现，读写操作通过重入锁实现同步，而且读写操作使用的是同一把锁，并没有实现读写锁分离；另外，锁本身的成本还是比较高的，锁容易导致线程上下文频繁的发生切换，了解CPU核存储硬件架构的可能会知道，每核CPU都会存在一个独享的高速缓存L1，假如线程切换到其它CPU上执行会导致之前CPU高速缓存L1中的数据不能再被使用，降低了高速缓存使用效率。因此，在高并发场景下，性能不是很优越。

```java
//向Queue中写入数据
public void put(E e) throws InterruptedException {
       checkNotNull(e);
       final ReentrantLock lock = this.lock;
       lock.lockInterruptibly();//可中断方式获取锁，实现同步
       try {
           while (count == items.length)
               notFull.await();
           insert(e);
       } finally {
           lock.unlock();
       }
}

//从Queue中取出数据
public E take() throws InterruptedException {
       final ReentrantLock lock = this.lock;
       lock.lockInterruptibly();//可中断方式获取锁，实现同步
       try {
           while (count == 0)
               notEmpty.await();
           return dequeue();
       } finally {
           lock.unlock();
       }
}
```

**Disruptor消息生产模型**

{% asset_img 640-1576722855401.webp %}

Producer生产出一个消息事件Event，需要放入到RingBuffer中，流程大致如下：

1、首先调用Sequencer.next()方法，获取RingBuffer上可用的序号用于将新生成的消息事件放入；

2、Sequencer首先对nextValue+1代表当前需要申请的RingBuffer序号(nextValue标记了之前已经申请过的序号,nextValue+1就是下一个可申请的序号)，但是nextValue+1指向的RingBuffer槽位存放的消息可能并没有被消费，如果直接返回这个序号给生产者，就会导致生产一方将该槽位的消息事件重新填充覆盖导致之前数据丢失，这里就需要一个判断：判断申请的RingBuffer序号代表的槽位之前的消息事件是否已被消费，判断逻辑如下：

```java
public long next(int n) {
    //n表示此次生产者期望获取多少个序号，通常是1
    if(n < 1) {
        throw new IllegalArgumentException("n must be > 0");
    }
    long nextValue = this.nextValue;
    //这里n一般是1，代表申请1个可用槽位，nextValue+n就代表了期望申请的可用槽位序号
    long nextSequence = nextValue + n;
    //减掉RingBuffer的bufferSize值，用于判断是否出现‘绕圈覆盖’
    long wrapPoint = nextSequence - bufferSize;
    //cachedValue缓存之前获取的最慢消费者消费到的槽位序号
    long cachedGatingSequence = this.cachedValue;
    //如果申请槽位序号-bufferSize比最慢消费者序号还大，代表生产者绕了一圈后又追赶上了消费者，这时候就不能继续生产了，否则把消费者还没消费的消息事件覆盖
    if(wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue) {
        /**
         cursor代表当前已经生产完成的序号，了解多线程可见性可能会知道：
         1、CPU和内存间速度不匹配，硬件架构上一般会在内存和CPU间还会存在L1、L2、L3三级缓存
         2、特别是L1高速缓存是CPU间相互独立不能共享的，线程操作可以看着基于L1缓存进行操作，就会导致线程间修改不会立即被其它线程感知，只有L1缓存的修改写入到主存然后其它线程将主存修改刷新到自己的L1缓存，这时线程1的修改才会被其它线程感知到
         3、线程修改对其它线程不能立即可见特别是在高并发下可能会带来些问题，JAVA中使用volatile可以解决可见性问题
         4、这里就是采用UNSAFE.putLongVolatile()插入一个StoreLoad内存屏障，具体可见JMM模型，主要保证cursor的真实值对所有的消费线程可见，避免不可见下消费线程无法消费问题
         */
        cursor.setVolatile(nextValue);
        long minSequence;
        //Util.getMinimumSequence(gatingSequences, nextValue)获取当前时刻所有消费线程中，消费最慢的序号
        //上面说过cachedValue是缓存的消费者最慢的序号
        //这样做目的：每次都去获取真实的最慢消费线程序号比较浪费资源，而是获取一批可用序号后，生产者只有使用完后，才继续获取当前最慢消费线程最小序号，重新获取最新资源
        while(wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue))) {
            //如果获取最新最慢消费线程最小序号后，依然没有可用资源，做两件事：
            //    1、唤醒waitStrategy上所有休眠线程，这里即是消费线程(避免因消费线程休眠而无法消费消息事件导致生产线程一直获取不到资源情况)
            //    2、自旋休眠1纳秒
            //可以看到，next()方法是一个阻塞接口，如果一直获取不到可用资源，就会一直阻塞在这里
            waitStrategy.signalAllWhenBlocking();
            LockSupport.parkNanos(1 L);
        }
        //有可用资源时，将当前最慢消费线程序号缓存到cachedValue中，下次再申请时就可不必再进入if块中获取真实的最慢消费线程序号，只有这次获取到的被生产者使用完才会继续进入if块
        this.cachedValue = minSequence;
    }
    //申请成功，将nextValue重新设置，下次再申请时继续在该值基础上申请
    this.nextValue = nextSequence;
    //返回申请到RingBuffer序号
    return nextSequence;
}
```

3、申请到可用序号后，提取RingBuffer中该序号中的Event，并重置Event状态为当前最新事件状态

4、重置完成后，调用Sequencer.publish()提交序号，提交序号主要就是修改cursor值，cursor标记已经生产完成序号，这样消费线程就可以来消费事件了

```java
@Override
public void publish(long sequence) {
    //修改cursor序号，消费者就可以进行消费
    cursor.set(sequence);
    //唤醒消费线程，比如消费线程消息到无可用消息时可能会进入休眠状态，当放入新消息就需要唤醒休眠的消费线程
    waitStrategy.signalAllWhenBlocking();
}
```

总结：消息事件生产主要包含三个步骤：

1、申请序号：表示从RingBuffer上获取可用的资源

2、填充事件：表示获取到RingBuffer上可用资源后，将新事件放入到该资源对应的槽位上

3、提交序号：表示第二部新事件放入到RingBuffer槽位全部完成，提交序号可供消费线程开始消费

**Disruptor消息处理模型**

{% asset_img 640-1576723548323.webp %}

消息处理端需要从RingBuffer中提取可用的消息事件，并注入到用户的业务逻辑中进行处理，流程大致如下：

1、消费端核心类是EventProcessor，它实现了Runnable接口，Disruptor在启动的时候会将所有注册上来的EventProcessor提交到线程池中执行，因此，一个EventProcessor可以看着一个独立的线程流用于处理RingBuffer上的数据

2、EventProcessor通过调用SequenceBarrier.waitFor()方法获取可用消息事件的序号，其实SequenceBarrier内部还是调用WaitStrategy.waitFor()方法，WaitStrategy等待策略主要封装如果获取消息时没有可用消息时如何处理的逻辑信息，是自旋、休眠、直接返回等，不同场景需要使用不同策略才能实现最佳的性能

```java
/**
* ProcessingSequenceBarrier中核心方法只有一个：waitFor(long sequence)，传入希望消费得到起始序号，返回值代表可用于消费处理的序号，一般返回可用序号>=sequence，但也不一定，具体看WaitStrategy实现
* 总结：
*      1、sequence：EventProcessor传入的需要进行消费的起始sequence
*      2、这里并不保证返回值availableSequence一定等于given sequence，他们的大小关系取决于采用的WaitStrategy
*          a.YieldingWaitStrategy在自旋100次尝试后，会直接返回dependentSequence的最小seq，这时并不保证返回值>=given sequence
*          b.BlockingWaitStrategy则会阻塞等待given sequence可用为止，可用并不是说availableSequence == given sequence，而应当是指 >=
*          c.SleepingWaitStrategy:首选会自旋100次，然后执行100次Thread.yield()，还是不行则LockSupport.parkNanos(1L)直到availableSequence >= given sequence
*/
@Override
public long waitFor(final long sequence)
throws AlertException, InterruptedException, TimeoutException {
    checkAlert();
    //调用WaitStrategy获取RingBuffer上可用消息序号，无可消费消息是该接口可能会阻塞，具体逻辑由WaitStrategy实现
    long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
    if(availableSequence < sequence) {
        return availableSequence;
    }
    //获取消费者可以消费的最大的可用序号，支持批处理效应，提升处理效率。
    //当availableSequence > sequence时，需要遍历 sequence --> availableSequence，找到最前一个准备就绪，可以被消费的event对应的seq。
    //最小值为：sequence-1
    return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
```

3、通过waitFor()返回的是一批可用消息的序号，比如申请消费7好槽位，waitFor()返回的可能是8表示从6到8这一批数据都已生产完毕可以进行消费

4、EventProcessor按照顺序从RingBuffer中取出消息事件，然后调用EventHandler.onEvent()触发用户的业务逻辑进行消息处理

```java
while(true) {
    try {
        //读取可消费消息序号
        final long availableSequence = sequenceBarrier.waitFor(nextSequence);
        if(batchStartAware != null) {
            batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
        }
        while(nextSequence <= availableSequence) {
            //循环提取所有可供消费的消息事件
            event = dataProvider.get(nextSequence);
            //将提取的消息事件注入到封装用户业务逻辑的Handler中
            eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
            nextSequence++;
        }
        sequence.set(availableSequence);
    }
}
```

5、当这批次的消息处理完成后，继续重复上面操作调用waitFor()继续获取可用的消息序号，周而复始

好了，这节主要对Disruptor的生产模型和消费模型进行了一个简单的介绍，后面会逐渐对Disruptor涉及到的每个核心组件进行分析，了解它们优秀的设计思想。

# 资料五（Disruptor源码解析 + 实战）

**juc下的队列**

{% asset_img 640-1576725625080.webp %}

1：从上图可以看出，juc下的队列基本采用加锁方式保证线程安全。通过不加锁的方式实现的队列都是无界的（无法保证队列的长度在限定的范围）。而加锁的方式可以实现有界队列。在稳定性要求特别高的系统中，为了防止生产者速度过快，导致内存溢出，只能选择有界队列。

2：加锁的方式通常严重影响性能。线程会因为竞争不到锁而被挂起，等锁被释放的时候，线程又会被恢复，这个过程中存在着很大的开销，并且通常会有较长时间的中断，因为当一个线程正在等待锁时，它不能做任何其他事情。如果一个线程在持有锁的情况下被延迟执行，例如发生了缺页错误、调度延迟或者其它类似情况，那么所有需要这个锁的线程都无法执行下去。如果被阻塞线程的优先级较高，而持有锁的线程优先级较低，就会发生优先级反转。

3：有界队列通常采用数组实现。但是采用数组实现又会引发另外一个问题false sharing(伪共享)。关于什么是伪共享之前的文章已经讲解。

**Disruptor**

**Disruptor是什么**

1：Disruptor是英国外汇交易公司LMAX开发的一个高性能队列，研发的初衷是解决内存队列的延迟问题（在性能测试中发现竟然与I/O操作处于同样的数量级）

2：Disruptor实现对了队列的功能并且是一个有界队列。可以用于生产者-消费者模型。

**Disruptor为什么快**

1：数据结构采用ringbuffer。其实可以理解成一个数组entries。每一个slot存储一个事件对象。初始化时，就已经分配好内存，而且新发布的数据只会覆盖，所以更少的GC。

2：Disruptor采用缓存行填充机制的形式解决了fasle sharing。保证读取变量的时候从cache line读取。

3：Disroptor中维护了一个long类型的sequence(序列)。每次根据位运算操作可以快速定位到实际slot，sequece&(entries.length-1)=index，比如一共有4槽，9&(8-1)=1。提示：队列的大小必须要2^n。

4：线程同时访问，由于他们都通过sequence访问ringBuffer，通过CAS取代了加锁，这也是并发编程的原则：把同步块最小化到一个变量上。这个sequence一直采用自增的形式。

**Disruptor核心类**

1：RingBuffer：Disruptor最主要的组件，仅仅负责存储和更新事件对象。

2：Sequence：Disruptor使用Sequence来表示一个特殊组件处理的序号。和Disruptor一样，每一个消费者（EventProcessor）都维持着一个Sequence。大部分的并发代码依赖这这个值。这个类维护了一个long类型的value，采用的unsafe进行的更新操作。

3：Sequencer：这是Disruptor真正的核心。实现了这个接口的两种生产者（单生产者和多生产者）均实现了所有的并发算法，为了在生产者和消费者之间进行准确快速的数据传递。

4：SequenceBarrier：由Sequencer生成，并且包含了已经发布的Sequence的引用，这些Sequence源于Sequencer和一些独立的消费者的Sequence。它包含了决定是否有供消费者消费的Event的逻辑。用来权衡当消费者无法从RingBuffer里面获取事件时的处理策略。（例如：当生产者太慢，消费者太快，会导致消费者获取不到新的事件会根据该策略进行处理，默认会堵塞）

5：WaitStrategy：决定一个消费者将如何等待生产者将Event置入Disruptor的策略。用来权衡当生产者无法将新的事件放进RingBuffer时的处理策略。（例如：当生产者太快，消费者太慢，会导致生产者获取不到新的事件槽来插入新事件，则会根据该策略进行处理，默认会堵塞）

6：Event：从生产者到消费者过程中所处理的数据单元。Disruptor中没有代码表示Event，因为它完全是由用户定义的。

7：EventProcessor：主要事件循环，处理Disruptor中的Event，并且拥有消费者的Sequence。它有一个实现类是BatchEventProcessor，包含了event loop有效的实现，并且将回调到一个EventHandler接口的实现对象。

8：EventHandler：由用户实现并且代表了Disruptor中的一个消费者的接口。

9：WorkHandler：在work模式下使用。由用户实现并且代表了Disruptor中的多个消费者的接口。

10：WorkProcessor：确保每个sequence只被一个processor消费，在同一个WorkPool中的处理多个WorkProcessor不会消费同样的sequence。

11：WorkerPool：一个WorkProcessor池，其中WorkProcessor将消费Sequence，所以任务可以在实现WorkHandler接口的worker之间移交

12：LifecycleAware：当BatchEventProcessor启动和停止时，实现这个接口用于接收通知。

**Sequence(序列)**

{% asset_img 640-1576725694032.webp %}

1：Sequence是用来标记事件发布者和事件消费者的位置。

2：Sequence真正计数的是value，采用缓冲行填充防止false sharing。在value的前后各有7个long型的填充值，这些值在这里的作用是做cpu cache line填充，防止发生伪共享。最坏的情况就是value位于cache line的头或者尾。

**框架类结构关系图**

{% asset_img 640-1576725713396.webp %}

**Cursored  获取当前序列值**

```java
public interface Cursored{
    /**
     * 获取当前序列值
     */
    long getCursor();
}
```

1：Cursored接口只提供了一个获取当前序列值的方法。

```java
public interface Sequenced{
    //获取队列的大小
    int getBufferSize();
    //判断队列中是否还有可用的容量
    boolean hasAvailableCapacity(final int requiredCapacity);
    //获取队列中剩余的有效容量
    long remainingCapacity();
    //申请下一个sequence，用于事件发布者发布数据，申请失败则自旋
    long next();
    //申请n个sequence，用于事件发布者发布数据，申请失败则自旋
    long next(int n);
    //尝试获取一个sequence
    long tryNext() throws InsufficientCapacityException;
    //尝试获取n个sequence
    long tryNext(int n) throws InsufficientCapacityException;
    //发布sequence
    void publish(long sequence);
    //批量发布sequence
    void publish(long lo, long hi);
}
```

**Sequencer**

```java
public interface Sequencer extends Cursored, Sequenced{
    //游标初始值
    long INITIAL_CURSOR_VALUE = -1L;
    //初始化RingBuffer为指定的sequence
    void claim(long sequence);
    //消费者调用，判断sequence是否可以消费
    boolean isAvailable(long sequence);
    //将sequence添加到gating sequences中
    void addGatingSequences(Sequence... gatingSequences);
    //从gating sequences中移除指定的sequence
    boolean removeGatingSequence(Sequence sequence);
    //事件处理者用来追踪ringBuffer中可以用的sequence
    SequenceBarrier newBarrier(Sequence... sequencesToTrack);
    //事件发布者获取gating sequence中最小的sequence
    long getMinimumSequence();
    //消费者用来获取从nextSequence到availableSequence之间最大的sequence。如果是多线程生产者判断nextSequence是否可用，否则返回nextSequence-1。单线程直接返回availableSequence
    long getHighestPublishedSequence(long nextSequence, long availableSequence);
    //我也不知道干啥的
    <T> EventPoller<T> newPoller(DataProvider<T> provider,Sequence... gatingSequences);
}
```

1：Sequencer中的方法大多是给事件发布者使用。newBarrier()给事件处理者使用。

**AbstractSequencer 管理事件处理者序列和事件发布者发布序列。**

```java
public abstract class AbstractSequencer implements Sequencer {
    //用来对gatingSequences做原子操作的。Sequence[]里面存储的是事件处理者处理到的序列。
    //如果不懂AtomicReferenceFieldUpdater请www.google.com
    private static final AtomicReferenceFieldUpdater<AbstractSequencer, Sequence[]> SEQUENCE_UPDATER =
        AtomicReferenceFieldUpdater.newUpdater(AbstractSequencer.class, Sequence[].class, "gatingSequences");
    //队列大小
    protected final int bufferSize;
    //等待策略
    protected final WaitStrategy waitStrategy;
    //事件发布者的已经发布到的sequence       
    protected final Sequence cursor = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);
    //事件处理者处理到的序列对象
    protected volatile Sequence[] gatingSequences = new Sequence[0];

    /**
     *检查队列大小是否是2^n，判断buffersize大小
     */
    public AbstractSequencer(int bufferSize, WaitStrategy waitStrategy) {
        if (bufferSize < 1) {
            throw new IllegalArgumentException("bufferSize must not be less than 1");}
        if (Integer.bitCount(bufferSize) != 1) {
            throw new IllegalArgumentException("bufferSize must be a power of 2"); }
        this.bufferSize = bufferSize;
        this.waitStrategy = waitStrategy;
    }

    /**
     * 获取事件发布者的序列
     */
    @Override
    public final long getCursor() {
        return cursor.get();
    }

    /**
     * 获取大小
     */
    @Override
    public final int getBufferSize() {
        return bufferSize;
    }

    /**
     * 把事件消费者序列维护到gating sequence
     */
    @Override
    public final void addGatingSequences(Sequence... gatingSequences) {
        SequenceGroups.addSequences(this, SEQUENCE_UPDATER, this, gatingSequences);
    }

    /**
     *  从gating sequence移除序列
     */
    @Override
    public boolean removeGatingSequence(Sequence sequence) {
        return SequenceGroups.removeSequence(this, SEQUENCE_UPDATER, sequence);
    }

    /**
     * 获取gating sequence中事件处理者处理到最小的序列值
     */
    @Override
    public long getMinimumSequence() {
        return Util.getMinimumSequence(gatingSequences, cursor.get());
    }

    /**
     * 创建了一个序列栅栏
     */
    @Override
    public SequenceBarrier newBarrier(Sequence... sequencesToTrack) {
        return new ProcessingSequenceBarrier(this, waitStrategy, cursor, sequencesToTrack);
    }

    /**
     * 这个方法不解释，我也不知道目前用来干嘛的。有知道的大佬可以赐教一下。谢谢
     */
    @Override
    public <T> EventPoller<T> newPoller(DataProvider<T> dataProvider, Sequence... gatingSequences) {
        return EventPoller.newInstance(dataProvider, this, new Sequence(), cursor, gatingSequences);
    }
    //重写toString
    @Override
    public String toString() {
        return "AbstractSequencer{" +
            "waitStrategy=" + waitStrategy +
            ", cursor=" + cursor +
            ", gatingSequences=" + Arrays.toString(gatingSequences) +
            '}';
    }
}
```

**SingleProducerSequencer 单线程事件发布者。**

1：从上面的图可以看出SingleProducerSequencer间接继承了AbstractSequencer。

2：SingleProducerSequencerFields维护事件发布者发布的序列和事件处理者处理到的最小序列。

3：SingleProducerSequencerPad缓冲行填充，防止false sharing。

**next()申请序列**

```java
//该方法是事件发布者申请序列
public long next(int n) {
    if (n < 1) {
        throw new IllegalArgumentException("n must be > 0");
    }
     //获取事件发布者发布到的序列值
    long nextValue = this.nextValue;
    long nextSequence = nextValue + n;
    //wrap 代表申请的序列绕一圈以后的位置
    long wrapPoint = nextSequence - bufferSize;
    //获取事件处理者处理到的序列值
    long cachedGatingSequence = this.cachedValue;
    /** 1.事件发布者要申请的序列值大于事件处理者当前的序列值且事件发布者要申请的序列值减去环的长度要小于事件处理者的序列值。
      * 2.满足(1)，可以申请给定的序列。
      * 3.不满足(1)，就需要查看一下当前事件处理者的最小的序列值(可能有多个事件处理者)。如果最小序列值大于等于
      * 当前事件处理者的最小序列值大了一圈，那就不能申请了序列(申请了就会被覆盖)，
      * */
    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue) {
        //wrapPoint > cachedGatingSequence 代表绕一圈并且位置大于事件处理者处理到的序列
        //cachedGatingSequence > nextValue 说明事件发布者的位置位于事件处理者的屁股后面
        //维护父类中事件生产者的序列
        cursor.setVolatile(nextValue);
        long minSequence;
        //如果事件生产者绕一圈以后大于事件处理者的序列，那么会在此处自旋
        while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue))) {
            LockSupport.parkNanos(1L);
        }
        //缓存最小值
        this.cachedValue = minSequence;
    }
    this.nextValue = nextSequence;
    return nextSequence;
}

//事件发布调用的方法。唤醒阻塞的消费者
public void publish(long sequence) {
    cursor.set(sequence);
    waitStrategy.signalAllWhenBlocking();
}
```

**实战单线程生产者**

```java
public static void main(String[] args) {
    /**
     * Create a new Disruptor.
     * @param eventFactory 事件对象的数据
     * @param ringBufferSize 数组大小，必须是2^n
     * @param threadFactory 线程工厂
     * @param producerType 生产者策略。ProducerType.SINGLE和ProducerType.MULTI 单个生产者还是多个生产者.
     * @param waitStrategy 等待策略。用来平衡事件发布者和事件处理者之间的处理效率。提供了八种策略。默认是BlockingWaitStrategy
     */
    //初始化的逻辑大概是创建根据ProducerType初始化创造SingleProducerSequencer或MultiProducerSequencer。
    //初始化Ringbuffer的时候会根据buffsiz把事件对象放入entries数组。
    Disruptor<TradeBO> disruptor = new Disruptor<>(() -> new TradeBO(), 2,
                                                   r -> {
                                                       Thread thread = new Thread(r);
                                                       thread.setName("实战单线程生产者");
                                                       return thread;
                                                   }, ProducerType.SINGLE, new BlockingWaitStrategy());
    //关联事件处理者。初始化BatchEventProcessor。把事件处理者加入gating sequence
    disruptor.handleEventsWith(new ConsumerA());
    disruptor.handleEventsWith(new ConsumerB());
    //启动消费者线程。BatchEventProcessor间接实现了Runnable。所以这一步就是启动线程。如果事件发布太快，消费太慢会根据不同的waitstrategy等待。
    disruptor.start();
    //发布事件
    for (int i = 1; i < 10; i++) {
        int finalI = i;
        //初始化了EventTranslator。意思就是给最开始初始化的对象赋值
        EventTranslator eventTranslator = (EventTranslator<TradeBO>) (event, sequence) -> {
            event.setId(finalI);
            event.setPrice((double) finalI);
        };
        //发布首先要申请序列，如果申请不到会自旋。
        disruptor.publishEvent(eventTranslator);
    }
    disruptor.shutdown();
}


class ConsumerB implements EventHandler<TradeBO> {
    @Override
    public void onEvent(TradeBO event, long sequence,
                        boolean endOfBatch) throws Exception {
        System.out.println("ConsumerB id=" + event.getId() + "price=" + event.getPrice());
    }
}
class ConsumerA implements EventHandler<TradeBO> {
    @Override
    public void onEvent(TradeBO event, long sequence,
                        boolean endOfBatch) throws Exception {
        System.out.println("ConsumerB id=" + event.getId() + "   price=" + event.getPrice());
    }
}

@Data
public class TradeBO {
    private Integer id;
    private Double price;
}
```

**MultiProducerSequencer**

**成员变量**

```java
//获取unsafe
private static final Unsafe UNSAFE = Util.getUnsafe();
//获取int[]的偏移量
private static final long BASE = UNSAFE.arrayBaseOffset(int[].class);
//获取元素的大小，也就是int的大小4个字节
private static final long SCALE = UNSAFE.arrayIndexScale(int[].class);
//gatingSequenceCache是gatingSequence。用来标识事件处理者的序列
private final Sequence gatingSequenceCache = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);
//availableBuffer用来追踪每个槽的状态
private final int[] availableBuffer;
private final int indexMask;
//转了几圈
private final int indexShift;
```

**构造函数**

```java
public MultiProducerSequencer(int bufferSize, final WaitStrategy waitStrategy) {
    //初始化父类
    super(bufferSize, waitStrategy);
    //初始化availableBuffer
    availableBuffer = new int[bufferSize];
    indexMask = bufferSize - 1;
    indexShift = Util.log2(bufferSize);
    //这个逻辑是。计算availableBuffer中每个元素的偏移量
    //定位数组每个值的地址就是(index * SCALE) + BASE
    initialiseAvailableBuffer();
}
private void initialiseAvailableBuffer() {
    for (int i = availableBuffer.length - 1; i != 0; i--) {
        setAvailableBufferValue(i, -1);
    }
    setAvailableBufferValue(0, -1);
}
private void setAvailableBufferValue(int index, int flag) {
    long bufferAddress = (index * SCALE) + BASE;
    //修改内存偏移地址为bufferAddress的值，改为flag
    UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);
}
```

**next()申请序列**

```java
public long next(int n) {
    if (n < 1) {
        throw new IllegalArgumentException("n must be > 0");
    }
    long current;
    long next;
    do {
        //获取事件发布者发布序列
        current = cursor.get();
        //新序列位置
        next = current + n;
        //wrap 代表申请的序列绕一圈以后的位置
        long wrapPoint = next - bufferSize;
        //获取事件处理者处理到的序列值
        long cachedGatingSequence = gatingSequenceCache.get();
        /** 1.事件发布者要申请的序列值大于事件处理者当前的序列值且事件发布者要申请的序列值减去环的长度要小于事件处理者的序列值。
         * 2.满足(1)，可以申请给定的序列。
         * 3.不满足(1)，就需要查看一下当前事件处理者的最小的序列值(可能有多个事件处理者)。如果最小序列值大于等于
         * 当前事件处理者的最小序列值大了一圈，那就不能申请了序列(申请了就会被覆盖)，
         * */
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current) {
            //wrapPoint > cachedGatingSequence 代表绕一圈并且位置大于事件处理者处理到的序列
            //cachedGatingSequence > current 说明事件发布者的位置位于事件处理者的屁股后面

            //获取最小的事件处理者序列
            long gatingSequence = Util.getMinimumSequence(gatingSequences, current);
            if (wrapPoint > gatingSequence) {
                LockSupport.parkNanos(1);
                continue;
            }
            //赋值
            gatingSequenceCache.set(gatingSequence);
            //通过cas修改
        } else if (cursor.compareAndSet(current, next)) {
            break;
        }
    }
    while (true);

    return next;
}
```

**publish()事件发布**

```java
public void publish(final long sequence) {
    //这里的操作逻辑大概是修改数组中的序列值
    setAvailable(sequence);
    waitStrategy.signalAllWhenBlocking();
}

private void setAvailable(final long sequence) {
    setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence));
}
//计算数组中位置 sequence&(buffsize-1)
private int calculateIndex(final long sequence) {
    return ((int) sequence) & indexMask;
}
//计算数组中的存储的数据
private int calculateAvailabilityFlag(final long sequence) {
    return (int) (sequence >>> indexShift);
}

private void setAvailableBufferValue(int index, int flag) {
    long bufferAddress = (index * SCALE) + BASE;
    UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);
}
```

**MultiProducerSequencer和SingleProducerSequencer区别**

1：SingleProducerSequencer内部维护cachedValue(事件消费者序列)，nextValue(事件发布者序列)。并且采用padding填充。这个类是线程不安全的。
2：MultiProducerSequencer每次获取序列都是从Sequence中获取的。Sequence中针对value的操作都是原子的。

**RingBuffer**

{% asset_img 640-1576726006366.webp %}

**EventSequencer**

```java
//这个接口是一个空方法
public interface EventSequencer<T> extends DataProvider<T>, Sequenced{  
}  
```

**DataProvider**

```java
//DataProvider 提供了根据序列获取对应的对象
//有两个地方调用。这个Event对象需要被生产者获取往里面填充数据。第二个是在消费时，获取这个Event对象用于消费。
public interface DataProvider<T>{
    T get(long sequence);
}
```

**EventSink  这个类提供了各种发布的姿势。**

1：EventSink接口是用来发布Event的，在发布的同时，调用绑定的Translator来初始化并填充Event。

2：填充Event是通过实现EventTranslator，EventTranslatorOneArg，EventTranslatorTwoArg，EventTranslatorThreeArg，EventTranslatorVararg这些EventTranslator来做的。

3：发布流程：申请下一个序列->申请成功则获取对应槽的Event->利用translator初始化并填充对应槽的Event->发布Event 。translator用户实现，用于初始化Event。

**RingBufferPad  用于缓存行填充**

**RingBufferFields 这个类的逻辑比较重要，讲解了event在数组中存储位置**

```java
abstract class RingBufferFields<E> extends com.lmax.disruptor.RingBufferPad {
    //Buffer数组填充
    private static final int BUFFER_PAD;
    //Buffer数组起始基址
    private static final long REF_ARRAY_BASE;
    //数组引用每个引用占用的大小=2^REF_ELEMENT_SHIFT
    private static final int REF_ELEMENT_SHIFT;
    private static final Unsafe UNSAFE = Util.getUnsafe();

    static {
        //获取Object[]引用大小。我本机4字节
        final int scale = UNSAFE.arrayIndexScale(Object[].class);
        if (4 == scale) {
            REF_ELEMENT_SHIFT = 2;
        } else if (8 == scale) {
            REF_ELEMENT_SHIFT = 3;
        } else {
            throw new IllegalStateException("Unknown pointer size");
        }
        //填充32或者16
        BUFFER_PAD = 128 / scale;
        // 计算Buffer数组起始基址。我本机是从32开始
        REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);
    }

    private final long indexMask;
    //保存了RingBuffer每个槽的Event对象。这个entries不会被修改。ps:引用不会被修改
    private final Object[] entries;
    protected final int bufferSize;
    //sequencer=SingleProducerSequencer or MultiProducerSequencer的引用
    protected final Sequencer sequencer;

    RingBufferFields(
        EventFactory<E> eventFactory,
        Sequencer sequencer) {
        this.sequencer = sequencer;
        this.bufferSize = sequencer.getBufferSize();

        if (bufferSize < 1) {
            throw new IllegalArgumentException("bufferSize must not be less than 1");
        }
        if (Integer.bitCount(bufferSize) != 1) {
            throw new IllegalArgumentException("bufferSize must be a power of 2");
        }

        this.indexMask = bufferSize - 1;
        this.entries = new Object[sequencer.getBufferSize() + 2 * BUFFER_PAD];
        fill(eventFactory);
    }
    //填充entries
    private void fill(EventFactory<E> eventFactory) {
        for (int i = 0; i < bufferSize; i++) {
            entries[BUFFER_PAD + i] = eventFactory.newInstance();
        }
    }

    @SuppressWarnings("unchecked")
    protected final E elementAt(long sequence) {
        return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
    }
}
```

**SequenceBarrier接口 消费者使用**

```java
public interface SequenceBarrier {

    /**
         * 等待一个序列变为可用，然后消费这个序列。消费线程中使用
         */
    long waitFor(long sequence) throws AlertException, InterruptedException, TimeoutException;

    /**
         * 获取当前可以读取的序列值。
         */
    long getCursor();
    /**
         * 当前栅栏是否发过通知。
         */
    boolean isAlerted();           
    /**
         * 通知消费者状态变化，然后停留在这个状态上，直到状态被清除。
         */
    void alert();
    /**
         * 清楚通知状态。
         */
    void clearAlert();
    /**
         * 检测是否发生了通知，如果已经发生了抛出AlertException异常。
         */
    void checkAlert() throws AlertException;
} 
```

**ProcessingSequenceBarrier**

```java
final class ProcessingSequenceBarrier implements SequenceBarrier {
//等待策略
private final WaitStrategy waitStrategy;
//当消费者之前没有依赖关系的时候，那么dependentSequence=cursorSequence
//存在依赖关系的时候，dependentSequence 里存放的是一组依赖的Sequence，get方法得到的是最小的序列值
//所谓的依赖关系是有两个消费者A、B，其中B需要在A之后进行消费，这A的序列就是B需要依赖的序列，因为B的消费速度不能超过A。
private final Sequence dependentSequence;
//判断是否执行shutdown
private volatile boolean alerted = false;
//cursorSequence 代表的是写指针。代表事件发布者发布到那个位置
private final Sequence cursorSequence;
//sequencer=SingleProducerSequencer or MultiProducerSequencer的引用
private final Sequencer sequencer;

ProcessingSequenceBarrier(
        final Sequencer sequencer,
        final WaitStrategy waitStrategy,
        final Sequence cursorSequence,
        final Sequence[] dependentSequences) {
    this.sequencer = sequencer;
    this.waitStrategy = waitStrategy;
    this.cursorSequence = cursorSequence;
    if (0 == dependentSequences.length) {
        dependentSequence = cursorSequence;
    } else {
        dependentSequence = new FixedSequenceGroup(dependentSequences);
    }
}

@Override
public long waitFor(final long sequence)
        throws AlertException, InterruptedException, TimeoutException {
    //检查是否中断
    checkAlert();
    //根据不同的策略获取可用的序列
    long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
    //判断申请的序列和可用的序列大小
    if (availableSequence < sequence) {
        return availableSequence;
    }
    //如果是单线程生产者直接返回availableSequence
    //多线程生产者判断是否可用，不可用返回sequence-1
    return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
//获取当前序列
@Override
public long getCursor() {
    return dependentSequence.get();
}
//判断是否中断
@Override
public boolean isAlerted() {
    return alerted;
}
//中断
@Override
public void alert() {
    alerted = true;
    waitStrategy.signalAllWhenBlocking();
}
//清除中断
@Override
public void clearAlert() {
    alerted = false;
}
//检查是否中断
@Override
public void checkAlert() throws AlertException {
    if (alerted) {
        throw AlertException.INSTANCE;
     }
   }
}
```

**事件处理 EventProcessor**

```java
public interface EventProcessor extends Runnable{
    //获取事件处理器使用的序列引用。 
    Sequence getSequence();
    //中断
    void halt();
    //判断是否运行
    boolean isRunning();
}
```

**BatchEventProcessor event模式单线程处理**

```java
//重点讲run方法，其它方法都比较简单
public final class BatchEventProcessor<T>
    implements EventProcessor {

    public void run() {
        //启动任务
        if (running.compareAndSet(IDLE, RUNNING)) {
            //清除中断状态
            sequenceBarrier.clearAlert();
            //判断一下消费者是否实现了LifecycleAware ,如果实现了这个接口，那么此时会发送一个启动通知
            notifyStart();
            try {
                //判断任务是否启动
                if (running.get() == RUNNING) {
                    //处理事件
                    processEvents();
                }
            } finally {
                //判断一下消费者是否实现了LifecycleAware ,如果实现了这个接口，那么此时会发送一个停止通知
                notifyShutdown();
                //重新设置状态
                running.set(IDLE);
            }
        } else {
            // 线程已经启动
            if (running.get() == RUNNING) {
                throw new IllegalStateException("Thread is already running");
            } else {
                //这里就是  notifyStart();notifyShutdown();
                earlyExit();
            }
        }
    }

    private void processEvents() {
        //定义一个event
        T event = null;
        //获取要申请的序列
        long nextSequence = sequence.get() + 1L;
        //循环处理事件。除非超时或者中断。
        while (true) {
            try {
                //根据等待策略来等待可用的序列值。 
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null) {
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }
                //根据可用的序列值获取事件。批量处理nextSequence到availableSequence之间的事件。
                while (nextSequence <= availableSequence) {
                    //获取事件
                    event = dataProvider.get(nextSequence);
                    //触发事件
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }
                //设置事件处理者处理到的序列值。事件发布者会根据availableSequence判断是否发布事件 
                sequence.set(availableSequence);
            } catch (final TimeoutException e) {
                //超时异常
                notifyTimeout(sequence.get());
            } catch (final AlertException ex) {
                //中断异常
                if (running.get() != RUNNING) {
                    break;
                }
            } catch (final Throwable ex) {
                //这里可能用户消费者事件出错。如果自己实现了ExceptionHandler那么就不会影响继续消费
                exceptionHandler.handleEventException(ex, nextSequence, event);
                //如果出现异常则设置为nextSequence
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    }
}
```

**WorkProcessor  work模式多线程处理**

```java
public void run() {
    //判断线程是否启动
    if (!running.compareAndSet(false, true)) {
        throw new IllegalStateException("Thread is already running");
    }
    //清除中断状态
    sequenceBarrier.clearAlert();
    //判断一下消费者是否实现了LifecycleAware ,如果实现了这个接口，那么此时会发送一个启动通知
    notifyStart();
    //事件处理标志
    boolean processedSequence = true;
    long cachedAvailableSequence = Long.MIN_VALUE;
    long nextSequence = sequence.get();
    T event = null;
    while (true) {
        try {
           //判断上一个事件是否已经处理完毕。  
            if (processedSequence) {
                //置为false
                processedSequence = false;
                do {
                    //获取下一个序列
                    nextSequence = workSequence.get() + 1L;
                    //更新当前已经处理到的
                    sequence.set(nextSequence - 1L);
                }
            //多个WorkProcessor共享一个workSequence，可以实现互斥消费，因为只有一个线程可以CAS更新成功
                while (!workSequence.compareAndSet(nextSequence - 1L, nextSequence));
            }
           //检查序列值是否需要申请。
            if (cachedAvailableSequence >= nextSequence) {
                //获取事件
                event = ringBuffer.get(nextSequence);
               //交给workHandler处理事件。  
                workHandler.onEvent(event);
               //设置事件处理完成标识
                processedSequence = true;
            } else {
                //申请可用序列
                cachedAvailableSequence = sequenceBarrier.waitFor(nextSequence);
            }
        } catch (final TimeoutException e) {
            notifyTimeout(sequence.get());
        } catch (final AlertException ex) {
            if (!running.get()) {
                break;
            }
        } catch (final Throwable ex) {
            //设置异常事件处理
            exceptionHandler.handleEventException(ex, nextSequence, event);
            processedSequence = true;
        }
    }
    //同上
    notifyShutdown();
    //停止
    running.set(false);
}
```

**WorkerPool**

1：多个WorkProcessor组成一个WorkerPool。
2：维护workSequence事件处理者处理的序列。

**waitStrategy 等待策略**

BlockingWaitStrategy：默认的等待策略。利用锁和等待机制的WaitStrategy，CPU消耗少，但是延迟比较高

BusySpinWaitStrategy：自旋等待。这种策略会利用CPU资源来避免系统调用带来的延迟抖动，当线程可以绑定到指定CPU(核)的时候可以使用这个策略。

LiteBlockingWaitStrategy：实现方法也是阻塞等待

SleepingWaitStrategy：是另一种较为平衡CPU消耗与延迟的WaitStrategy，在不同次数的重试后，采用不同的策略选择继续尝试或者让出CPU或者sleep。这种策略延迟不均匀。

TimeoutBlockingWaitStrategy：实现方法是阻塞给定的时间，超过时间的话会抛出超时异常。

YieldingWaitStrategy：实现方法是先自旋(100次)，不行再临时让出调度(yield)。和SleepingWaitStrategy一样也是一种高性能与CPU资源之间取舍的折中方案，但这个策略不会带来显著的延迟抖动。

PhasedBackoffWaitStrategy：实现方法是先自旋(10000次)，不行再临时让出调度(yield)，不行再使用其他的策略进行等待。可以根据具体场景自行设置自旋时间、yield时间和备用等待策略。

**实战多线程消费者**

```java
public static void main(String[] args) {
    //创建一个RingBuffer，注意容量是2。
    RingBuffer<TradeBO> ringBuffer = RingBuffer.createSingleProducer(() -> new TradeBO(), 2);
    //创建2个WorkHandler其实就是创建2个WorkProcessor
    WorkerPool<TradeBO> workerPool =
        new WorkerPool<TradeBO>(ringBuffer, ringBuffer.newBarrier(),
                                new IgnoreExceptionHandler(),
                                new ConsumerC(), new ConsumerD());
    //将WorkPool的工作序列集设置为ringBuffer的追踪序列。
    ringBuffer.addGatingSequences(workerPool.getWorkerSequences());
    //创建一个线程池用于执行Workhandler。
    Executor executor = Executors.newFixedThreadPool(4);
    //启动WorkPool。
    workerPool.start(executor);
    //往RingBuffer上发布事件
    for (int i = 0; i < 4; i++) {
        int finalI = i;
        EventTranslator eventTranslator = (EventTranslator<TradeBO>) (event, sequence) -> {
            event.setId(finalI);
            event.setPrice((double) finalI);
        };
        ringBuffer.publishEvent(eventTranslator);
        System.out.println("发布[" + finalI + "]");
    }
}
//程序执行结果。可以看出，多个线程消费者处理位于不同位置的事件
	发布[0]
    ConsumerC id=0   price=0.0
    发布[1]
    发布[2]
    ConsumerC id=2   price=2.0
    ConsumerD id=1   price=1.0
    ConsumerC id=3   price=3.0
    发布[3]
```

**DSL**

1:所谓DSL我的理解就是消费者这里相互依赖。

{% asset_img 640-1576726329157.webp %}

```java
dw.consumeWith(handler1a, handler2a);
dw.after(handler1a).consumeWith(handler1b);
dw.after(handler2a).consumeWith(handler2b);
dw.after(handler1b, handler2b).consumeWith(handler3);
ProducerBarrier producerBarrier = dw.createProducerBarrier();
```