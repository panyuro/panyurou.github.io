---
title: java锁常见面试题
date: 2024-05-21 23:16:07
tags:
categories:
---

# 1. sychronized

Synchronized是Java中解决并发问题的一种最常用的方法，Synchronized的作用主要有三个：

- **确保互斥访问**。确保被同步的方法或代码块在同一时刻只能由一个线程执行，其他线程必须等待该线程执行完毕后才能进入。
- **保证可见性**。保证线程间对共享变量的修改是可见的。具体来说，当一个线程修改了共享变量的值后，`synchronized` 关键字确保其他线程能够立即看到这个修改。
- **防止指令重排序**。指令重排序是一种优化技术，编译器和处理器可能会重新排列指令的执行顺序，以提高性能。但是在多线程环境下，指令重排序可能会导致线程间的执行顺序和预期的不一致。`synchronized` 通过内存屏障（Memory Barrier）机制，确保在进入和退出同步块时，内存的读写操作不会被重排序。

## 1. 应用方式
synchronized关键字最主要有以下3种应用方式，下面分别介绍

1.修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁

2.修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁

3.修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。

## 2. 实现原理

基于对象锁（monitor）机制。编译用sychronized修饰的同步代码块，再用javap -v查看字节码文件：执行同步代码块后首先要先执行monitorenter指令，退出的时候monitorexit指令。使用Synchronized进行同步，其关键就是必须要对对象的监视器monitor进行获取，当线程获取monitor后才能继续往下执行，否则就只能等待。而这个获取的过程是互斥的，即同一时刻只有一个线程能够获取到monitor。每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一。

## 2. Lock和synchronized区别

1. Lock是一个接口，而synchronized是Java中的关键字,synchronized是内置的语言实现， lock是通过代码实现的.

2. synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；

3. 通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。

4. Lock可以提高多个线程进行读操作的效率。

5. Lock可以让等待锁的线程响应中断，线程可以中断去干别的事务，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够响应中断；

   **在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时（即有大量线程同时竞争），此时Lock的性能要远远优于synchronized。所以说，在具体使用时要根据适当情况选择。**

# 2. AQS

## 1 概念

AQS，全称为 **AbstractQueuedSynchronizer**，是 Java 中提供的一种实现同步器的框架。它是 `java.util.concurrent.locks` 包的一部分。其核心是维护一个同步状态 `state` 以及一个 FIFO 等待队列。它提供了多种方法来操纵同步状态，并管理进入等待队列的线程。

#### 同步状态

同步状态是一个 `int` 类型的变量，通常用于表示资源的可用性。例如，在一个独占锁中，`state` 为 0 表示锁是可用的，1 表示锁已被持有。

#### 等待队列

当一个线程尝试获取锁但失败时，它会被加入到 AQS 的等待队列中。这个等待队列是一个双向链表，所有尝试获取锁的线程都排成队列，等待被唤醒并再次尝试获取锁。

## 2. 使用

AQS 通常不会直接使用，而是通过子类实现具体的同步器，如 `ReentrantLock`、`Semaphore` 等。

# 3.  ReentrantLock

## 1. 公平锁的加锁实现：

公平锁保证线程按照请求锁的顺序来获取锁.

1.加锁时会调用lock()方法去获取锁，lock()会调用用acquire()(AQS中)，而这个acquire方法内部又去调用了tryAcquire的方法

1. tryAcquire方法通过getState获取当前同步状态

   - 发现state为0。
     1. **判断是否有比当前线程等待时间更长的线程**。如果有，当前线程将等待。
     2. 没有的话，使用原子操作尝试将锁的状态从 0 改为 1，表示获取锁成功，设置锁的拥有者为当前线程， 获取成功，

   - 不为0，检查当前线程是否已经持有锁，判断重入锁的逻辑。
     1. 是重入锁的话，增加重入次数， 也是获取成功，

2. 获取失败时，调用 `addWaiter` 将当前线程加入等待队列。尝试通过CAS把当前现在追加到队尾，修改为节点为最新的节点，如果修改失败，意味着有并发，这个时候进入enq中的死循环，进行“自旋”的方式修改

