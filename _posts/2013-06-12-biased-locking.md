---
tags: [多线程, 性能, 技术]
layout: post
category: tech
title: Biased Locking
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

测试结论:

1. `+UseBiasedLocking` 对单线程访问下的锁, 性能提升是巨大的. 不同cpu架构提升倍率不同
2. 至少到现在为止, 只对内置的`synchronized`字段的锁有效. 对`ReentrantLock`无效
3. `一旦有另一个线程锁过这个锁之后, 就算完全没有发生锁竞争, 优化就会取消`
4. 取消的锁优化只针对这个锁对象, 而不是代码块. 新建个锁对象, 又可以被优化
5. 取消过优化的锁, 加锁的消耗比直接不开启UseBiasedLocking, 更大. 大5%左右

对比下synchronized字段和ReentrantLock的性能
----

单线程下, 使用ReentrantLock: 53,030,318 ops/sec. 比synchronized慢3倍. 和Martin Thompson的[测试(需翻墙)](http://mechanical-sympathy.blogspot.com/2011/11/biased-locking-osr-and-benchmarking-fun.html)结果不同. Martin测试的结果是synchronized字段的锁只有在2个线程竞争的情况下, 才比reentrantLock更快. 线程数越多, ReentrantLock的优势越大.

Martin的测试是在java 1.6下面跑的, 我拿了他的测试代码, 在java 1.7u21 台式机下重新跑了一遍


<table>
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
        <td>61,910,074</td>
    </tr>

    <tr>
        <td>2</td>
        <td>30,945,544</td>
        <td>22,919,502</td>
        <td>12,111,286</td>
    </tr>

    <tr>
        <td>3</td>
        <td>27,954,956</td>
        <td>26,095,822</td>
        <td>32,936,206</td>
    </tr>

    <tr>
        <td>4</td>
        <td>29,484,435</td>
        <td>28,944,632</td>
        <td>33,352,128</td>
    </tr>
</table>



贴代码
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

