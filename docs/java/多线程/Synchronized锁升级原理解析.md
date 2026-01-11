---
description: 本文详细解析了 Java 中 synchronized 关键字的锁升级过程，涵盖无锁、偏向锁、轻量级锁和重量级锁的实现机制。通过对象头 Mark Word 的变化、CAS 操作、自旋锁与重量级锁的对比，结合实际代码与内存布局图示，帮助读者深入理解 synchronized 在高并发场景下的性能优化与适用场景，并解答锁升级过程中哈希码去向、JIT 编译器优化等常见面试问题。
title: 深入剖析 synchronized 锁升级机制：从偏向锁到重量级锁的演进原理
tag:
  - 多线程
sidebar: true
comment: true
recommend: 3
---
> 无锁-》偏向锁-》轻量锁-》重量锁
>

面试题：

1.谈谈你对synchronized的理解

2.synchronized的锁升级你聊聊![image-20260111213952905](.\image\image-20260111213952905.png)

**synchronized锁：**<font style="color:#E8323C;">**由对象头中的mark word根据锁标志位的不同而被复用及锁升级**</font>

![image-20260111215220222](.\image\image-20260111215220222.png)

## 一、synchronized 性能变化
### 1.Java5 重量级
1. 重量级锁，假如锁的竞争比较激烈的话，性能下降
2. **<font style="color:#E8323C;">Java5之前，用户态和内核态的切换</font>**
    Java的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就**<font style="color:#2F54EB;">需要操作系统介入</font>**，需要在户态与核心态之间切换，这种切换会消耗大量的系统资源，因为用户态与内核态都有各自专用的内存空间，专用的寄存器等，用户态切换至内核态需要传递给许多变量、参数给内核，内核也需要保护好用户态在切换时的一些寄存器值、变量等，以便内核态调用结束后切换回用户态继续工作。

在Java早期版本中，**<font style="color:#F5222D;">synchronized属于重量级锁，效率低下，因为监视器锁〈monitor)是依赖于底层的操作系统的MutexLock(系统互斥最)来实现的</font>**，**挂起线程和恢复线程都需要转入内核态去完成，阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态切换需要耗费处理器时间**，如果同步代码块中内容过于简单，这种切换的时间可能比用户代码执行的时间还长”，时间成本相对较高，这也是为什么早期的

synchronized效率低的原因Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，**<font style="color:#F5222D;">引入了轻量级锁和偏向锁</font>**

**<font style="color:#F5222D;">markOop.hpp</font>**
![image-20260111215342928](.\image\image-20260111215342928.png)

Monitor可以理解为一种同步工具，也可理解为一种同步机制，常常被描述为一个Java对象。**<font style="color:#F5222D;">Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁。</font>**
**<font style="color:#1890FF;">Monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从内核态到用户态的切换，成本非常高。</font>**

**<font style="color:#1890FF;">Mutex Lock</font>**

**<font style="color:#1890FF;">Monitor是在jvm底层实现的，底层代码是c++。本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的转换，状态转换需要耗费很多的处理器时间成本非常高。</font>**<font style="color:#F5222D;">**所以synchronized是Java语言中的一个重量级操作。**</font>

**<font style="color:#F5222D;">Monitor与Java对象以及线程是如何关联？</font>**

1.如果一个java对象被某个线程锁住，则**该java对象的Mark Word字段中LockWord指向monitor的起始地址**2.**Monitor剩Owner字段会存放拥有相关联对象锁的线程id**

**Mutex Lock的切换需要从用户态转换到核心态中，因此状态转换需要耗费很多的处理器时间**
![image-20260111215406785](.\image\image-20260111215406785.png)![image-20260111215417248](.\image\image-20260111215417248.png)2.Java6开始，优化Synchronized

> 为了减少获得锁和释放锁所带来的的性能消耗，引入了轻量级锁和偏向锁
>

## 二、 synchronized锁种类及升级步骤
### 1.多线程访问情况，3种
1. 只有一个线程来访问，有且唯一only one
2. 有多个线程（2个线程A、B来交替访问）
3. 竞争激烈，更多个线程来访问

### 2.升级流程
synchronized用的锁是存在**Java对象头里的Mark Word中锁升级功能主要依赖MarkWord中锁标志位和释放偏向锁标志位**
![](.\image\image-20260111215427966.png)

1. 偏向锁 ： **<font style="color:#1890FF;">MarkWord存储的是偏向的线程ID;</font>**
2. 轻量锁：  **<font style="color:#1890FF;">MarkWord存储的是指向线程栈中Lock Record的指针;</font>**
3. 重量锁：  **<font style="color:#1890FF;">MarkWord存储的是指向堆中的monitor对象的指针;</font>**

