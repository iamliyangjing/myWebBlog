---
description: 本文全面介绍了 Java 中的 Future 接口及其实现类 FutureTask，重点剖析了其在异步多线程任务执行中的优缺点。通过对比传统 Future 的阻塞与轮询问题，引出了更强大的 CompletableFuture 框架，详细讲解其核心方法、使用场景以及实际电商比价案例，帮助开发者掌握现代异步编程的核心技术，提升系统性能与代码可维护性。
title: 深入解析 Java CompletableFuture：异步编程的未来与实践
tag:
  - 多线程
sidebar: true
comment: true
recommend: 3
---

## 一、接口理论

Future接口（Future实现类）定义了操作<font style="color:#E8323C;">异步任务</font><font style="color:#E8323C;">执行一些方法</font>，如获取异步任务的执行结果，取消任务的执行、判断任务是否被取消、判断任务执行是否完毕等。

比如主线程让一个子线程去执行任务，子线程可能比较耗时，启动子线程开始执行任务后，

主线程就去做其他事情了，忙其它事情或者先执行完，过了一会才去获取子任务的执行结果或变更的任务状态。

## 二、Future接口常用实现类FutureTask异步任务

### 1.future接口能干什么

Future是java5新加的一个接口，它提供了一种<font style="color:#E8323C;">异步并行计算的功能</font>。

如果主线程需要执行一个很耗时的计算任务，我们就可以通过future把这个任务放到异步线程中执行。

主线程继续处理其他任务或者先行结束，再通过Future获取计算结果。

代码说话：

**Runnable接口**

**Callable接口**

**Future接口和FutureTask实现类**

**<font style="color:#E8323C;">目的：异步多线程任务执行且返回有结果，三个特点：多线/有返回/异步任务</font>**

**<font style="color:#E8323C;">（班长为老师去买水作为新启动的异步多线程任务且买到水有结果返回）</font>**

futureTask实现Callable    thread接口

![image-20260111214031309](.\image\image-20260111214031309.png)

他的构造方法含有**<font style="color:#E8323C;"> callable 和runnable </font>**接口

futureTask 如何获取返回值

```java
public class CompletableFutureDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<String> futureTask = new FutureTask<>(new MyThread2());
        Thread t1 = new Thread(futureTask, "t1");
        t1.start();;
        System.out.println(futureTask.get());
    }
}

class MyThread implements Runnable{

    @Override
    public void run() {

    }
}

class MyThread2 implements Callable<String>{

    @Override
    public String call() throws Exception {
        System.out.println("---- come in call()");
        return "hello callable!";
    }
}
```



### 2.Future编码实战和优缺点分析

#### 1.优点

**<font style="color:#E8323C;">future+线程池异步多线程任务配合，能显著提高程序的执行效率</font>**

<font style="color:#000000;">例子说明：</font>

使用单个线程

```java
    private static void  m1(){
        //3个任务，目前只有一个main线程来处理，请问耗时多久？
        long startTime = System.currentTimeMillis();

        try {
            TimeUnit.MILLISECONDS.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        try {
            TimeUnit.MILLISECONDS.sleep(300);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        try {
            TimeUnit.MICROSECONDS.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        long endTime = System.currentTimeMillis();

        System.out.println("-----costTime"+(endTime-startTime)+"ms");
        System.out.println(Thread.currentThread().getName()+"t---end");
    }
```

使用多future+线程池异步创建线程

```java
    public static void main(String[] args) {
        //3个任务，目前开启多个异步任务线程来处理，请问耗时多久？
        ExecutorService threadPool = Executors.newFixedThreadPool(3);
        long startTime = System.currentTimeMillis();

        FutureTask<String> futureTask = new FutureTask<>(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "task1 over";
        });
        threadPool.submit(futureTask);

        FutureTask<String> futureTask1 = new FutureTask<>(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "task1 over";
        });
        threadPool.submit(futureTask1);
        try {
            TimeUnit.MILLISECONDS.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long endTime = System.currentTimeMillis();
        System.out.println("-----costTime"+(endTime-startTime)+"ms");
    }
```

**<font style="color:#E8323C;">为什么使用线程池：如果用new Thread（future）；这样创建线程则会new 三次Thread。</font>**

