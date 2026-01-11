---
description: 本文系统梳理了 Java 并发编程的基石——CAS（比较并交换）技术。内容涵盖 java.util.concurrent.atomic 原子类应用、Unsafe 类与 CPU 汇编指令（cmpxchg）的底层实现机制、自旋锁的设计原理，以及 CAS 面临的循环开销与 ABA 问题，并结合代码示例演示了如何使用 AtomicStampedReference 解决数据一致性问题。
title: 从汇编指令到 Java 应用：彻底搞懂 CAS 机制与原子操作
tag:
  - 多线程
sidebar: true
comment: true
recommend: 2
---
## 一、原子类
java.util.concurrent.atomic下的类

**没有使用CAS之前：**

多线程环境<font style="color:#E8323C;">**不使用原子类**</font>保证线程安全i++(基本数据类型)

使用CAS之后:

多线程环境：使用<font style="color:#E8323C;">**原子类**</font>保证线程安全i++(基本数据类型)

### 什么是CAS
compare and swap 的缩写，中文翻译成<font style="color:#E8323C;">比较并交换</font>，<font style="color:#000000;">实现并发算法时常用到的一种技术。</font>

**<font style="color:#000000;">它包含三个操作数----内存位置、预期原值、及更新值</font>**

**<font style="color:#000000;">执行CAS操作的时候，将内存位置的值与预期原值比较：</font>**

**<font style="color:#000000;">如果</font>**<font style="color:#F5222D;">相匹配</font>**<font style="color:#000000;">，那么处理器会自动将该位置值更新为新值，</font>**

**<font style="color:#000000;">如果</font>**<font style="color:#F5222D;">不匹配</font>**<font style="color:#000000;">，那么处理器不做任何操作，多个线程同时执行CAS操作只有一个会成功。</font>**

CAS有3个操作数，位置内存值V，旧的预期值A，要修改的更新值B。

当且仅当旧的预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做或重来<font style="color:#F5222D;">当它重来重试的这种行为成为----自旋!!</font>
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660901687841-e0b00b9d-5dd9-4dd1-9f75-46e0e02c37bd.png)

### 硬件级别的保证
CAS是JDK提供的<font style="color:#2F54EB;">**非阻塞**</font>原子性操作，它通过**<font style="color:#2F54EB;">硬件保证</font>**了比较-更新的原子性。

它是非阻塞的且自身具有原子性，也就是说这玩意效率更高且通过硬件保证，说明这玩意更可靠。

CAS是一条CPU的原子指令(**<font style="color:#F5222D;">cmpxchg指令</font>**），不会造成所谓的数据不一致问题，**Unsafe**提供的CAS方法（如compareAndSwapXXX）底层实现即为CPU指令cmpxchg。

执行cmpxchg指令的时候，会判断当前系统是否为多核系统，如果是就给总线加锁，只有一个线程会对总线加锁成功，加锁成功之后会执行cas操作，**<font style="color:#F5222D;">也就是说CAS的原子性实际上是CPU实现独占的</font>**，比起用synchronized重量级锁，这里的排他时间要短很多，所以在多线程情况下性能会比较好。
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660902569263-6e3f88da-2f51-4b4a-b458-272dc71bf30e.png)

## 二、CAS底层原理及对unsafe类的理解
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660902993002-0a8a8602-96f8-4966-80f5-7bf6186df8a5.png)
### 1、Unsafe
是CAS的核心类，由于Java方法无法直接访问底层系统，需要通过本地（native）方法来访问，Unsafe相当于一个后门，基于该类可以直接操作特定内存的数据。**<font style="color:#F5222D;">Unsafe类存在于sun.misc包中</font>**，其内部方法操作可以像C的<font style="color:#F5222D;">**指针**</font>一样直接操作内存，因为Java中CAS操作的执行依赖于Unsafe类的方法。

**<font style="color:#F5222D;">注意Unsafe类中的所有方法都是native修饰的，也就是说Unsafe类中的方法都直接调用操作系统底层资源执行相应任务</font>**

### <font style="color:#000000;">2、变量valueOffset</font>
表示该变量值在内存中的<font style="color:#FA541C;">**偏移地址**</font>，因为Unsafe就是根据内存偏移地址获取数据的值。
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660904797027-33b09c0c-a936-4c62-a3ba-198bc6f7f5b9.png)

我们知道i++线程不安全的，那么atomicInteger.getAndIncrement()能保证原子性

CAS的全称为Compare-And-Swap，**<font style="color:#FA541C;">它是一条CPU并发原语。</font>**

它的功能是判断内存某个位置的值是否为预期值，如果是则更改为新的值，这个过程是原子的。