### <font style="color:#000000;">3.无锁状态</font>
![](.\image\image-20260111215439716.png)
### 4.  偏向锁
偏向锁：单线程竞争

当线程A第一次竞争到锁时，通过操作修改Mark Word中的偏向线程ID、偏向模式。

如果不存在其他线程竞争，那么持有偏向锁的线程**<font style="color:#F5222D;">将永远不需要进行同步</font>**

#### <font style="color:#000000;">主要作用</font>
**<font style="color:#F5222D;">当一段同步代码一直被同一个线程多次访问，由于只有一个线程那么该线程在后续访问时便会自动获得锁。</font>**

**避免多次从用户态切换会内核态**

```java
class Ticket{
    private int number=50;
    Object lockObject = new Object();
    public void sale(){
        synchronized (lockObject){
            if(number>0){
                System.out.println(Thread.currentThread().getName()+"卖出第"+number--+"张"+"\t"+"还剩"+number);
            }
        }
    }
}
public class saleTicketDemo {

    public static void main(String[] args) {
        Ticket ticket = new Ticket();
        new Thread(()->{
            for (int i = 0; i < 55; i++) {
                 ticket.sale();
            }
        },"a").start();
        new Thread(()->{
            for (int i = 0; i < 55; i++) {
                ticket.sale();
            }
        },"b").start();
        new Thread(()->{
            for (int i = 0; i < 55; i++) {
                ticket.sale();
            }
        },"c").start();
    }
}
```

**<font style="color:#F5222D;">结论：</font>**

Hotspot的作者经过研究发现，大多数情况下:

多线程的情况下，锁不仅不存在多线程竞争，还存在**<font style="color:#F5222D;">锁由同一个线程多次获得的情况</font>**，

偏向锁就是在这种情况下出现的，它的出现是为了解决**<font style="color:#F5222D;">只有在一个线程执行同步时提高性能。</font>**

**备注:**

+ 偏向锁会偏向于第一个访问锁的线程，如果在接下来的运行过程中，该锁没有被其他的线程访问，则持有偏向锁的线程将永远不需要触发同步。
+ 也即偏向锁在资源没有竞争情况下消除了同步语句，懒的连CAS操作都不做了，直接提高程序性能

#### <font style="color:#F5222D;">偏向锁的持有</font>
<font style="color:#000000;">说明：</font>

在实际应用运行过程中发现，“锁总是同一个线程持有，很少发生竞争”，也就是说<font style="color:#F5222D;">**锁总是被第一个占用他的线程拥有，这个线程就是锁的偏向线程**</font>。

那么只需要在锁第一次被拥有的时候，记录下偏向线程ID。这样偏向线程就一直持有着锁(后续这个线程进入和退出这段加了同步锁的代码块时，**<font style="color:#F5222D;">不需要再次加锁和释放锁</font>**。而是直接会去检查锁的MarkWord里面是不是放的自己的线程ID)。

**<font style="color:#F5222D;">如果相等</font>**，表示偏向锁是偏向于当前线程的，就不需要再尝试获得锁了，直到竞争发生才释放锁。以后每次同步，检查锁的偏向线程ID与当前线程ID是否一致，如果一致直接进入同步。无需每次加锁解锁都去CAS更新对象头。如果自始至终使用锁的线程只有一个，很明显偏向锁几乎没有额外开销，性能极高。

**<font style="color:#F5222D;">如果不等</font>**，表示发生了竞争，锁已经不是总是偏向于同一个线程了，这个时候会尝试使用CAS来替换MarkWord里面的线程ID为新线程的ID，（**<font style="color:#E8323C;">特殊情况：只有两个线程</font>**）

**<font style="color:#F5222D;">竞争成功</font>**，表示之前的线程不存在了，MarkWord里面的线程ID为新线程的ID，锁不会升

级，仍然为偏向锁;

**<font style="color:#F5222D;">竞争失败</font>**，这时候可能需要升级变为轻量级锁，才能保证线程间公平竞争锁。

**<font style="color:#2F54EB;">注意，偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程是不会主动释放偏向锁的。技术实现:</font>**

一个synchronized方法被一个线程抢到了锁时，那这个方法所在的对象就会在其所在的Mark Word中将偏向锁修改状态位，同时还会有占用54位来存储线程指针作为标识，若该线程再次访问同一个synchronized方法时，该线程只需去对象头的Mark Word中去判断一下是否有偏向锁指向本身的ID，无需在进入Monitor去竞争对象了

**细化锁案例：**