![image-20260111214048219](.\image\image-20260111214048219.png)

#### 2.缺点

##### A.  get()阻塞

+ _<font style="color:#629755;">get 容易导致阻塞，一般建议放在程序后面，一旦调用不见不散，非要等到结果才会离开，不管你是否计算完成，容易程序阻塞。</font>_
+ _<font style="color:#629755;">假如我不愿意等待很长时间，我希望过时不候，可以自动离开。  </font>

例子1：

**<font style="color:#000000;">若在业务代码前调用get（），则会阻塞get（）后面的代码</font>**

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        FutureTask<String> futureTask = new FutureTask<String>(() -> {
            System.out.println(Thread.currentThread().getName() + "\t------come in");
            //暂停几秒钟线程
            TimeUnit.SECONDS.sleep(5);
            return "task over";
        });
        Thread t1 = new Thread(futureTask, "t1");
        t1.start();
        // 不见不散 非要等到结果才会离开，不管你是否计算完成
        System.out.println(futureTask.get());
        System.out.println(Thread.currentThread().getName()+"\t ----忙其它任务了");
    }
```

过时不候，给get（）设置超时时间

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        FutureTask<String> futureTask = new FutureTask<String>(() -> {
            System.out.println(Thread.currentThread().getName() + "\t------come in");
            //暂停几秒钟线程
            TimeUnit.SECONDS.sleep(5);
            return "task over";
        });
        Thread t1 = new Thread(futureTask, "t1");
        t1.start();
        // 不见不散 非要等到结果才会离开，不管你是否计算完成
//        System.out.println(futureTask.get());
        System.out.println(Thread.currentThread().getName()+"\t ----忙其它任务了");
        System.out.println(futureTask.get(3,TimeUnit.SECONDS));
    }
```

##### B. IsDone()轮询

+ **轮询的方式会耗费无谓的CPU资源，而且也不见得能及时地得到计算结果。**
+ **如果想要异步获得结果，通常都会以轮询的方式去获取结果，尽量不要阻塞**

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        FutureTask<String> futureTask = new FutureTask<String>(() -> {
            System.out.println(Thread.currentThread().getName() + "\t------come in");
            //暂停几秒钟线程
            TimeUnit.SECONDS.sleep(5);
            return "task over";
        });
        Thread t1 = new Thread(futureTask, "t1");
        t1.start();
        // 不见不散 非要等到结果才会离开，不管你是否计算完成
        System.out.println(Thread.currentThread().getName()+"\t ----忙其它任务了");
//        System.out.println(futureTask.get(3,TimeUnit.SECONDS));
        while (true){
            if(futureTask.isDone()){
                System.out.println(futureTask.get());
                break;
            }else {
                //暂停线程
                TimeUnit.MILLISECONDS.sleep(500);
                System.out.println("正在处理中");
            }
        }
    }
