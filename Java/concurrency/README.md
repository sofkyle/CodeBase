# Java线程同步机制
### Java锁分类
**内部锁（Intrinsic Lock）**：通过synchronized关键字实现。
**显式锁（Explicit Lock）**：通过java.concurrent.locks.Lock接口的实现类实现，如：java.concurrent.locks.ReentrantLock。


### 可重入性
一个线程在其持有一个锁的时候能否再次（或者多次）申请**该**锁。
如果一个线程持有一个锁的时候还能够继续成功申请该锁，则称该锁是可重入的，否则称为非可重入的。
>**TIPS：**可重入锁可以被理解为一个对象，该对象包含一个计数器属性，表示相应的锁还没有被任何线程持有。每次线程获得一个可重入锁的时候，该锁的计数器值会被增加1。每次一个线程释放锁的时候，该锁的计数器属性值就会被减1。一个线程初次获得可重入锁时相应的开销相对大，因为必须与其他线程“竞争”以获得锁。


### 内部锁
#### 示例
``` java
public short nextSequence() {
	synchronized (锁句柄) {
		// 同步块
	}
}
```
锁句柄是一个对象的引用，可以填写为**this**关键字，即表示以当前对象为锁句柄。锁句柄对应的监视器就被称为相应同步块的**引导锁**。线程在执行临界区代码的时候必须持有该临界区的引导锁。

#### 内部锁的调度
Java虚拟机会为每个内部锁分配一个**入口集（Entry Set）**，用来记录等待获得相应内部锁的线程。只有一个申请者能够成为该锁的持有线程，其他申请失败者会进入**BLOCKED**状态并被存入相应锁的入口集中等待。
>**TIPS：**申请的锁被其持有线程释放的时候，该锁的入口集中的**任意一个线程**会被Java虚拟机唤醒。Java虚拟机对内部锁的调度仅支持**非公平调度**，被唤醒的等待线程即将要占用处理器运行时可能还有其他新的活跃线程与该线程抢占这个被释放锁，因此被唤醒的线程不一定就能成为该锁的持有线程。


### 显式锁：Lock接口
#### 示例
``` java
// 创建一个Lock接口实例
private final Lock lock;

// 申请锁
lock.lock();
try {
	// 同步块
} finally {
	// 必须释放锁
	lock.unlock();
}
```

#### 显式锁的调度
**ReentrantLock**既支持非公平锁也支持公平锁。

>**TIPS：**公平锁对线程的暂停和唤醒有了约束，故而增加了
上下文切换的代价。因此，公平锁适合于锁被持有的时间相对较长或者线程申请锁的平均间隔时间相对长的情形。显式锁默认使用的是非公平调度策略。


### 读写锁
Java中对读写锁的抽象使用的是：java.util.concurrent.locks.ReadWriteLock接口，其默认实现类是：java.util.concurrent.locks.ReentrantReadWriteLock。
ReentrantReadWriteLock所实现的读写锁是个可重入锁，并且支持**锁的降级**，即支持一个线程持有写锁的情况下可以继续获得相应的读锁。


### 内存屏障
内存屏障是对一类仅针对内存读、写操作指令的跨处理器架构的比较底层的抽象。内存屏障是被插入到两个指令之间进行使用的，其作用是禁止编译器、处理器重排序从而保障有序性。它在指令序列中就像一堵墙，使其两侧的指令无法“穿越”它。
><font color="red" face="黑体">疑问点：</font>
><font color="red">分类：
>可见性：加载屏障、存储屏障
>有序性：获取屏障、释放屏障
></font>


### 锁与重排序
与锁有关的重排序规则可以理解为指令相对于临界区的“许进不许出”。
#### 重排序规则
**假定只有一个临界区时：**
一、临界区内的操作不允许被重排序到临界区之外；
二、临界区内的操作之间允许被重排序；
三、临界区外的操作之间可以被重排序；
**扩展到多个临界区：**
四、锁申请与锁释放操作不能被重排序；
五、两个锁申请操作不能被重排序；
六、两个锁释放操作不能被重排序。
七、临界区外的操作可以被重排到临界区之内。


### 轻量级同步机制：volatile关键字
**volatile变量不会被编译器分配到寄存器进行存储，对volatile变量的读写操作都是内存访问操作。**
#### Volatile原子性
在原子性方面它仅能保障对所修饰变量的读写原子性，volatile关键字的使用不会引起上下文切换。
>**TIPS：**在Java语言中，对long型和double型以外的任何类型的写操作都是原子操作，通过使用volatile关键字可以保障long/double型变量写操作的原子性。
#### Volatile有序性
禁止了如下重排序：
* 写volatile变量操作与该操作之前的任何读、写操作不会被重排序；
* 读volatile变量操作与该操作之后的任何读、写操作不会被重排序。


### CAS与原子变量
**CAS(Compare and Swap)**是对一种处理器指令的称呼。
CAS只是保障了共享变量更新这个操作的原子性，它并不保障可见性。

#### 原子操作工具：原子变量类
原子变量类（Atomics）是基于CAS实现的能够保障对共享变量进行read-modify-write更新操作的原子性和可见性的一组工具类。
分组 | <center>类</center> 
- | :-
基础数据类型| AtomicInteger、AtomiceLong、AtomicBoolean
数组型| AtomicIntegerArray、AtomictLongArray、AtomicReferenceArray
字段更新器| AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater
引用型| AtomicReference、AtomicStampedReference、AtomicMarkableReference


#### ABA问题
对于共享变量V，当前线程看到它的值为A的那一刻，其他线程已经将其值更新为B，接着在当前线程执行CAS的时候该变量的值又被其他线程更新为A，那么此时我们是否认为V的值没有被其他线程更新过呢？或者说这种结果是否可以接受呢？
规避ABA问题的方案：为变量的更新引入版本号。


### 对象的发布
**对象发布**是指使对象能够被其作用域之外的线程访问。

#### 重识static与final
* 多线程环境中，**static**关键字能保障线程正确读取到相应字段的初始值，但无法保障正确读取到更新值。
* 由于重排序的作用，一个线程读取到一个对象的引用时，该对象可能尚未初始化完毕，在多线程环境中，**final**保证字段发布到其他线程的时候都是初始化完毕的。

### 安全发布与逸出
**安全发布**就是指对象以一种线程安全的方式被发布。
**逸出**是指一个对象的发布出现我们不期望的结果或者对象发布本身不是我们所期望的时候。