## 2. 公平锁的释放锁实现：

1. 头节点在释放同步状态的时候，会调用unlock（），而unlock会调用release()，release()会调用tryRelease方法尝试释放当前线程持有的锁,先判断当前线程是否为持有锁的线程，如果是，则执行减减操作，否则抛出异常（同步状态）

2. 成功的话调用unparkSuccessor() 唤醒后继线程，并返回true，否则直接返回false，

   注意：

   - 只有线程A把此锁全部释放了，状态值减到0了，其他线程才有机会获取锁，当A把锁完全释放后，state恢复为0
   - 队列中的节点在被唤醒之前都处于阻塞状态。当一个线程节点被唤醒然后取得了锁，对应节点会从队列中删除

## 3. 非公平锁的加锁实现：

- 非公平锁在尝试获取锁时，是先直接 CAS 设置 state 变量，如果设置成功，表明加锁成功，当前线程就成为了锁的持有者。
- 抢占失败,再调用 acquire 方法将线程置于队列尾部排队。

> 非公平锁的机制：如果新来了一个线程，试图访问一个同步资源，只需要确认当前没有其他线程持有这个同步状态，即可获取到。

这个区别很重要，因为线程在阻塞和非阻塞之间切换时需要比较长的时间，如果刚好线程A释放了资源，A会去唤醒下一个排着队的Node节点，当这个唤醒操作还没完成的时候，这时又来了一个线程B，线程B发现当前没人持有这个资源，于是自己就迅速拿到了这个资源，充分利用了线程A去唤醒B的这一段时间，这就是公平锁和非公平锁之间的差异，这里也体现了非公平锁性能较高的地方。

# 4. ReentrantReadWriteLock

ReentrantReadWriteLock是 Lock 的另一种实现方式，我们已经知道了 ReentrantLock 是一个排他锁，同一时间只允许一个线程访问，而 ReentrantReadWriteLock 允许多个读线程同时访问，但不允许写线程和读线程、写线程和写线程同时访问。相对于排他锁，提高了并发性。
## 

# 5. CountDownLatch & CyclicBarrier

## 1. CountDownLatch

CountDownLatch是一个计数器闭锁，通过它可以完成类似于阻塞当前线程的功能，即：一个线程或多个线程一直等待，直到其他线程执行的操作完成。

CountDownLatch用一个给定的计数器来初始化，该计数器的操作是原子操作，即同时只能有一个线程去操作该计数器。调用该类await方法的线程会一直处于阻塞状态，直到其他线程调用countDown方法使当前计数器的值变为零，每次调用countDown计数器的值减1。当计数器值减至零时，所有因调用await()方法而处于等待状态的线程就会继续往下执行。这种现象只会出现一次，因为计数器不能被重置，如果业务上需要一个可以重置计数次数的版本，可以考虑使用CycliBarrier。


## 2 CyclicBarrier

CyclicBarrier也是一个同步辅助类，它允许一组线程相互等待，直到到达某个公共屏障点（common barrier point）。

通过它可以完成多个线程之间相互等待，只有当每个线程都准备就绪后，才能各自继续往下执行后面的操作。类似于CountDownLatch，它也是通过计数器来实现的。当某个线程调用await方法时，该线程进入等待状态，且计数器加1，当计数器的值达到设置的初始值时，所有因调用await进入等待状态的线程被唤醒，继续执行后续操作。因为CycliBarrier在释放等待线程后可以重用，所以称为循环barrier。CycliBarrier支持一个可选的Runnable，在计数器的值到达设定值后（但在释放所有线程之前），该Runnable运行一次，注，Runnable在每个屏障点只运行一个。

## 3. 区别

CountDownLatch主要是实现了1个或N个线程需要等待其他线程完成某项操作之后才能继续往下执行操作，描述的是1个线程或N个线程等待其他线程的关系。CyclicBarrier主要是实现了多个线程之间相互等待，直到所有的线程都满足了条件之后各自才能继续执行后续的操作，描述的多个线程内部相互等待的关系。
CountDownLatch是一次性的，而CyclicBarrier则可以被重置而重复使用。



# 6 CAS

CAS（Compare and Swap）有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