```

#### 3.结论

**<font style="color:#F5222D;">Future对于结果的获取不是很友好，只能通过</font>**<font style="color:#FADB14;">阻塞或轮询的方式</font><font style="color:#F5222D;">得到任务的结果</font>

### <font style="color:#000000;">3.想完成一些复杂的任务</font>

#### 1.回调通知

应对Future的完成时间，完成了可以告诉我，也就是我们的回调通知

通过轮询的方式去判断任务是否完成这样非常**<font style="color:#F5222D;">占CPU并且代码不优雅</font>**

#### 2.创建异步任务

Future+线程池配合

#### 3.多个任务前后依赖可以组合处理（水煮鱼）

想将多个异步任务的计算结果组合起来**<font style="color:#F5222D;">，后一个异步任务的计算结果需要前一个异步任务的值</font>**，将两个或多个异步技术合成一个异步计算，这几个异步计算互相独立，同时后面又依赖前一个处理的结果。

![image-20260111214100719](.\image\image-20260111214100719.png)

**ps -ef|grep tomcat 过滤**

**就像打游戏一样，两个线程A、B，有一个win 就通知另一个线程.**

#### 4.对计算速度最快的

当Future集合某个任务最快结束时，返回结果，返回第一名处理结果。

**<font style="color:#F5222D;">Future能做的，CompletableFuture也能做</font>**

## 三、CompletableFuture对Future的改进

### 1.CompletableFuture 为什么出现

get()方法在Future计算完成之前会一直处在<font style="color:#F5222D;">阻塞状态</font>下，

IsDone()方法容易耗费CPU资源,

对于真正的异步处理我们希望是可以通过传入回调函数，在Future结束时自动调用该回调函数，这样，我们就不用等待结果。

**<font style="color:#F5222D;">阻塞的方式和异步编程的设计理念相违背，而轮询的方式会耗费无谓的CPU资源</font>**。因此，JDK8设计出了CompletableFuture。

CompletableFuture提供了一种**<font style="color:#F5222D;">观察者模式类似</font>**的机制，可以让任务执行完成后通知监听的一方。

### 2.CompletableFuture和CompletionStage源码分别介绍类架构
![image-20260111214119587](.\image\image-20260111214119587.png)

Future只有5个方法，而CompletionStage**<font style="color:#F5222D;">有很多API</font>**

![image-20260111214131685](.\image\image-20260111214131685.png)

![image-20260111214144463](.\image\image-20260111214144463.png)
![image-20260111214156675](.\image\image-20260111214156675.png)
![image-20260111214211625](.\image\image-20260111214211625.png)

### 3.核心的四个静态方法，来创建一个异步任务

#### A.runAsync

+ **<font style="color:#F5222D;">runAsync </font>**无返回值

```java
	// 线程池构造
	public static CompletableFuture<Void> runAsync(Runnable runnable,
                                                   Executor executor) {
        return asyncRunStage(screenExecutor(executor), runnable);
    }
	// 默认使用的是Fork-Joinpool线程池
    public static CompletableFuture<Void> runAsync(Runnable runnable) {
        return asyncRunStage(asyncPool, runnable);
    }
```

#### B.supplyAsync

+ **<font style="color:#F5222D;">supplyAsync </font>**有返回值

```java
	// 线程池构造
	public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                       Executor executor) {
        return asyncSupplyStage(screenExecutor(executor), supplier);
    }
	// 默认使用的是Fork-Joinpool线程池
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }
```

+ 上述Excutor executor参数说明

**没有指定Executor的方法，直接使用默认的Fork.JoinPool.commonPool()**作为它的线程池执行异步代码。

如果指定线程池，则使用我们自定义的或者特别指定的线程池执行异步代码

**<font style="color:#F5222D;">无返回值</font>**

```java
//使用自定义的线程池  
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService threadPool = Executors.newFixedThreadPool(3);

        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            System.out.println(Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },threadPool);
        System.out.println(future.get());
    }
//使用默认的线程池
public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            System.out.println(Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(future.get());
}
```

**<font style="color:#F5222D;">有返回值</font>**

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService threadPool = Executors.newFixedThreadPool(3);
        CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "hello world";
        },threadPool);
        System.out.println(completableFuture.get());

        threadPool.shutdown();;
    }
```

尽量不要使用new CompletableFuture 来创建，而是使用**<font style="color:#F5222D;">runAsync 和SupplyAsync  </font>**

+ Code**<font style="color:#F5222D;">通用演示</font>**，**<font style="color:#871400;">减少阻塞和轮询</font>**

> 从Java8开始引入了CompletableFuture，**<font style="color:#F5222D;">它是Future的功能增强版</font>**，**<font style="color:#FF4D4F;">减少阻塞和轮询</font>**，可以传入回调对象，当异步任务完成或者发送异常时，自动调用回调对象的回调方法

可以完全替代Future接口

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println(Thread.currentThread().getName() + "---- come in");
            int result = ThreadLocalRandom.current().nextInt(10);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("----- 1秒钟后出结果" + result);
            return result;
        });
        System.out.println(Thread.currentThread().getName()+"线程先去忙其他任务了");
        System.out.println(completableFuture.get());
    }