Atomiclnteger类主要利用<font style="color:#FA541C;">**CAS(compare and swap)+ volatile和 native方法来保证原子操作**</font>，从而避免 synchronized的高开销，执行效率大为提升。
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660905138894-7930868a-c9be-46a1-bb2a-4e4c53e1661e.png)

CAS并发原语体现在JAVA语言中就是sun.misc.Unsafe类中的各个方法。调用UnSafe类中的CAS方法，JVM会帮我们实现出<font style="color:#2F54EB;">**CAS汇编指令**</font>。这是一种完全依赖于**<font style="color:#2F54EB;">硬件</font>**的功能，通过它实现了原子操作。再次强调，由于CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，<font style="color:#F5222D;">**并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成所谓的数据不一致问题。**</font>
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660907941085-fd429275-cc7d-4722-87d2-53a16bd30b4d.png)

假设线程A和线程B两个线程同时执行getAddInt操作(分别跑在不同CPU上)；

1 AtomicInteger里面的Value原始值为3，即主内存中的AtomicInteger的value为3，根据JMM模型，线程A和线程B各自持有一份值为3的value的副本分别到各自的工作内存

2 线程A通过getIntVolatile（var1，var2）拿到value值3，这时线程A被挂起。

3.线程B也通过getIntVolatile(var1,var2）方法获取value值3，此时刚好线程B<font style="color:#F5222D;">**没有被挂起**</font>并执行compareAndSwapInt方法比较内存值为3，成功修改内存值为4，线程B打完收工，一切OK。

4.这时线程A恢复，执行compareAndSwapInt方法修饰，发现自己手里的值数字3和主内存的值数字4不一致，说明该值已经被其他线程抢先一步修改过了，那A线程本次修改失败，**<font style="color:#F5222D;">只能重新读取重新来一遍了</font>**。

5.线程A重新获取Value值，因为变量value被volatile修饰，所以其他线程对它的修改，线程A总是能够看到，线程A继续执行CompareAndSwapInt进行比较替换，直到成功。

### 3.底层汇编
#### native修饰的方法代表的是底层方法
Unsafe类中的compareAndSwapInt，是一个本地方法，该方法的实现位于unsafe.cpp中
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660908691959-633e7d7b-3a9d-490d-ad9d-82d2375a067e.png)

JDK提供的CAS机制，在汇编层级会禁止变量两侧的指令优化，然后使用cmpxchg指令比较并更新变量值（原子性）

**<font style="color:#F5222D;">（Atomic::cmpxchg(x,addr,e)）== e;</font>**
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660908860821-08ce21f6-9a5b-4087-b0e3-8cafab889530.png)
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660909067925-7aa642a8-9e1f-4add-8294-59fe9c8e5863.png)

你只需要记住：CAS是靠硬件实现的从而在硬件层面提升效率，最底层还是交给硬件来保证原子性和可靠性

实现方式是基于硬件平台的汇编指令，在intel的CPU中（X86机器上），使用的是汇编指令cmpxchg指令。

核心思想就<font style="color:#F5222D;">**是，比较要更新变量的值V和预期值E（compare），相等才会将V的值设为N**</font>

**<font style="color:#F5222D;">如果不相等就自旋。</font>**

## <font style="color:#000000;">三、原子引用</font>
### 1.AtomicInteger原子整形，可否有其他原子类型
1. AtomicBook
2. AtomicOrder

```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
class User
{
    String userName;
    int age;
}

public class AtomicReferenceDemo {
    public static void main(String[] args) {
        AtomicReference<User> atomicReference = new AtomicReference<>();
        User z3 = new User("zs",22);
        User li4 = new User("li4", 28);
        atomicReference.set(z3);
        System.out.println(atomicReference.compareAndSet(z3,li4)+"\t"+atomicReference.get().userName);
        System.out.println(atomicReference.compareAndSet(z3,li4)+"\t"+atomicReference.get().userName);
    }
}
```

![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660909712514-f912ec09-f2c1-4b77-be8e-cca5b81eacbe.png)

### 2.自旋锁
（spinlock）

CAS是实现自旋锁的基础，CAS利用CPU指令保证了操作的原子性，以达到锁的效果，至于自旋呢，看字面意思也很明白，自己旋转，是指尝试获取锁的线程不会立即阻塞，而是<font style="color:#F5222D;">**采用循环的方式去尝试获取锁**</font>，当线程发现锁被占用时，会不断循环判断锁的状态，直到获取。这样的好处是减少线程上下文切换的消耗，缺点是循环会消耗CPU
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660915775953-1807e1c4-2678-4e50-912d-3a03b8055603.png)

