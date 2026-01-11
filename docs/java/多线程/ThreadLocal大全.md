---
description: 深入解析 ThreadLocal 的设计思想与底层实现机制，从 Thread / ThreadLocal / ThreadLocalMap 的关系入手，结合源码剖析弱引用、内存泄漏成因及线程池场景下的风险，并总结 ThreadLocal 的正确使用姿势与最佳实践。
title: ThreadLocal 全面解析：源码原理、内存泄漏与最佳实践
tag:
  - 多线程
sidebar: true
comment: true
recommend: 1
---
# ThreadLocal 全面解析：源码原理、内存泄漏与最佳实践

> 该类提供线程局部变量

问题：

1. ThreadLocal中的ThreadLocalMap的数据结构和关系
2. ThreadLocal的key是弱引用，这是为什么？
3. ThreadLocal内存泄漏问题你知道吗?
4. ThreadLocal中最后为什么要加remove方法？

## 一、是什么		

ThreadLocal提供线程局部变量。这些变量与正常的变量<font color='red'>**不同**</font>，因为每一个线程在访问ThreadLocal实例的时候(通过其get或set方法)<font color='red'>**都有自己的、独立初始化的变量副本**</font>。ThreadLocal实例通常是类中的私有静态字段，使用它的目的是希望将状态（例如，用户ID或事务ID)与线程关联起来。

## 二、能干嘛

实现**每一个线程都有自己专属的本地变量副本**(自己用自己的变量不麻烦别人，不和其他人共享，人人有份，人各一份)，**主要解决了让每个线程绑定自己的值，通过使用get()和lset()方法，获取默认值或将其值更改为当前线程所存的副本的值从而避免了线程安全问题**，比如我们之前讲解的8锁案例，资源类是使用同一部手机，多个线程抢夺同一部手机使用，假如人手一份是不是天下太平? ?

![image-20260108215940688](.\image\image-20260108215940688.png)

![image-20260108215948916](.\image\image-20260108215948916.png)

## 三、api介绍

> 五种方法

![image-20260108220011386](.\image\image-20260108220011386.png)

## 四、永远的helloworld讲起

> 五个销售卖房子，集团高层只关心销售总量的准确统计数，按照总销售额统计，方便集团公司给部分发送奖金

加锁

```java
class House{
    int saleCount=0;
    public synchronized void saleHorse(){
        ++saleCount;
    }
}

public class ThreadLocalDemo {
    public static void main(String[] args) throws InterruptedException {
        House house = new House();

        for(int i =1;i<=5;i++){
            new Thread(()->{
                int size = new Random().nextInt(5) + 1;
                System.out.println(size);
                for (int j = 0; j < size; j++) {
                    house.saleHorse();
                }
            },String.valueOf(i)).start();
        }

        TimeUnit.MILLISECONDS.sleep(300);
        System.out.println(Thread.currentThread().getName()+"\t"+house.saleCount);
    }
}
```

> 改需求：比如某房产中介销售都有自己的销售额指标，自己专属自己的，不和别人掺和。

<font color='red'>**不加锁解决线程安全问题**</font>

```java
class House{
    int saleCount=0;
    public synchronized void saleHorse(){
        ++saleCount;
    }
    ThreadLocal<Integer> saleVolume = ThreadLocal.withInitial(()->0);
    public void saleVolumeByThreadLocal(){
        saleVolume.set(saleVolume.get()+1);
    }
}

public class ThreadLocalDemo {
    public static void main(String[] args) throws InterruptedException {
        House house = new House();
        for(int i =1;i<=5;i++){
            new Thread(()->{
                int size = new Random().nextInt(5) + 1;
                for (int j = 0; j < size; j++) {
                    house.saleHorse();
                    house.saleVolumeByThreadLocal();
                }
                System.out.println(Thread.currentThread().getName()+"\t"+"卖出"+house.saleVolume.get());
            },String.valueOf(i)).start();
        }

        TimeUnit.MILLISECONDS.sleep(300);
        System.out.println(Thread.currentThread().getName()+"\t"+house.saleCount);
    }
}
```

1. <font color='red'>**必须回收自定义的ThreadLocal变量，**</font>尤其在线程池场景下，线程经常复用，ThreadLocal会出问题

**场景复现**

```java
class MyData{
    ThreadLocal<Integer> threadLocalfield = ThreadLocal.withInitial(()->0);
    public void add(){
        threadLocalfield.set(1+threadLocalfield.get());
    }
}
public class ThreadDemo2 {
    public static void main(String[] args) {
        MyData myData = new MyData();
        ExecutorService threadPool = Executors.newFixedThreadPool(3);
        try {
            for (int i =0;i<=10;i++){
                threadPool.submit(()->{
                    Integer beforInt = myData.threadLocalfield.get();
                    myData.add();
                    Integer afterInt = myData.threadLocalfield.get();
                    System.out.println(Thread.currentThread().getName()+"\t before"+beforInt+"\t after"+afterInt);
                });
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }finally {
            threadPool.shutdown();
        }
    }
}
```