```

![image-20260111214220774](.\image\image-20260111214220774.png)

1.主线程不要立刻结束，否则**<font style="color:#F5222D;">completablefuture默认使用的线程池会立刻关闭“暂停3秒钟线程</font>**

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture.supplyAsync(()->{
            System.out.println(Thread.currentThread().getName() + "---- come in");
            int result = ThreadLocalRandom.current().nextInt(10);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("----- 1秒钟后出结果" + result);
            return result;
            //v：上一步的值  e ：触发的异常
        }).whenComplete((v,e)->{
            if(e==null){
                //为什么没有打印 出这句话
                //因为main线程结束了 我们的completable的默认线程池forkjoin相当于守护线程
                System.out.println("---计算完成，更新系统updateVa:"+v);
            }
        }).exceptionally(e->{
            e.printStackTrace();
            System.out.println("异常情况："+e.getCause()+"\t"+e.getMessage());
            return null;
        });

        System.out.println(Thread.currentThread().getName()+"先去忙其他任务");
        //主线程不要立刻结束，否则completablefuture默认使用的线程池会立刻关闭“暂停3秒钟线程
        TimeUnit.SECONDS.sleep(3);
    }
```

2.也可以**<font style="color:#F5222D;">自定义线程池</font>**，这样主线程main结束，completablefuture 也不会立刻结束

```java
        ExecutorService executorService = Executors.newFixedThreadPool(3);
        try {
            CompletableFuture.supplyAsync(()->{
                System.out.println(Thread.currentThread().getName() + "---- come in");
                int result = ThreadLocalRandom.current().nextInt(10);
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("----- 1秒钟后出结果" + result);
                return result;
                //v：上一步的值  e ：触发的异常
            },executorService).whenComplete((v,e)->{
                if(e==null){
                    //为什么没有打印 出这句话
                    //因为main线程结束了 我们的completable的默认线程池forkjoin相当于守护线程
                    System.out.println("---计算完成，更新系统updateVa:"+v);
                }
            }).exceptionally(e->{
                e.printStackTrace();
                System.out.println("异常情况："+e.getCause()+"\t"+e.getMessage());
                return null;
            });
            System.out.println(Thread.currentThread().getName()+"先去忙其他任务");
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            executorService.shutdown();
        }
```

出现**<font style="color:#F5222D;">异常</font>**

```java
      ExecutorService executorService = Executors.newFixedThreadPool(3);
        try {
            CompletableFuture.supplyAsync(()->{
                System.out.println(Thread.currentThread().getName() + "---- come in");
                int result = ThreadLocalRandom.current().nextInt(10);
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("----- 1秒钟后出结果" + result);
                if(result>2){
                    int i = 10/0;
                }
                return result;
                //v：上一步的值  e ：触发的异常
            },executorService).whenComplete((v,e)->{
                if(e==null){
                    //为什么没有打印 出这句话
                    //因为main线程结束了 我们的completable的默认线程池forkjoin相当于守护线程
                    System.out.println("---计算完成，更新系统updateVa:"+v);
                }
            }).exceptionally(e->{
                e.printStackTrace();
                System.out.println("异常情况："+e.getCause()+"\t"+e.getMessage());
                return null;
            });
            System.out.println(Thread.currentThread().getName()+"先去忙其他任务");
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            executorService.shutdown();
        }

```

![image-20260111214232489](.\image\image-20260111214232489.png)

+ CompletableFuture 的优点

1. 异步任务结束时，会自动回调某个对象的方法；
2. 主线程设置好回调后，不在关心异步任务的执行，异步任务之间可以顺序执行。
3. **<font style="color:#F5222D;">异步任务出错时，会自动回调某个对象的方法。</font>**

**<font style="color:#F5222D;"></font>**

### <font style="color:#000000;">4.案例精讲-从电商网站的比价需求说开去</font>

#### A.函数式编程已经主流

Lambda表达式+Stream流式调用+Chain链式调用+Java8函数式编程

![image-20260111214353902](.\image\image-20260111214353902.png)
![image-20260111214404938](.\image\image-20260111214404938.png)
![image-20260111214419841](.\image\image-20260111214419841.png)<!-- 这是一张图片，ocr 内

![image-20260111214429585](.\image\image-20260111214429585.png)

#### B.<font style="color:#F5222D;">说说join和get对比</font>