```java
public class SpinLockDemo {

    AtomicReference<Thread> atomicReference = new AtomicReference<>();

    public void lock(){
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName()+"come in");
        while (!atomicReference.compareAndSet(null, thread)) {

        }
    }

    public void unlock(){
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread, null);
        System.out.println(Thread.currentThread().getName()+"task over,unlock..");
    }

    public static void main(String[] args) throws InterruptedException {
        SpinLockDemo spinLockDemo = new SpinLockDemo();
        new Thread(()->{
            spinLockDemo.lock();
            //暂停几秒钟线程
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            spinLockDemo.unlock();
        },"A").start();

        //暂停400毫秒
        TimeUnit.MILLISECONDS.sleep(400);

        new Thread(()->{
            spinLockDemo.lock();

            spinLockDemo.unlock();
        },"B").start();
    }
}
```

## 四、CAS缺点
+ **<font style="color:#F5222D;">循环时间长开销很大。</font>**
  ![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660916839376-5049ed42-644e-4006-ad48-f879bf052693.png)

+ 引出来ABA问题

**<font style="color:#000000;">CAS会导致“ABA问题”。</font>**

CAS算法实现一个重要前提需要取出内存中某时刻的数据并在当下时刻比较并替换，那么在这个时间差类会导致数据的变化。比如说一个线程1从内存位置V中取出A，这时候另一个线程2也从内存中取出A，并且线程2进行了一些操作将值变成了B，然后线程2又将V位置的数据变成A，这时候线程1进行CAS操作发现内存中仍然是A，预期OK，然后线程1操作成功。

**<font style="color:#F5222D;">尽管线程1的CAS操作成功，但是不代表这个过程就是没有问题的。</font>**

### <font style="color:#F5222D;">1.版本号 （戳记号）</font>
**AtomicStampedReference**

```java
@Getter
@Setter
@AllArgsConstructor
class Book{
    private int id;
    private String bookName;
}

/**
 * @author 86183
 */
public class AtomicStampedDemo {

    public static void main(String[] args) {
        Book javabook = new Book(1, "javabook");
        AtomicStampedReference<Book> stampedReference = new AtomicStampedReference<>(javabook,1);
        System.out.println(stampedReference.getReference().getBookName()+"\t"+stampedReference.getStamp());

        Book mysqlBook = new Book(2, "mysqlBook");
        boolean b = stampedReference.compareAndSet(javabook, mysqlBook, stampedReference.getStamp(), stampedReference.getStamp() + 1);
        System.out.println(b+"\t"+stampedReference.getReference().getBookName()+"\t"+stampedReference.getStamp());

        boolean b1 = stampedReference.compareAndSet(mysqlBook, javabook, stampedReference.getStamp(), stampedReference.getStamp() + 1);
        System.out.println(b1+"\t"+stampedReference.getReference().getBookName()+"\t"+stampedReference.getStamp());
    }

}
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1660919432566-6bc5690d-8774-4a75-b0c6-2925fc89e849.png)
```

### 2.多线程环境下产生ABA问题
```java
public class ABADemo {

    static AtomicInteger atomicInteger = new AtomicInteger(100);
    static AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(100,1);
    public static void main(String[] args) {
        new Thread(()->{
            int stamp = stampedReference.getStamp();
            System.out.println(Thread.currentThread().getName()+"\t"+"首次版本号"+stamp);
            //暂停500毫秒，保证后面的t4线程初始化拿到的版本号和我一样
            try {
                TimeUnit.MILLISECONDS.sleep(500);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            stampedReference.compareAndSet(100,101,stampedReference.getStamp(),stampedReference.getStamp()+1);
            System.out.println(Thread.currentThread().getName()+"\t"+"2次流水号"+stampedReference.getStamp());
            stampedReference.compareAndSet(101,100,stampedReference.getStamp(),stampedReference.getStamp()+1);
            System.out.println(Thread.currentThread().getName()+"\t"+"3次流水号"+stampedReference.getStamp());
            },"t3").start();

        new Thread(()->{
            int stamp = stampedReference.getStamp();
            System.out.println(Thread.currentThread().getName()+"\t"+"首次版本号"+stamp);
            //暂停500毫秒，保证后面的t4线程初始化拿到的版本号和我一样
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            boolean b = stampedReference.compareAndSet(100, 2022, stamp, stamp + 1);
            System.out.println(b);
            System.out.println(stampedReference.getReference()+stampedReference.getStamp());
        },"t4").start();
    }

    public static void happen(){
        new Thread(()->{
            atomicInteger.compareAndSet(100,101);
            try {
                TimeUnit.MILLISECONDS.sleep(10);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            atomicInteger.compareAndSet(101,100);
        },"t1").start();

        new Thread(()->{
            try {
                TimeUnit.MILLISECONDS.sleep(200);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            };
            System.out.println(atomicInteger.compareAndSet(100,2022)+"\t"+atomicInteger.get());
        },"t2").start();
    }
}
```

