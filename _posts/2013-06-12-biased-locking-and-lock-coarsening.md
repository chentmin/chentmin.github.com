---
tags: [多线程, 性能, 技术]
layout: post
category: tech
title: Biased Locking and Lock Coarsening
tagline: 
---

Biased Locking 介绍
----

锁, 如果只有单线程访问, 也会有很大的额外消耗. 内部实现是通过Atomic的CAS操作, 将当前持有锁的线程设为自己. CAS操作是个消耗很大的操作, 如果CAS成功, 则需要将其他cpu缓存中的这个volatile的field删除. jvm为了优化单线程访问下的锁, 避免每次加锁都需要CAS操作, 可将锁对象偏向(Bias)给这个线程, 以后每次只要是这个线程加这个锁, 都不再需要CAS操作. 一旦有第二个线程需要锁这个对象时, bais优化即刻取消, fall back到使用CAS获得锁.

测试单线程下Biased Locking带来的性能提升
----

测试流程: 使用不同的JVM启动参数运行

1. `-XX:-UseBiasedLocking` 关闭BiasedLocking. JAVA 6起默认是启用的
2. `-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0` 启用. jvm默认从启动4秒后才启用BiasedLocking优化, 因为不少程序刚启动时, 锁竞争很激烈, 为了避免频繁取消BiasedLocking优化, 所以设了4秒之后才启动. 使用`-XX:BiasedLockingStartupDelay=0`参数可关闭4秒的延时.

测试数据: 使用1.7u10, 在2012年中的Macbook Air下跑

1. 关闭BiasedLocking优化: 172,198,544 ops/sec
2. 开启BiasedLocking优化: 1,500,881,768 ops/sec 将近9倍的提升
3. 不管是否开启BiasedLocking, 使用ReentrantLock: 53,030,318 ops/sec. 比synchronized慢3倍

1.7u21, 在台式机下跑

1. 关闭BiasedLocking优化: 200,601,805 ops/sec
2. 开启BiasedLocking优化: 1,784,997,099 ops/sec
3. 不管是否开启BiasedLocking, 使用ReentrantLock: 63,636,002 ops/sec. 比synchronized慢3倍

测试结论:

1. `+UseBiasedLocking` 对单线程访问下的锁, 性能提升是巨大的. 不同cpu架构提升倍率不同
2. 至少到现在为止, 只对内置的`synchronized`字段的锁有效. 对`ReentrantLock`无效
3. `一旦有另一个线程锁过这个锁之后, 就算完全没有发生锁竞争, 优化就会取消`
4. 取消的锁优化只针对这个锁对象, 而不是代码块. 新建个锁对象, 又可以被优化
5. 取消过优化的锁, 加锁的消耗比直接不开启UseBiasedLocking, 更大. 大5%左右

对比下synchronized字段和ReentrantLock的性能
----