**get要抛出异常，join不用抛出异常**

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> {
            return "hello 1234";
        });
//      System.out.println(completableFuture.get());
        System.out.println(completableFuture.join());
    }
```

#### C. 大厂业务需求说明

> 需求说明

![image-20260111214439553](.\image\image-20260111214439553.png)

解决方案

![image-20260111214448412](.\image\image-20260111214448412.png)
功能版：

```java
public class CompletableFutureMallDemo {

    static List<NetMall> list = Arrays.asList(
            new NetMall("id"),
            new NetMall("dangdang"),
            new NetMall("taobao")
    );

    public static List<String> getPrice(List<NetMall> list,String productName){
        // 《mysql》 int taobao  price 19.43
        return list.stream().map(netMall -> String.format(productName+"in %s price is %.2f",
                netMall.getNetMallName(),netMall.calcPrice(productName))).collect(Collectors.toList());
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        long startTime = System.currentTimeMillis();
        List<String> k = getPrice(list, "mysql");
        for (String s : k) {
            System.out.println(s);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("----costTime"+(endTime-startTime)+"ms");
    }
}



class NetMall{
    @Getter
    private String netMallName;

    public NetMall(String netMallName){
        this.netMallName=netMallName;
    }

    public double calcPrice(String productName){
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return ThreadLocalRandom.current().nextDouble()*2+productName.charAt(0);
    }
}
```

性能版 （completableFuture）

```java
    /**
     * List<NetMall>----->List<CompletableFuture>---->List<String></>
     * @param list
     * @param productName
     * @return
     */
    public static List<String> getPriceCompletableFuture(List<NetMall> list,String productName){
        // 《mysql》 int taobao  price 19.43
        return list.stream().map(netMall -> CompletableFuture.supplyAsync(() -> String.format(productName + "in %s price is %.2f",
                        netMall.getNetMallName(),
                        netMall.calcPrice(productName)))).collect(Collectors.toList()).stream()
                .map(s -> s.join())
                .collect(Collectors.toList());
    }
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        long startTime = System.currentTimeMillis();
        List<String> k = getPrice(list, "mysql");
        for (String s : k) {
            System.out.println(s);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("----costTime"+(endTime-startTime)+"ms");

        long startTime2 = System.currentTimeMillis();
        k = getPriceCompletableFuture(list, "mysql");
        for (String s : k) {
            System.out.println(s);
        }
        long endTime2 = System.currentTimeMillis();
        System.out.println("----costTime"+(endTime2-startTime2)+"ms");
    }
```



![image-20260111214503590](.\image\image-20260111214503590.png)性能版花费时间更少



### 5.CompletableFuture常用方法

#### A. 获得结果和触发技术

![1666016337027-fe2ebf53-7ab3-4922-9975-20f7fa6a9930](.\image\1666016337027-fe2ebf53-7ab3-4922-9975-20f7fa6a9930-17681391862606.jpeg)

#### B.对计算结果进行处理


![yuque_mind](.\image\yuque_mind.jpeg)



#### C.对计算结果进行消费

![yuque_mind (2)](.\image\yuque_mind (2).jpeg)

#### D.对计算速度进行选用

applyToEither：

谁快用谁，看那个线程用的最快

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
        CompletableFuture<String> completableFutureA = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "playA";
        });

        CompletableFuture<String> completableFutureB = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "playB";
        });

        CompletableFuture<String> result = completableFutureA.applyToEither(completableFutureB, f -> {
            return f + " is winner";
        });

        System.out.println(Thread.currentThread().getName()+"\t"+"------:"+result.join());
    }
```

#### E.对计算结果进行合并

1. 两个CompletionStage任务都完成后，最终能把两个任务的结果一起交给thenCombine来处理
2. 先完成的先等着，等待其他分支任务
3. thenCombine

```java
    public static void main(String[] args) {
        CompletableFuture<Integer> completableFuture = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 20;
        });

        CompletableFuture<Integer> completableFuturex = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 20;
        });

        CompletableFuture<Integer> result = completableFuturex.thenCombine(completableFuture, (x, y) -> {
            System.out.println("---合并结果");
            return x + y;
        });
        System.out.println(result.join());
    }
```