阿里巴巴ThreadLocal规范
![image-20260108220432204](.\image\image-20260108220432204.png)
- 每个Thread内有自己的**实例副本**且该副本只由当前线程自己使用
- 既然其他Thread不可访问，那就不存在多个线程共享的问题
- 统一设置初始值，但是每个线程对这个值得修改都是各自线程互相独立的。
总结:
1. 加入synchronized或者Lock控制资源的访问顺序 
2. 人手一份,大家各自安好没必要争抢。
### 1.ThreadLocal源码分析
Thread、`ThreadLocal`、`ThreadLocalMap`的关系
每次 new 一个线程都会有一个 `ThreadLocalMap`
![image-20260108220527672](.\image\image-20260108220527672.png)
**再次体会，各自线程，人手一份**
![image-20260108220543443](.\image\image-20260108220543443.png)
**关系图**
![image-20260108220613054](.\image\image-20260108220613054.png)
总结：
`ThreadLocalMap`从字面上就可以看出这是一个保存ThreadLocal对象的map(其实是以ThreadLocal为Key)，不过是经过了两层包装的ThreadLocal对象:
![image-20260108220632887](.\image\image-20260108220632887.png)
<font color='red'>**JVM内部维护了一个线程版的Map<ThreadLocal,Value>**(**通过ThreadLocal对象的set方法，结果把ThreadLocal对象自己当做key,放进了ThreadLoalMap中**)</font>,每个线程要用到这个T的时候，用当前的线程去Map里面获取，**通过这样让每个线程都拥有了自己独立的变量**，人手一份，竞争条件被彻底消除，在并发模式下是绝对安全的变量。

### 2.ThreadLocal 内存泄漏问题

>  内存泄漏： <font color='green'>**不再会被使用的对象或者变量占用的内存不能被回收，就是内存泄漏。**</font>

因为threadlocalmap使用了弱引用

![image-20260108220737793](.\image\image-20260108220737793.png)

​                  **ThreadLocalMap 与 WeakReference**

ThreadLocalMap从字面上就可以看出这是一个保存ThreadLocal对象的map(以ThreadLocal为Key)，不过是经过了两层包装的ThreadLocal对象:

(1）第一层包装是使用`WeakReference`<ThreadLocal<?>>将ThreadLocal对象<font color='red'>**变成一个弱引用的对象;**</font>

(2）第二层包装是定义了一个专门的类Entry来扩展`WeakReference`<ThreadLocal<?>>；

#### 四种引用分别是什么？

##### 整体架构

![image-20260108220846394](.\image\image-20260108220846394.png)

JAVA技术允许使用<font color='red'>**finalize()** </font>方法在垃圾收集器将对象从内存中清除出去之前做必要的清理工作

##### 强引用

<font color='red'>**当内存不足，JVM开始垃圾回收，对于强引用的对象，就算是出现了OOM也不会对该对象进行回收，**</font>死都不收。

强引用是我们最常见的普通对象引用，只要还有强引用指向一个对象，就能表明对象还“活着”，垃圾收集器不会碰这种对象。

在Java中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，它处于可达状态，它是不可能被垃圾回收机制回收的，**即使该对象以后永远都不会被用到，JVM也不会回收。**因此强引用是造成Java内存泄漏的主要原因之一。

对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为 null,一般认为就是可以被垃圾收集的了(当然具体回收时机还是要看垃圾收集策略)。

```java
class MyObject{
    @Override
    protected void finalize() throws Throwable {
        //finalize 的通常目的是在对象被不可撤的丢弃之前执行清理操作
        System.out.println("--------invoke finalize method~");
    }
}

public class ReferenceDemo {

    public static void main(String[] args) {
        MyObject myObject = new MyObject();
        System.out.println("gc before： "+myObject);
        myObject = null;
        //人工开启GC ，一般不用
        System.gc();
        System.out.println("gc after： "+myObject);
    }
}
```

![image-20260108220957121](.\image\image-20260108220957121.png)

##### 软引用

软引用是一种相对强引用弱化了一些的引用，需要用java.lang.ref.SoftReference类来实现，可以让对象豁免一些垃圾收典。

对于只有软引用的对象来说：

​		<font color='blue'>**当系统内存充足时它不会被回收，**</font>

​		<font color='blue'>**当系统内存不足时它会被回收。**</font>

软引用通常用在对内存敏感的程序中，比如高速缓存就有用到软引用，**内存够用的时候就保留，不够用就回收!**

```java
class MyObject{
    @Override
    protected void finalize() throws Throwable {
        //finalize 的通常目的是在对象被不可撤的丢弃之前执行清理操作
        System.out.println("--------invoke finalize method~");
    }
}

public class ReferenceDemo {
    public static void main(String[] args) throws InterruptedException {
        SoftReference<MyObject> softReference = new SoftReference<>(new MyObject());
        System.out.println("----softReference"+softReference.get());
        System.gc();
        TimeUnit.SECONDS.sleep(1);
        System.out.println("----gc after内存够用"+softReference.get());
        try {
            byte[] bytes = new byte[20 * 1024 * 1024];
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            System.out.println("内存不够:"+softReference.get());
        }

    }

    public static void StrongReference(String[] args) {
        MyObject myObject = new MyObject();
        System.out.println("gc before： "+myObject);
        myObject = null;
        //人工开启GC ，一般不用
        System.gc();
        System.out.println("gc after： "+myObject);
    }
}
```

![image-20260108221116768](.\image\image-20260108221116768.png)

##### 弱引用

弱引用需要用java.lang.ref.WeakReference类来实现，它比软引用的生存期更短，

**对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管JVM的内存空间是否足够，都会回收该对象占用的内存。.**

```java
class MyObject{
    @Override
    protected void finalize() throws Throwable {
        //finalize 的通常目的是在对象被不可撤的丢弃之前执行清理操作
        System.out.println("--------invoke finalize method~");
    }
}

public class ReferenceDemo {
    
    public static void main(String[] args) throws InterruptedException {
        WeakReference<MyObject> weakReference = new WeakReference<>(new MyObject());
        System.out.println("----gc after内存够用"+weakReference.get());
        System.gc();
        TimeUnit.SECONDS.sleep(1);
        System.out.println("内存不够:"+weakReference.get());
    }
}
```

![image-20260108221153101](.\image\image-20260108221153101.png)

<font color='blue'>***软引用的适用场景：***</font>

假如有一个应用需要读取大量的本地图片

- <font color='blue'>**如果每次读取图片都从硬盘读取则会严重影响性能。**</font>
- <font color='blue'>**如果一次性全部加载到内存中又可能造成内存溢出。**</font>

此时使用软引用可以解决这个问题。

设计思路是：用一个HashMap来保存图片的路径和相应图片对象关联的软引用之间的映射关系，在内存不足时，JVM会自动回收这些换成图片对象所占用的空间，从而有效避免了OOM的问题。

##### 虚引用

1. <font color='blue'>**虚引用必须和引用队列(ReferenceQueue)联合使用**</font>

虚引用需要java.lang.ref.PhantomReference类来实现,顾名思义，<font color='blue'>**就是形同虚设**</font>，与其他几种引用都不同，虚引用并不会决定对象的生命周期。<font color='red'>**如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收**</font>，它不能单独使用也不能通过它访问对象，虚引用必须和引用队列(ReferenceQueue)联合使用

2. <font color='blue'>**PhantomReference的get方法总是返回null**</font>

虚引用的主要作用是跟踪对象被垃圾回收的状态。<font color='red'>**仅仅是提供了一种确保对象被finalize以后，做某些事情的通知机制。PhantomReference的get方法总是返回null**</font>，因此无法访问对应的引用对象。

3. <font color='blue'>**处理监控通知使用**</font>

换句话说，设置虚引用关联对象的唯一目的，就是在这个对象被收集器回收的时候收到一个系统通知或者后续添加进一步的处理，用来实现比finalize机制更灵活的回收操作

```java
 public static void main(String[] args) {
        MyObject myObject = new MyObject();
        //引用队列
        ReferenceQueue<MyObject> referenceQueue = new ReferenceQueue<>();
        //虚引用
        PhantomReference<MyObject> myObjectPhantomReference = new PhantomReference<>(myObject,referenceQueue);
        System.out.println(myObjectPhantomReference.get());

        List<byte []> list = new ArrayList<>();
        new Thread(()->{
            while (true){
                list.add(new byte[1*1024*1024]);
                try {
                    TimeUnit.MILLISECONDS.sleep(500);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                System.out.println(myObjectPhantomReference.get()+"\t"+"list add ok");
            }
        },"t1").start();


        new Thread(()->{
            while (true){
                Reference<? extends MyObject> reference = referenceQueue.poll();
                if(reference!=null){
                    System.out.println("-------有虚对象回收假如了队列");
                }
            }
        },"t2").start();
    }
```

![image-20260108221600669](.\image\image-20260108221600669.png)

![image-20260108221607502](.\image\image-20260108221607502.png)

#### 关系

![image-20260108221619252](.\image\image-20260108221619252.png)

<font color='blue'>**ThreadLocal是一个壳子，真正的存储结构是ThreadLocal里有ThreadLocalMap这么个内部类**</font>，<font color='red'>**每个Thread对象维护着一个TheadLocalMap的引用**</font>，ThreadLocalMap是ThreadLocal的内部类，用Entry来进行存储。

1)调用ThreadLocal的set)方法时，实际上就是往ThreadLocalMap设置值，key是ThreadLocal对象，值Value是传递进来的对象