Martin Thompson的[测试(需翻墙)](http://mechanical-sympathy.blogspot.com/2011/11/biased-locking-osr-and-benchmarking-fun.html)结果结论: synchronized字段的锁只有在2个线程竞争的情况下, 才比reentrantLock更快. 线程数越多, ReentrantLock的优势越大.

Martin的测试是在java 1.6下面跑的, 我拿了他的测试代码, 在java 1.7u21 台式机下重新跑了一遍


<table  border="1">
    <tr>
        <td>Threads</td>
        <td>-UseBiasedLocking</td>
        <td>+UseBiasedLocking</td>
        <td>ReentrantLock</td>
    </tr>

    <tr>
        <td>1</td>
        <td>51,945,186</td>
        <td>567,973,991</td>
        <td>62,413,339</td>
    </tr>

    <tr>
        <td>2</td>
        <td>29,288,753</td>
        <td>23,477,670</td>
        <td>11,320,053</td>
    </tr>

    <tr>
        <td>3</td>
        <td>28,461,054</td>
        <td>27,059,716</td>
        <td>32,936,206</td>
    </tr>

    <tr>
        <td>4</td>
        <td>29,016,052</td>
        <td>28,865,443</td>
        <td>33,917,586</td>
    </tr>


    <tr>
        <td>5</td>
        <td>29,042,465</td>
        <td>27,892,191</td>
        <td>33,959,625</td>
    </tr>


    <tr>
        <td>6</td>
        <td>28,936,762</td>
        <td>27,720,316</td>
        <td>35,117,010</td>
    </tr>

</table>

意外的发现
-----

使用Martin的代码跑, -UseBiasedLocking 单线程, 数据是 51,945,186 ops/sec. 但是用我自己的代码跑, 同样的参数下 200,601,805 ops/sec. 看来看去代码都是相同的, 唯一的解释就是jvm给我的代码做了更多的优化. 最后发现, 带上 `-XX:-EliminateLocks` 参数后跑我的代码, 性能终于**只有**51,793,821 ops/sec 了. 

Lock Coarsening
-----

把多次加锁合并为一次加锁. 比如以下代码

    synchronized{
        doStuff();
    }

    synchronized{
        doAnotherStuff();
    }

jvm可以合并为

    synchronized{
        doStuff();
        doAnotherStuff();
    }

甚至在2个同步块之间有其他没同步的代码, jvm也可以把那些代码移到同步块中. jvm只是不可以把同步块中的代码移出去.

但是这么做可能会导致一次操作拥有太长时间的锁, 所以jvm在长循环中不会使用这个优化

只要把Martin的代码的count从long改为int, 代码性能马上也提升到 207,231,294 ops/sec 了

    private void jvmLockInc(){
        // long count = iterationLimit / numThreads;
        int count = (int) (iterationLimit / numThreads);
        while (0 != count--){
            synchronized (jvmLock){
                ++counter;
            }
        }
    }

Martin的测试, with Lock Coarsening
----

重新跑一遍Martin的测试


<table  border="1">
    <tr>
        <td>Threads</td>
        <td>-UseBiasedLocking</td>
        <td>+UseBiasedLocking</td>
        <td>ReentrantLock</td>
    </tr>

    <tr>
        <td>1</td>
        <td>207,268,577</td>
        <td>1,673,483,322</td>
        <td>62,634,436</td>
    </tr>

    <tr>
        <td>2</td>
        <td>91,527,148</td>
        <td>86,358,425</td>
        <td>61,109,372</td>
    </tr>

    <tr>
        <td>3</td>
        <td>113,451,592</td>
        <td>107,976,924</td>
        <td>54,830,405</td>
    </tr>

    <tr>
        <td>4</td>
        <td>120,471,186</td>
        <td>112,403,653</td>
        <td>42,624,056</td>
    </tr>


    <tr>
        <td>5</td>
        <td>114,966,499</td>
        <td>106,730,465</td>
        <td>41,076,857</td>
    </tr>


    <tr>
        <td>6</td>
        <td>118,888,054</td>
        <td>116,308,038</td>
        <td>38,167,327</td>
    </tr>

</table>

Lock Coarsening优化大幅提升了synchronized的性能, 但同样不会对ReentrantLock优化. 依然, 不开启BiasedLocking优化, 比开启了但是取消了优化的锁, 性能高5%左右. 不过成功应用了BiasedLocking优化, 那性能提升可是8,9倍. 

贴我的测试代码
---

    public class LockObject {

        public static int counter;
        
        public synchronized void operationWithLock(){
            ++counter;
        }
        
    }


    public class BenchBiasedLocking {
        private static volatile LockObject lobj = new LockObject();

        private static long benchSingleThread(int reps) {
            LockObject lo = lobj;

            long startTime = System.nanoTime();
            for (int i = reps; --i >= 0;) {
                lo.operationWithLock();
            }
            long duration = System.nanoTime() - startTime;
            
            return (((long) reps) * TimeUnit.SECONDS.toNanos(1)) / duration;
        }

        public static void main(String[] args) throws InterruptedException {
            for (int i = 0; i < 10; i++) {
                benchSingleThread(80000000);
            };
            
            System.out.println("Single Thread");
            System.out.println(benchSingleThread(80000000));
            
            Thread t = new Thread() {
                public void run() {
                    System.out.println("Another thread calling");
                    System.out.println(benchSingleThread(1000000));
                }
            };
            t.start();
            t.join();
            System.out.flush();
            System.out.println("Old thread after another thread called");
            System.out.println(benchSingleThread(180000000));

            lobj = new LockObject();
            System.out.println("After new object");
            System.out.println(benchSingleThread(180000000));

            System.out.println(LockObject.counter); // prevent JIT
        }
    }