java.util.concurrent(J.U.C)种提供的atomic包中的类，使用的是乐观锁，用到的机制就是CAS，当多个线程尝试使用CAS同时更新一个变量时，只有其中一个线程可能更新变量的值，而其他线程都失败，失败的线程不会被挂起，而是被告知这次竞争失败，并可以再次尝试。

## 1. AtomicInteger

以AtomicInteger为例，研究在没有锁的情况下是如何做到数据正确性的。
例如 AtomicInteger 中有一个原子方式 i++ 操作，即

1. 调用incrementAndGet(),而incrementAndGet() 调用 unsafe下的方法getAndAddInt()
2. getAndAddInt()中有一个 valueOffset 参数，这个值是 value 值在 AtomicInteger 类型中内存的偏移地址。传入的 valueOffset 参数会在后续方法中，直接从内存位置读取这个字段的值。
3. 得到最新值后，调用 compareAndSwapInt 来更新最新值，如果对象 o 中 offset 偏移位置的值等于期望值(expected)，就将该 offset 处的值更新为 x，当更新成功时，返回 true。结合前面调用来看，如果当前值是 v，就设置为 v+1。否则重试直到成功为止。
   

## 2. CAS问题

1. **ABA问题**。
   因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。
   ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。
   从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

2. **循环时间长开销大。**
   自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

3. **只能保证一个共享变量的原子操作。**
   当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。

   

# 7 ThreadLocal

ThreadLocal 提供线程内部的局部变量，在本线程内随时随地可取，隔离其他线程

**早期设计：**

每个 ThreadLocal类都创建一个 Map，然后用线程的 ID threadID 作为 Map 的 key，要存储的局部变量作为 Map 的 value，这样就能达到各个线程的值隔离的效果。

 **JDK8 ThreadLocal 的设计：**

每个 Thread 维护一个 ThreadLocalMap 哈希表，这个哈希表的 key 是 ThreadLocal 实例本身，value才是真正要存储的值 Object。

**这样设计有如下几点优势：**
1） 这样设计之后每个 Map 存储的 Entry 数量就会变小，因为之前的存储数量由Thread 的数量决定，现在是由 ThreadLocal 的数量决定。
2） 当 Thread 销毁之后，对应的 ThreadLocalMap 也会随之销毁，生命周期与线程相同，能减少内存的使用。
注：ThreadLocalMap其实是线程自身的一个成员属性threadLocals的类型。也就是线程本地数据都存在这个threadLocals应用的ThreadLocalMap中。

## 1. 底层实现

## 1. set(Tvalue)

1 ) 获取当前线程 Thread 对象，进而获取此线程对象中维护的 ThreadLocalMap 对象。
2 ) 判断当前的 ThreadLocalMap 是否存在：
如果存在，则调用 map.set 设置此实体 entry。
如果不存在，则调用 createMap 进行 ThreadLocalMap 对象的初始化，并将此实体 entry 作为第一个值存放至 ThreadLocalMap 中。

## 2. get()

1. 获取当前线程 Thread 对象，进而获取此线程对象中维护的 ThreadLocalMap 对象。

2. 判断当前的 ThreadLocalMap 是否存在：

   - 如果存在，则以当前的 ThreadLocal 为 key，调用 ThreadLocalMap 中的 getEntry 方法获取对应的存储实体 e。找到对应的存储实体 e，获取存储实体 e 对应的 value 值，即为我们想要的当前线程对应此 ThreadLocal 的值，返回结果值。

   - 如果不存在，则证明此线程没有维护的 ThreadLocalMap 对象，调用 setInitialValue 方法进行初始化。返回 setInitialValue 初始化的值。

3. setInitialValue 方法的操作如下：
   1. 调用 initialValue 获取初始化的值。
   2. 获取当前线程 Thread 对象，进而获取此线程对象中维护的 ThreadLocalMap 对象，并判断当前的 ThreadLocalMap 是否存在：
      - 如果存在，则调用 map.set 设置此实体 entry。
      - 如果不存在，则调用 createMap 进行 ThreadLocalMap 对象的初始化，并将此实体 entry 作为第一个值存放至 ThreadLocalMap 中。
        