2)调用ThreadLocal的get()方法时，实际上就是往ThreadLocalMap获取值，key是ThreadLocal对象

ThreadLocal本身并不存储值(ThreadLocal是一个壳子)，它只是自己作为一个key来让线程从ThreadLocalMap获取value。正因为这个原理，所以ThreadLocal能够实现“数据隔离”，获取当前线程的局部变量值，不受其他线程影响～

#### 为什么Entry使用弱引用

```java
    public void function(){
        ThreadLocal<String> tl = new ThreadLocal<>();
        tl.set("123456");
        tl.get()
    }
```

line1新建了一个ThreadLocal对象，t1是强引用指向这个对象;

line2调用set()方法后新建一个Entry，通过源码可知**Entry对象里的k是弱引用**指向这个对象。

![image-20260108221829892](.\image\image-20260108221829892.png)

##### 1.为什么源代码用弱引用?

当function方法执行完毕后，栈帧销毁强引用 tl 也就没有了。但此时线程的ThreadLocalMap里某个entry的key引用还指向这个对象

若这个key引用是<font color='blue'>**强引用**</font>，就会**导致key指向的ThreadLocal对象及v指向的对象不能被gc回收，造成内存泄漏**;

若这个key引用是<font color='blue'>**弱引用**</font>就**大概率**会减少内存泄漏的问题(**还有一个key为snull的雷，第2个坑后面讲**)。

使用弱引用，就可以使ThreadLocal对象在方法执行完毕后顺利被回收且**Entry的key引用指向为null**。

##### 2.弱引用就万事大吉了吗？

若这个key引用是<font color='blue'>**弱引用**</font>就<font color='red'>**大概率**</font>会减少内存泄漏的问题（<font color='red'>**还有一个key为null的雷，第二个坑后面讲**</font>）。使用弱引用，就可以使ThreadLocal对象在方法执行完毕后顺利被回收且Entry的<font color='red'>**key引用指向为null。**</font>

<font color='red'>**原因：ThreadLocalMap使用ThreadLocal的弱引用作为key**</font>，如果一个ThreadLocal没有外部强引用引用他，那么系统gc的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key约null的Entry的value，**如果当前线程再迟迟不结束的话(比如正好用在线程池)，这些key为null的Entry的value就会一直存在一条强引用链。**

**重点：**虽然弱引用，保证了 key指向的ThreadLocal对象能被及时回收，但是v指向的value对象是需要ThreadLocalMap调用get、set发现key为null才会去回收整个enty 、 value，<font color='red'>**因此弱引用不能100%保证内存不泄露。我们要在不使用某个ThreadLocal对象后，手动调用remove方法来去除它**</font>，尤其是在线程池中，不仅仅是内存泄露的问题**，因为线程池中的线程是重复使用的，意味着这个线程的ThreadLocalMap对象也是重复使用的，如果我们不手动调用remove方法，那么后面的线程就有可能获取到上个线程遗留下来的value值，造成bug。**

##### 3.set、get方法会去检查所以键为null的Entry对象

- **expungeStaleEntry 清除旧的状态**
- **set()**
- **get()**
- **remove() clear 将引用设置为null**
- **结论**

- - 从前面的set.getEntry.remove方法看出，在threadLocal的生命周期里，针对threadLocal存在的内存泄漏的问题,都会通过expungeStaleEntry.cleanSomeSlots,replaceStaleEntry这三个方法清理掉key为null的脏entry.

## 五、总结

### 1.最佳实践

- 尽量使用ThreadLocal.withInitial(()-> 初始化值);
- 建议把ThreadLocal修饰为static（ThreadLocal能实现了线程的数据隔离，不在于它自己本身，而在于Thread的ThreadLocalMap，**所以ThreadLocal可以只初始化一次，只分配一块存储空间就足以了，没必要酢为成员变量多次被初始化。**）
- **用完记得手动remove（强制）**

![image-20260108222406736](.\image\image-20260108222406736.png)