**偏向锁的操作不用直接捅到操作系统，不涉及****<font style="color:#2F54EB;">用户到内核转换，不必要直接升级为最高级</font>**，我们以一个account对象的"对象头"为例子。**
![image-20260111215553593](.\image\image-20260111215553593.png)

假如有一个线程执行到synchronized代码块的时候，JVM使用CAS操作把线程指针HID记录到Mark Word当中、并修改标偏向标示，标示当前线程就获得该锁。锁对象变成偏向锁（通过CAS修改对象头里的锁标志位〉，字面意思是“偏向于第一个获得它的线程”的锁。执行完同步代码块后，线程并不会主动释放偏向锁。

![image-20260111215607891](.\image\image-20260111215607891.png)

这时线程获得了锁，可以执行同步代码块。当该线程第二次到达同步代码块时会判断此时持有锁的线程是否还是自己（**持有锁的线程ID也在对象头里**)，JVM通过account对象的Mark Word判断:当前线程ID还在，说明还持有着这个对象的锁，就可以继续进入临界区工作。**<font style="color:#F5222D;">由于之前没有释放锁，这里也就不需要重新加锁。如果自始至终使用锁的线程只有一个，很明显偏向锁几乎没有额外开销，性能极高</font>**。结论：**JVM不用和操作系统协商设置Mutex(争取内核)，它只需要记录下线程ID就标示自己获得了当前锁，不用操作系统接入**。

上述就是偏向锁 : 在没有其他线程竞争的时候，一直偏向偏心当前线程，当前线程可以一直执行。

#### JVM 命令
![image-20260111215628601](.\image\image-20260111215628601.png)
**<font style="color:#E8323C;">实际上偏向锁在JDK1.6之后默认是开启的，但是启动时间有延迟，所以需要添加参数-XX：BiaseLockingStartupDelay=0，让其在程序启动时立刻启动</font>**

**<font style="color:#E8323C;">开启偏向锁</font>**

-XX：+UseBiaseLoking   -XX：BiaseLockingStartupDelay = 0

**<font style="color:#E8323C;">关闭偏向锁： 关闭之后程序默认会直接进入 --------------》》》》 轻量级锁状态。</font>**

-XX：-UseBiaseLocking
![image-20260111215644262](.\image\image-20260111215644262.png)code实现偏向锁

1. 加JVM命令 延迟 0S
2. 通过sleep

```java
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        Object o = new Object();
        synchronized (o){
            //hashcode 调用 才有
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
    }
```

![image (1)](.\image\image (1).png)

偏向锁带有线程id的情况后面，54为就不是0了

```java
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        Object o = new Object();
        //hashcode 调用 才有
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
        System.out.println("===================");
        new Thread(()->{
            synchronized (o){
                System.out.println(ClassLayout.parseInstance(o).toPrintable());
            }
        },"t1").start();
    }
![](https://cdn.nlark.com/yuque/0/2022/png/791535/1661928136233-39195072-70c9-4756-848d-57d12f399add.png)
```

![image (2)](C:\Users\liyl7\Downloads\image (2).png)

#### 偏向锁的撤销

+ 当有另外线程来逐步竞争锁的时候，就不能使用偏向锁了，要升级为轻量级锁
+ 竞争线程尝试CAS更新对象头失败，会等待到**<font style="color:#E8323C;">全局安全点</font>**（**<font style="color:#E8323C;">此时不会执行任何代码</font>**）撤销偏向锁

偏向锁使用一种等到**<font style="color:#E8323C;">竞争出现才释放锁的机制</font>**，只有当其他线程竞争锁时，持有偏向锁的原来线程才会被撤销。**<font style="color:#E8323C;">撤销需要等待全局安全点(该时间点上没有字节码正在执行)</font>**，同时检查持有偏向锁的线程是否还在执行:

第一个线程正在执行synchronized方法(**<font style="color:#E8323C;">处于同步块</font>**)，它还没有执行完，其它线程来抢夺，该偏向锁会被取消掉并出现锁升级。此时轻量级锁由原持有偏向锁的线程持有，继续执行其同步代码，而正在竞争的线程会进入自旋等待获得该轻量级锁。

第一个线程执行完成synchronized方法(**<font style="color:#E8323C;">退出同步块</font>**)，则将对象头设置成无锁状态并撤销偏向锁，重新偏向。第一个线程执行完成同步方法(退出同步块)，则将对象头设置成无锁状态并撤销偏向锁，重新偏向.
废除

**<font style="color:#E8323C;">JDK 15 以后 废除了偏向锁</font>**

