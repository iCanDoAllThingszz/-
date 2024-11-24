# 1. 线程基础知识

##### 1.1 线程的历史: 对CPU性能压榨的历史
- 单进程人工切换 (纸袋机, 一根纸袋一个进程)
- 多进程批处理 (一个纸袋上写5个进程, 进程间一口气串行执行)
- 多进程并行处理 (操作系统出现后, 有进程阻塞住时, 操作系统可以调度其他进程占用CPU)
- 多线程 (更细粒度的任务调度, 除了进程间的调度切换, 一个进程内部的不同任务也来回调度切换占有CPU)
- 纤程/协程: 绿色线程, 由用户管理而不是OS管理的线程

##### 1.2 什么是进程/线程?
计算机的组成部分: CPU, 内存, 系统总线bus, IO总线, 磁盘, 外接设备(USB, 显卡, 网卡 ...)

- 进程: 双击QQ.exe时, 将QQ.exe这个可执行文件加载到内存中, 此时内存中就有一个进程 QQ.exe (一个程序可以在内存中存放多份, 比如我可以同时打开登陆2个qq), 内存中的这个应用程序就是进程。 <br/>
操作系统会为进程分配资源, 比如内存, 端口号, 进程描述符PCB等。 `进程是计算机资源分配的最小单元`。
- 线程: 进程中不同的执行路径就是不同的线程, main方法开启的线程叫主线程, 多线程实际就是一套代码的多个路径并发执行。 <br/>
以QQ.exe为例, 进程加载进内存后, 实际这个进程包含了很多线程(比如监听消息的, 发送消息的 ...)。 `线程不单独占有资源, 共享进程的资源(因此就容易产生多线程间的线程安全问题)。 线程是操作系统调度执行的最小单元`。

##### 1.3 什么是线程的切换?
假设现在有两个线程T1, T2。CPU有ALU(逻辑计算单元), Registers(寄存器组), PC(程序计数器, 存储当前执行到的指令所在的内存地址)。 <br/>
1. 在程序开始执行的时候, 假设此时执行的线程T1, 指令放在PC中, 数据放在寄存器组中
2. 在进行线程切换时, T1 -> T2, 此时T1的数据还在寄存器组中, T1的指令也还在PC中; 在线程切换时会先将T1的数据和指令放在Cache中(保存T1线程上下文), 操作系统再把T2的数据和指令从Cache/内存中取出, 分别放在T2的寄存器组中, PC中。<br/>
在这个过程中, CPU不会区分T1和T2, 它只负责运算。操作系统负责线程间的切换。
3. 如果此时又要进行线程切换, T2 -> T1, 就把T2的数据和指令放在Cache中, 取出T1的数据和指令, 放入T1的寄存器组中, PC中。这就是 `Context switch (上下文切换)`, 可以看出操作系统的线程调度切换也是要消耗资源的。

##### 1.4 单核CPU使用多线程运行是否有意义?
一个核心的CPU在一个时间段内只能运行一个线程, 那么多线程运行的意义在哪里? <br/>

首先 `单核CPU上运行多线程程序是有意义的`, 比如 a线程占用CPU运行了一会后, 需要等待网络IO输入才能继续运行, 此时a线程不需要CPU, 就可以让它让出CPU, 进行线程切换, 让其他线程继续使用CPU, 达到充分使用CPU的目的(核心目标就是极限的压榨CPU资源)。

ps: 线程按照占用CPU多少可以分为: CPU密集型、IO密集型。

##### 1.5 线程数是不是越大越好?
既然单核CPU上运行多线程程序是有意义的, 那么线程数是不是越多越好呢? <br/>

不是, 线程数过多, 会导致CPU频繁切换线程, 导致CPU效率下降, 反而会降低程序性能。 