### <font style="color:#000000;">5. 轻量级锁</font>
> 多线程竞争，但是任意时刻最多只有一个线程竞争，即不存在锁竞争太过激烈的情况，也就没有线程阻塞。
>

**<font style="color:#F5222D;">本质就是自旋锁CAS</font>**

**<font style="color:#000000;">64位标记图</font>**

![image-20260111215950677](.\image\image-20260111215950677.png)

轻量级锁是为了在线程**<font style="color:#F5222D;">近乎交替</font>**执行同步块时提高性能。

主要目的: 在没有多线程竞争的前提下，**<font style="color:#F5222D;">通过CAS</font>**减少重量级锁使用操作系统互璧量产生 的性能消耗，**<font style="color:#F5222D;">说白了先自旋，不行才升级为阻塞</font>**

升级时机: 当关闭偏向锁功能或多线程竞争偏向锁会导致偏向锁升级为轻量级锁

假如线程A已经拿到锁，这时线程B又来抢该对象的锁，由于该对象的锁已经被线程A拿到，当前该锁已是偏向锁了。

而线程B在争抢时发现对象头Mark Word中的线程ID不是线程B自己的线程ID(而是线程A)，那线程B就会进行CAS操作希望能获得锁。**<font style="color:#F5222D;">此时线程B操作中有两种情况:</font>**

**<font style="color:#2F54EB;">如果锁获取成功</font>**，直接替换Mark Word中的线程ID为B自己的ID(A→B)，重新偏向于其他线程(即将偏向锁交给其他线程，相当于当前线程"M被"释放了锁)，该锁会保持偏向锁状态，A线程Over，B线程上位;

![image-20260111220011484](.\image\image-20260111220011484.png)

**<font style="color:#2F54EB;">如果锁获取失败</font>**，则偏向锁升级为轻量级锁（**<font style="color:#F5222D;">设置偏向锁标识为0并设置锁标志位为00</font>**)，此时轻量级锁由原持有偏向锁的线程持有，继续持续其同步代码，而正在竞争的线程B会自动进入自旋等待获得该轻量级锁。

![image-20260111220045929](.\image\image-20260111220045929.png)

**<font style="color:#F5222D;">轻量级锁的加锁</font>**

JVM会为每个线程在当前线程的栈帧中创建用于存储锁记录的空间，官方成为Displaced Mark Word。若一个线程获得锁时发现是轻量级锁，会把锁的MarkWord复制到自己的Displaced Mark Word里面。然后线程尝试用CAS将锁的MarkWord替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示Mark Word已经被替换成了其他线程的锁记录，说明在与其它线程竞争锁，当前线程就尝试使用自旋来获取锁。

自旋CAS:不断尝试去获取锁，能不升级就不往上捅，尽量不要阻塞

**<font style="color:#F5222D;">轻量级锁的释放</font>**

在释放锁时，当前线程会使用CAS操作将Displaced Mark Word的内容复制回锁的Mark Word里面。如果没有发生竞争，那么这个复制的操作会成功。如果有其他线程因为自旋多次导致轻量级锁升级成了重量级锁，那么CAS操作会失败,此时会释放锁并唤醒被阻塞的线程。

竞争轻量级锁失败时，自旋尝试抢占锁

轻量级锁每次退出同步块都需要释放锁，而偏向锁是在竞争发生时才释放锁

### 6. 重量级锁
> 重量级锁   指向互斥量（重量级锁）的指针
>

![image-20260111220107511](.\image\image-20260111220107511.png)

**<font style="color:#1890FF;">重量级锁原理</font>**

**<font style="color:#000000;">Java中Synchronized的重量级锁，是基于进入和退出Monitor对象实现的，在编译时会将同步代码块的开始位置插入monitor enter指令，在结束位置插入monitor exit指令。</font>**

**<font style="color:#000000;">当线程执行到monitor enter指令时，会尝试获取对象所对应的Monitor所有权，如果获取到了，即获取到了锁，会在Monitor的owner中存放当前线程的id，这样它将会处于锁定状态，除非退出同步块，否则其他线程无法获取到这个Monitor。</font>**

## <font style="color:#000000;">三、总结</font>
### <font style="color:#F5222D;">1.锁发生升级后，请问hashcode去哪里了？</font>

![image-20260111220134872](.\image\image-20260111220134872.png)

锁升级为轻量级或重量级后，Mark Word中保存的分别是**<font style="color:#E8323C;">线程里面的锁记录指针</font>**和**<font style="color:#E8323C;">重量级锁指针</font>**，已经没有

位置在保存哈希码了，GC年龄了，**<font style="color:#E8323C;">那么这些信息被移动到哪里去了呢？</font>**

![image-20260111220154243](.\image\image-20260111220154243.png)

**<font style="color:#2F54EB;">在无锁状态下</font>**，Mark Word中可以存储对象的identity hash code值。当对象的hashCode()方法第一次被调用时，JVM会生成对应的identity hash code值并将该值存储到Mark Word中。

**<font style="color:#2F54EB;">对于偏向锁，</font>**在线程获取偏向锁时，会用Thread lD和epoch值覆盖identity hash code所在的位置。如果一个对象的hashCode()方法已经被调用过一次之后，这个对象不能被设置偏向锁。因为如果可以的化，那Mark Word中的identity hash code必然会被偏向线程ld给覆盖，这就会造成同一个对象前后两次调用hashCode()方法得到的结果不一致。

**<font style="color:#2F54EB;">升级为轻量级锁时</font>**，JVM会在当前线程的栈帧中创建一个锁记录(Lock Record)空间，用于存储锁对象的Mark Word拷贝，该拷贝中可以包含identity hash dode，所以**<font style="color:#F5222D;">轻量级锁可以和identity hash code共存</font>**，哈希码和GC年龄自然保存在此，释放锁后会将这些信息写回到对象头。



**<font style="color:#F5222D;">升级为重量级锁后</font>**，Mark Word保存的重量级锁指针，代表重量级锁的ObjectMonitor类里有字段记录非加锁状态下的Mark Vfgord，锁释放后也会将信息写回到对象头。

```java
    public static void main(String[] args) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        Object o = new Object();
        System.out.println("本应该是偏向锁");
        System.out.println(ClassLayout.parseInstance(o).toPrintable());

        o.hashCode();
        synchronized (o){
            System.out.println("本应该是偏向锁,调用了hashcode");
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }

                // 发生重量级
        TimeUnit.SECONDS.sleep(5);
        Object o = new Object();
        System.out.println("本应该是偏向锁");
        System.out.println(ClassLayout.parseInstance(o).toPrintable());

        synchronized (o){
            o.hashCode();
            System.out.println("本应该是偏向锁,调用了hashcode");
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
    }
```

![image-20260111220219793](.\image\image-20260111220219793.png)

![image-20260111220232007](.\image\image-20260111220232007.png)

### 2.各种锁的优缺点，syn实现原理
![](.\image\image-20260111220412486.png)

**<font style="color:#F5222D;">偏向锁</font>**: 适用于单线程适用的情况，在不存在锁竞争的时候进入同步方法/代码块则使用偏向锁。

**<font style="color:#F5222D;">轻量级锁</font>**: 适用于竞争较不激烈的情况(这和乐观锁的使用范围类似)，存在竞争时升级为轻量级锁，轻量级锁采用的是自旋锁，如果同步方法/代码块执行时间很短的话，采用轻量级锁虽然会占用cpu资源但是相对比使用重量级锁还是更高效。

**<font style="color:#F5222D;">重量级锁</font>**：适用于竞争激烈的情况，如果同步方法/代码块执行时间很长，那么使用轻量级锁自旋带来的性能消耗就比使用重量级锁更严重，这时候就需要升级为重量级锁。

### 3.JIT编译器对锁的优化
**<font style="color:#F5222D;">锁消除</font>**

```java
    static Object objectLock= new Object();
    public void m1(){
        //锁消除问题，JIT编译器无视他，synchronized(o)，每次new出来，不存在
        //每个线程一把锁
        Object o = new Object();
        synchronized (o){
            System.out.println("----hello lockjit"+"\t"+o.hashCode()+"\t"+objectLock.hashCode());
        }
    }

    public static void main(String[] args) {
        LockJIT lockJIT = new LockJIT();
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                lockJIT.m1();
            },String.valueOf(i)).start();
        }
    }
```

**<font style="color:#F5222D;">锁粗化</font>**

```java
  // 假如方法中首尾相接，前后相邻的都是同一个锁对象，那JIT编译器就会把这几个synchronized块合并成一个大块，
    // 加粗加大范围，一次申请锁使用即可，避免此次的申请和释放锁，提升了性能。
    static Object objectLock=new Object();

    public static void main(String[] args) {
        new Thread(()->{
            // 叫做锁粗化
            // JIT 底层会给你将锁 优化成一个锁
            synchronized (objectLock){
                System.out.println("11111");
            }
            synchronized (objectLock){
                System.out.println("22222");
            }
            synchronized (objectLock){
                System.out.println("3333");
            }
            synchronized (objectLock){
                System.out.println("44444");
            }
        },"t1").start();
    }
```