##### 1.6 线程池中的线程数量设置多少合适?
1. 先看CPU配置是几核的
2. 除去预留给其他程序的cpu, 线程数=cpu核心数即可。(无脑线程数=cpu核数是不合适的, 机器除了跑我们的应用程序, 操作系统、其他应用程序都得跑)
3. 实际工作中还是需要通过压测来找到一个比较合适的线程数

一个参考公式是: N(threads) = N(cpu) * U(cpu) * (1 + W/C) <br/>
- N(cpu)是cpu的核心数目, 可以通过 ` Runtime.getRuntime().avaliableProcessors()` 得到
- U(cpu)是期望的CPU利用率 (0-1)
- W/C 是等待时间与计算时间的比例

举例: 假设CPU只有一个核, 我们的线程, 一半时间会做计算, 另外一半时间需要等待资源, 此时线程数设置为2就比较合适。 `N = 1 * 1 * (1 + 0.5/0.5) = 2` 

一个线程做计算, 一个线程等待资源 ...

##### 1.7 怎么知道一个线程有多少时间在等待, 多少时间在计算呢?
一般需要通过性能分析工具, 比如Java的JProfiler来测算; 如果是服务器上的程序, 可以用阿里的arthas来测算。

# 2. 创建线程的5种方式

创建线程的5种方式:
1. 继承Thread类, 重写run方法, 用start()方法启动;
2. 实现Runnable接口, 重写run方法, 通过 new Thread(Runnable target).start() 启动; <br/>
使用Runnable接口创建线程相比继承Thread类创建线程是一种更好的方式, 因为避免了单继承;
3. 使用lambda表达式创建线程: new Thread(() -> { ... }).start();
4. 使用线程池创建
5. 使用Callable / FutureTask创建线程, 此方法创建的线程相比Runnable接口创建的线程, 可以获得返回值, 并且可以抛出异常。

```java
import java.util.concurrent.*;

public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        
        ExecutorService service = Executors.newCachedThreadPool();
  
        service.execute(()->{
            System.out.println("hello ThreadPool");
        });

        //带返回值的线程Callable 正常重写的run方法 public void run() 因为重写不能修改它的返回值类型 所以没有返回值
        //如果我想获得线程的返回值呢?
        //实现Callable接口 并重写其中的call方法 在实现Callable<T>接口的时候指定泛型来指定call方法的返回值类型
        //通过Future对象得到Callable的返回值 (实现了Callable的线程是异步运行的 也就是说其它线程不用阻塞等待它进行完才能执行 Callable也等待调度被执行 等Callable有返回值了 返回给Future就可以 这就是Future的含义)
        Future<String> future = service.submit(new MyCall());
        System.out.println(future.get());   //get()方法 这是一个阻塞类型的方法 在拿到Callable的返回值前会阻塞当前线程
        //ps:Java中的线程池在进行任务提交时，有两种方式：execute和submit方法。 execute只能提交Runnable类型的任务，无返回值。 submit既可以提交Runnable类型的任务，也可以提交Callable类型的任务，会有一个类型为Future的返回值，但当任务类型为Runnable时，返回值为null

        //使用FutureTask
        //不用线程池 自己启动Callable呢?
        //new Thread()参数不能直接传Callable 只能传Runnable
        //FutureTask是一个将来会产生返回值的Task,它实现了RunnableFuture接口 RunnableFuture接口又继承了Runnable,Future接口
        //所以FutureTask既能当作Runnable运行 又能像Future一样持有返回值 FutureTask既是一个Future可以接收对象 又是一个task可以被执行
        FutureTask<String> futureTask = new FutureTask<>(new MyCall());
        Thread thread = new Thread(futureTask);
        thread.start();
        System.out.println(futureTask.get()); //get()这里依旧阻塞当前线程到MyCall线程执行完拿到返回值为止

        service.shutdown();
    }
}

class MyCall implements Callable<String> {

    @Override
    public String call() throws Exception {
        System.out.println("hello Callable");
        return "success";
    }
}
```

# 3. 线程的状态

