# 并发：    
## 1.运行中的线程怎么终止
 解答链接：https://www.cnblogs.com/javastack/p/12851165.html    
###  （1）interrupt+异常法

```
public class MyThread extends Thread {
    public void run(){
        super.run();
        try {
            for(int i=0; i<500000; i++){
                if(this.interrupted()) {
                    System.out.println("线程已经终止， for循环不再执行");
                        throw new InterruptedException();
                }
                System.out.println("i="+(i+1));
            }

            System.out.println("这是for循环外面的语句，也会被执行");
        } catch (InterruptedException e) {
            System.out.println("进入MyThread.java类中的catch了。。。");
            e.printStackTrace();
        }
    }
}
使用Run.java运行的结果如下：

...
i=203798
i=203799
i=203800
线程已经终止， for循环不再执行
进入MyThread.java类中的catch了。。。
java.lang.InterruptedException
    at thread.MyThread.run(MyThread.java:13)

```
### （2）return方法

```
public class MyThread extends Thread {
    public void run(){
        while (true){
            if(this.isInterrupted()){
                System.out.println("线程被停止了！");
                return;
            }
            System.out.println("Time: " + System.currentTimeMillis());
        }
    }
}

public class Run {
    public static void main(String args[]) throws InterruptedException {
        Thread thread = new MyThread();
        thread.start();
        Thread.sleep(2000);
        thread.interrupt();
    }
}
输出结果：

...
Time: 1467072288503
Time: 1467072288503
Time: 1467072288503
线程被停止了！
不过还是建议使用“抛异常”的方法来实现线程的停止，因为在catch块中还可以将异常向上抛，使线程停止事件得以传播
```
        
## 2.说说自己是怎么使用 synchronized 关键字，在项目中用到了吗？
synchronized关键字最主要的三种使用方式：

1.**修饰实例方法**: 作用于当前对象实例加锁，进入同步代码前要获得当前对象实例的锁        
2.**修饰静态方法**: 也就是给当前类加锁，会作用于类的所有对象实例，因为静态成员不属于任何一个实例对象，是类成员（ static 表明这是该类的一个静态资源，不管new了多少个对象，只有一份）。所以如果一个线程 A 调用一个实例对象的非静态 synchronized 方法，而线程 B 需要调用这个实例对象所属类的静态 synchronized 方法，是允许的，不会发生互斥现象，**因为访问静态 synchronized 方法占用的锁是当前类的锁，而访问非静态 synchronized 方法占用的锁是当前实例对象锁**。      
3.**修饰代码块**: 指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。      

**总结**： synchronized 关键字加到 static 静态方法和 synchronized(class)代码块上都是是给 Class 类上锁。synchronized 关键字加到实例方法上是给对象实例上锁。尽量不要使用 synchronized(String a) 因为JVM中，字符串常量池具有缓存功能！


```
一个常见的面试题为例讲解一下 synchronized 关键字的具体使用。

面试中面试官经常会说：“单例模式了解吗？来给我手写一下！给我解释一下双重检验锁方式实现单例模式的原理呗！”

双重校验锁实现对象单例（线程安全）

public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public  static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
另外，需要注意 uniqueInstance 采用 volatile 关键字修饰也是很有必要。

uniqueInstance 采用 volatile 关键字修饰也是很有必要的， uniqueInstance = new Singleton(); 这段代码其实是分为三步执行：

1.为 uniqueInstance 分配内存空间
2.初始化 uniqueInstance
3.将 uniqueInstance 指向分配的内存地址
但是由于 JVM 具有指令重排的特性，执行顺序有可能变成 1->3->2。
指令重排在单线程环境下不会出现问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。
例如，线程 T1 执行了 1 和 3，此时 T2 调用 getUniqueInstance() 后发现 uniqueInstance 不为空，
因此返回 uniqueInstance，但此时 uniqueInstance 还未被初始化。

使用 volatile 可以禁止 JVM 的指令重排，保证在多线程环境下也能正常运行。
```
## 3.谈谈 synchronized和ReentrantLock 的区别
**① 两者都是可重入锁**

两者都是可重入锁。“可重入锁”概念是：自己可以再次获取自己的内部锁。比如一个线程获得了某个对象的锁，此时这个对象锁还没有释放，当其再次想要获取这个对象的锁的时候还是可以获取的，如果不可锁重入的话，就会造成死锁。同一个线程每次获取锁，锁的计数器都自增1，所以要等到锁的计数器下降为0时才能释放锁。

**② synchronized 依赖于 JVM 而 ReentrantLock 依赖于 API**

synchronized 是依赖于 JVM 实现的，前面我们也讲到了 虚拟机团队在 JDK1.6 为 synchronized 关键字进行了很多优化，但是这些优化都是在虚拟机层面实现的，并没有直接暴露给我们。ReentrantLock 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock() 方法配合 try/finally 语句块来完成），所以我们可以通过查看它的源代码，来看它是如何实现的。

**③ ReentrantLock 比 synchronized 增加了一些高级功能**

相比synchronized，ReentrantLock增加了一些高级功能。主要来说主要有三点：**①等待可中断；②可实现公平锁；③可实现选择性通知**（锁可以绑定多个条件）

1.**ReentrantLock提供了一种能够中断等待锁的线程的机制**，通过lock.lockInterruptibly()来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。      
2.**ReentrantLock可以指定是公平锁还是非公平锁**。而synchronized只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。 ReentrantLock默认情况是非公平的，可以通过 ReentrantLock类的ReentrantLock(boolean fair)构造方法来制定是否是公平的。       
3.synchronized关键字与wait()和notify()/notifyAll()方法相结合可以实现等待/通知机制，**ReentrantLock类当然也可以实现**，但是需要借助于Condition接口与newCondition() 方法。Condition是JDK1.5之后才有的，它具有很好的灵活性，比如可以实现多路通知功能也就是**在一个Lock对象中可以创建多个Condition实例（即对象监视器）**，**线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。 在使用notify()/notifyAll()方法进行通知时，被通知的线程是由 JVM 选择的，用ReentrantLock类结合Condition实例可以实现“选择性通知”** ，这个功能非常重要，而且是Condition接口默认提供的。而synchronized关键字就相当于整个Lock对象中只有一个Condition实例，所有的线程都注册在它一个身上。如果执行notifyAll()方法的话就会通知所有处于等待状态的线程这样会造成很大的效率问题，而Condition实例的signalAll()方法 只会唤醒注册在该Condition实例中的所有等待线程。
如果你想使用上述功能，那么选择ReentrantLock是一个不错的选择。

**④ 性能已不是选择标准**


## 4.Java死锁和CPU100%问题的排查技巧        
链接：https://www.cnblogs.com/aflyun/p/11141369.html
### 01死锁排查      
1. 什么是死锁？

2. 为什么会出现死锁？

3. 怎么排查代码中出现了死锁？

4. 如何避免写出死锁的代码？

作为技术人员（工程师），在出现问题的时候，能够尽快的去解决这个问题。但是在学习技术知识的时候，还是脚踏实地，多问一些为什么，一个好的问题，能够让自己思考，这方面的能力也一定要锻炼锻炼哦，这样才能更好的理解和掌握知识，并探究/触碰到更深入的地方。

1、啥是死锁？
死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。[百度百科：死锁]

注：进程和线程都可以发生死锁，只要满足死锁的条件！

2、为啥子会出现死锁？
从上面的概念中我们知道

（1）必须是两个或者两个以上进程（线程）

（2）必须有竞争资源         

3、怎么排查死锁？       
第一个姿势：**使用 jps + jstack**       
一：在windons命令窗口，使用jps -l       
二：使用 jstack -l 12316        

第二个姿势：**使用jconsole**        
在window打开 JConsole，JConsole是一个图形化的监控工具！
一：在windons命令窗口 ，输出 JConsole       
二：选择到线程的tab上       

4、如何避免死锁？       
我看了阿里巴巴中最新的开发规约，里面有对避免死锁的说明，具体如下：

【强制】对多个资源、数据库表、对象同时加锁时，**需要保持一致的加锁顺序，否则可能会 造成死锁**。 说明：线程一需要对表 A、B、C 依次全部加锁后才可以进行更新操作，那么线程二的加锁顺序也必须是 A、B、C，否则可能出现死锁。     

### 02 CPU100%排查技巧      
一、 使用top命令查看cpu占用资源较高的PID       

二、通过jps找到当前用户下的java程序PID      
执行jps -l能够打印出所有的应用的PID，找到有一个PID和这个cpu使用100%一样的ID             

三、 使用 pidstat -p < PID > 1 3 -u -t
-p：指定进程号      
-u：默认的参数，显示各个进程的cpu使用统计       
-t：显示选择任务的线程的统计信息外的额外信息
        
四、找到cpu占用较高的线程TID ，通过上图发现是 3467的TID占用cup较大

五、 因为jstack命令输出文件记录的线程ID是16进制。因此我们先将TID转换为十六进制的表示方式        

PID输出当前进程的线程信息，jstack PID  /temp/test.log       

六、查找 TID对应的线程(输出的线程id为十六进制)，找到之后具体分析这个线程在干什么，为什么会占用这么多的 CPU资源。        

## 5.Java内存泄漏排查    
链接：https://blog.csdn.net/fishinhouse/article/details/80781673        
    https://www.cnblogs.com/liululee/p/11056779.html

### 一、内存泄漏和内存溢出      
**1.1 内存溢出**        
java.lang.OutOfMemoryError，是指程序在申请内存时，没有足够的内存空间供其使用，出现OutOfMemoryError。        
**产生原因**        
产生该错误的原因主要包括：

1.JVM内存过小。       
2.程序不严密，产生了过多的垃圾。

**程序体现**        
一般情况下，在程序上的体现为：

1.内存中加载的数据量过于庞大，如一次从数据库取出过多数据。        
2.集合类中有对对象的引用，使用完后未清空，使得JVM不能回收。       
3.代码中存在死循环或循环产生过多重复的对象实体。      
4.使用的第三方软件中的BUG。       
5.启动参数内存值设定的过小。      

**错误提示**        
此错误常见的错误提示：
tomcat:java.lang.OutOfMemoryError: PermGen space        
tomcat:java.lang.OutOfMemoryError: Java heap space      
weblogic:Root cause of ServletException java.lang.OutOfMemoryError      
resin:java.lang.OutOfMemoryError        
java:java.lang.OutOfMemoryError     

**解决方法**        

增加JVM的内存大小       
优化程序，释放垃圾      
主要思路就是避免程序体现上出现的情况。避免死循环，防止一次载入太多的数据，提高程序健壮型及时释放。因此，从根本上解决Java内存溢出的唯一方法就是修改程序，及时地释放没用的对象，释放内存空间。     

**1.2内存泄漏**  

Memory Leak，是指程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存，迟早会被占光。      
在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点：        
1.首先，这些**对象是可达的**，即在有向图中，存在通路可以与其相连；      
2.其次，这些**对象是无用的**，即程序以后不会再使用这些对象。        
如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。
关于内存泄露的处理页就是提高程序的健壮型，因为内存泄漏是纯代码层面的问题。

**1.3 内存溢出和内存泄漏的联系**        

内存泄漏会最终会导致内存溢出。      
相同点：都会导致应用程序运行出现问题，性能下降或挂起。      
不同点：  
1) 内存泄漏是导致内存溢出的原因之一，内存泄漏积累起来将导致内存溢出。       
2) 内存泄漏可以通过完善代码来避免，内存溢出可以通过调整配置来减少发生频率，但无法彻底避免。

### 二、Java内存泄漏排查
**2.1确定频繁Full GC现象**          
首先通过“虚拟机进程状况工具：**jps找出正在运行的虚拟机进程，最主要是找出这个进程在本地虚拟机的唯一ID**（LVMID，Local Virtual Machine Identifier），因为在后面的排查过程中都是需要这个LVMID来确定要监控的是哪一个虚拟机进程。
同时，对于本地虚拟机进程来说，LVMID与操作系统的进程ID（PID，Process Identifier）是一致的，使用Windows的任务管理器或Unix的ps命令也可以查询到虚拟机进程的LVMID。      
jps命令格式为：
jps [ options ] [ hostid ]      
使用命令如下：      
使用**jps：jps -l**     
使用ps：ps aux | grep tomat     
        
找到你需要监控的ID（假设为20954），再利用“虚拟机统计信息监视工具：jstat”监视虚拟机各种运行状态信息。        
jstat命令格式为：       
jstat [ option vmid [interval[s|ms] [count]] ]      
使用命令如下：      
**jstat -gcutil 20954 1000**       
意思是每1000毫秒查询一次，一直查。gcutil的意思是已使用空间站总空间的百分比。

jstat执行结果       
查询结果表明：这台服务器的新生代Eden区（E，表示Eden）使用了28.30%（最后）的空间，两个Survivor区（S0、S1，表示Survivor0、Survivor1）分别是0和8.93%，老年代（O，表示Old）使用了87.33%。程序运行以来共发生Minor GC（YGC，表示Young GC）101次，总耗时1.961秒，发生Full GC（FGC，表示Full GC）7次，Full GC总耗时3.022秒，总的耗时（GCT，表示GC Time）为4.983秒。

**2.2定位到代码**       
**VisualVM**是一种工具，它提供了一个可视化界面，用于查看有关基于Java技术的应用程序运行时的详细信息。
使用VisualVM，您可以查看与本地应用程序和远程主机上运行的应用程序相关的数据。您还可以捕获有关JVM软件实例的数据，并将数据保存到本地系统。为了从Java VisualVM的所有功能中受益，您应该运行Java平台标准版（Java SE）版本6或更高版本。

**为JVM启用远程连接**       
在生产环境中，通常很难访问运行代码的实际机器。幸运的是，我们可以远程分析我们的Java应用程序。        

首先，我们需要在目标机器上授予自己JVM访问权限。为此，请使用以下内容创建名为jstatd.all.policy的文件：

grant codebase "file:${java.home}/../lib/tools.jar" {

   permission java.security.AllPermission;
   
};              

创建文件后，我们需要使用jstatd - Virtual Machine jstat Daemon工具启用与目标VM的远程连接，如下所示：        
jstatd -p <PORT_NUMBER> -J-Djava.security.policy=<PATH_TO_POLICY_FILE>      
例如：

jstatd -p 1234 -J-Djava.security.policy=D:\jstatd.all.policy        
通过在目标VM中启动jstatd，我们能够连接到目标计算机并远程分析应用程序的内存泄漏问题。

**连接到远程主机**      
在客户端计算机中，打开提示并键入jvisualvm以打开VisualVM工具。

使用Java VisualVM，我们可以**对Java Heap进行内存监视**，并确定其行为是否存在内存泄漏。      
**仅仅30秒之后，老年代几乎已满，表明即使使用Full GC，老年代也在不断增长，这是内存泄漏的明显迹象。**

检测此泄漏原因的一种方法如下图所示（单击放大），使用**带有heapdump的Java VisualVM生成**。在这里，我们看到50％的Hashtable $ Entry对象在堆中，而第二行指向MemLeak类。因此，内存泄漏是由MemLeak类中使用的哈希表引起的。      

最后，在OutOfMemoryError之后观察Java Heap，其中Young和Old代完全填满。

## 6.多线程共享数据的方式       
链接：https://www.cnblogs.com/liaokailin/p/3786320.html     
以火车售票的共享数据为例：      
 1.案例分析-01

通过代码实现火车票出售的例子

在实现代码之前先对问题进行分析：火车票出售应该是在多个窗口进行的（即多个线程），以一个车的班次来说，该班次的火车票张数即为多个窗口共享的数据

即**这份共享数据为出售特定班次的火车票，这个动作在多个窗口都是不变的，变更的只有火车票的剩余张数**.


```
package org.lkl.thead;

/**
 * 
 * Function : 多线程共享数据
 * 
 * @author : Liaokailin CreateDate : 2014-6-13 version : 1.0
 */
public class MultiThreadShareData {
    public static void main(String[] args) {
        SellTicket s = new SellTicket(); // 共享对象
        new Thread(s, "1").start();
        new Thread(s, "2").start();
        new Thread(s, "3").start();
        new Thread(s, "4").start();
        new Thread(s, "5").start();
    }

}

/**
 * 出售火车票  
 */
class SellTicket implements Runnable {
    private int ticket = 100;

    @Override
    public void run() {

        while (true) {
            synchronized (SellTicket.class) {
            if (ticket > 0) {
            
                    try {
                        Thread.sleep(30);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    ticket-- ;
                    System.out.println("窗口" + Thread.currentThread().getName()
                            + "出售合肥到北京火车票剩余车票" + ticket);
                }else{
                    break ;
                }
            }
        }

    }
}
```

通过上面的代码可以发现，多线程共享的数据是卖火车票，而这个动作在各个线程中都是不变的，因此可以做如下的小结：        

**多线程中各个线程执行的代码相同，那么可以使用同一个Runnable对象来实现共享数据。**

2. 案例分析-02

和上面问题乡对应的是 **如果多个线程中执行的代码不同呢？** 该如何去处理数据之间的共享？

例如： 

设计4个线程，其中两个线程每次对j增加1，另外两个线程对j每次减少1 。
通过问题可以发现，线程中执行的代码显示是不同的，一个是j++ 一个是j--，在前面我们说过 **要同步互斥的代码或者算法应该分别放在独立的方法中，并且这些方法都放在同一个类中，这样子比较容易实现他们之间的同步互斥和通信**


```
class Business{
    private int j = 0;
    
    public synchronized void increment(){
        j++ ;
        System.out.println(Thread.currentThread().getName()+",j="+j);
    }
    public synchronized void decrement(){
        j-- ;
        System.out.println(Thread.currentThread().getName()+",j="+j);
    }
}
```

要实现多个线程的不同操作，可以将不同的操作防止在不同的Runnable中 然后将不同的Runnable对象传递给不同的线程


```
package org.lkl.thead;


/**
 *   设计4个线程，其中两个线程每次对j增加1，另外两个线程对j每次减少1 
 */
public class ShareDataFoo {
    public static void main(String[] args) {
        Business b = new Business() ;
        for(int i= 0 ;i<2;i++){
            new Thread(new OperatorFoo01(b)).start();
            new Thread(new OperatorFoo02(b)).start();
            
        }
        
    }

    
}
class OperatorFoo01 implements Runnable{
    private Business business;
    public OperatorFoo01(Business business){
        this.business = business ;
    }
    @Override
    public void run() {
        business.increment() ;
    }
}

class OperatorFoo02 implements Runnable{
    private Business business;
    public OperatorFoo02(Business business){
        this.business = business ;
    }
    @Override
    public void run() {
        business.decrement() ;
    }
}

class Business{
    private int j = 0;
    
    public synchronized void increment(){
        j++ ;
        System.out.println(Thread.currentThread().getName()+",j="+j);
    }
    public synchronized void decrement(){
        j-- ;
        System.out.println(Thread.currentThread().getName()+",j="+j);
    }
}

结果：
Thread-0,j=1
Thread-1,j=0
Thread-2,j=1
Thread-3,j=0
```

如果每个线程执行的代码不同，这时候需要不同的Runnable对象，有如下两种方式来实现这些Runnable对象之间的数据共享

1.**将共享数据封在在一个对象中，然后将这个对象逐一传递给各个不同Runnable对象。每个线程对共享数据的操作方法也分配到那个对象身上去完成，这样容易实现针对该数据进行的各个操作的互斥和通行** （上面我们的例子就是采用这样的方式）

2.**将这些Runnable对象作为某个类的内部类，共享数据作为这个外部类中的成员变量，每个线程对共享数据的操作也分配给外部类，以便对共享数据进行的各个操作的互斥和通信，作为内部类的各个Runnable对象调用外部类的这些方法** 比如下面的例子：       

```
package org.lkl.thead;


/**
 *   设计4个线程，其中两个线程每次对j增加1，另外两个线程对j每次减少1 
 */
public class ShareDataFoo2 {
    public static void main(String[] args) {
        ShareDataFoo2 foo = new ShareDataFoo2() ;
        
        for(int i= 0 ;i<2;i++){
            new Thread(foo.new OperAdd()).start();
            new Thread(foo.new OperSub()).start();
            
        }
        
    }
    
    private int j = 0 ;
    
    public synchronized  void increment(){
        j++ ;
        System.out.println(Thread.currentThread().getName()+",j="+j);
    }
    
    public synchronized void decrement(){
        j-- ;
        System.out.println(Thread.currentThread().getName()+",j="+j);
    }

     class OperAdd implements Runnable{
        @Override
        public void run() {
            increment() ;
        }
    }
    
     class OperSub implements Runnable{
        @Override
        public void run() {
            decrement() ;
        }
    }
}
```

## 7.线程八锁       

链接：https://www.cnblogs.com/shamao/p/11045282.html      

即多线程的八个案例：            

**（1）两个线程调用同一个对象的两个同步方法**       

```
public class Demo {
    public static void main(String[] args) {
        Number number = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getTwo();
            }
        }).start();
    }
}

class Number {
    public synchronized void getOne() {
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}

运行结果：
1 one
2 two
```
结果分析：

被synchronized修饰的方法，锁的对象是方法的调用者。因为两个方法的调用者是同一个，所以两个方法用的是同一个锁，先调用方法的先执行。        

**（2）新增Thread.sleep()给某个方法**       

```
public class Demo {
    public static void main(String[] args) {
        Number number = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getTwo();
            }
        }).start();
    }
}

class Number {
    public synchronized void getOne() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}  

运行结果如下：

1 // 等待一秒。
2 one
3 two

```
结果说明：

被synchronized修饰的方法，锁的对象是方法的调用者。因为两个方法的调用者是同一个，所以两个方法用的是同一个锁，先调用方法的先执行，第二个方法只有在第一个方法执行完释放锁之后才能执行。               

**（3）新增一个线程调用新增的一个普通方法**     

```
public class Demo {
    public static void main(String[] args) {
        Number number = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getTwo();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getThree();
            }
        }).start();
    }
}

class Number {
    public synchronized void getOne() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }

    public void getThree() {
        System.out.println("three");
    }
}

运行结果如下：

1 three
2 // 等待一秒。
3 one
4 two
```
结果说明：

新增的方法没有被synchronized修饰，不是同步方法，不受锁的影响，所以不需要等待。其他线程共用了一把锁，所以还需要等待。      

**（4）两个线程调用两个对象的同步方法，其中一个方法有Thread.sleep()**      

```
public class Demo {
    public static void main(String[] args) {
        Number numberOne = new Number();
        Number numberTwo = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                numberOne.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                numberTwo.getTwo();
            }
        }).start();
    }
}

class Number {
    public synchronized void getOne() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}

运行结果如下：

1 two
2 // 等待一秒。
3 one

```
结果说明：

被synchronized修饰的方法，锁的对象是方法的调用者。因为用了两个对象调用各自的方法，所以两个方法的调用者不是同一个，所以两个方法用的不是同一个锁，后调用的方法不需要等待先调用的方法。        

**（5）将有Thread.sleep()的方法设置为static方法，并且让两个线程用同一个对象调用两个方法**      

```
public class Demo {
    public static void main(String[] args) {
        Number number = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getTwo();
            }
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}

运行结果如下：

1 two
2 // 等待一秒。
3 one

```
结果说明：

被synchronized和static修饰的方法，锁的对象是类的class对象。仅仅被synchronized修饰的方法，锁的对象是方法的调用者。因为两个方法锁的对象不是同一个，所以两个方法用的不是同一个锁，后调用的方法不需要等待先调用的方法。        

**（6）将两个方法均设置为static方法，并且让两个线程用同一个对象调用两个方法**      


```
public class Demo {
    public static void main(String[] args) {
        Number number = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                number.getTwo();
            }
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("one");
    }

    public static synchronized void getTwo() {
        System.out.println("two");
    }
}

运行结果如下：

1 // 等待一秒。
2 one
3 two
```
结果说明：

被synchronized和static修饰的方法，锁的对象是类的class对象。因为两个同步方法都被static修饰了，所以两个方法用的是同一个锁，后调用的方法需要等待先调用的方法。      

**（7）将两个方法中有Thread.sleep()的方法设置为static方法，另一个方法去掉static修饰，让两个线程用两个对象调用两个方法**        


```
public class Demo {
    public static void main(String[] args) {
        Number numberOne = new Number();
        Number numberTwo = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                numberOne.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                numberTwo.getTwo();
            }
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}

运行结果如下：

1 two
2 // 等待一秒。
3 one
```
结果说明：

被synchronized和static修饰的方法，锁的对象是类的class对象。仅仅被synchronized修饰的方法，锁的对象是方法的调用者。即便是用同一个对象调用两个方法，锁的对象也不是同一个，所以两个方法用的不是同一个锁，后调用的方法不需要等待先调用的方法。      

**（8）将两个方法均设置为static方法，并且让两个线程用同一个对象调用两个方法**      

```
public class Demo {
    public static void main(String[] args) {
        Number numberOne = new Number();
        Number numberTwo = new Number();

        new Thread(new Runnable() {
            @Override
            public void run() {
                numberOne.getOne();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                numberTwo.getTwo();
            }
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("one");
    }

    public static synchronized void getTwo() {
        System.out.println("two");
    }
}

运行结果如下：

1 // 等待一秒。
2 one
3 two
```
结果说明：

被synchronized和static修饰的方法，锁的对象是类的class对象。因为两个同步方法都被static修饰了，即便用了两个不同的对象调用方法，两个方法用的还是同一个锁，后调用的方法需要等待先调用的方法。       

总结        

**一个类里面如果有多个synchronized方法，在使用同一个对象调用的前提下，某一个时刻内，只要一个线程去调用其中的一个synchronized方法了，其他的线程都只能等待**，换句话说，某一时刻内，只能有唯一一个线程去访问这些synchronized方法。

**锁的是当前对象this，被锁定后，其他线程都不能进入到当前对象的其他的synchronized方法。**

加个普通方法后发现和同步锁无关。

换成**静态同步方法后，情况又变化**。

所有的**非静态同步方法用的都是同一把锁：实例对象本身**。

也就是说如果一个对象的非静态同步方法获取锁后，该对象的其他非静态同步方法必须等待获取锁的方法释放锁后才能获取锁，可是其他对象的非静态同步方法因为跟该对象的非静态同步方法用的是不同的锁，所以毋须等待该对象的非静态同步方法释放锁就可以获取他们自己的锁。

所有的**静态同步方法用的也是同一把锁：类对象本身**。

这两把锁是两个不同的对象，所以**静态同步方法与非静态同步方法之间不会有竞争条件。但是一旦一个静态同步方法获取锁后，其他的静态同步方法都必须等待该方法释放锁后才能获取锁**，而不管是同一个对象的静态同步方法，还是其他对象的静态同步方法，只要它们属于同一个类的对象，那么就需要等待当前正在执行的静态同步方法释放锁。


## 8.JAVA线程间通信方式     
链接：https://blog.csdn.net/wlddhj/article/details/83793709     https://www.cnblogs.com/hapjin/p/5492619.html      
**一、Synchronized、wait、notify**     

```
public class Demo1 {

    private final List<Integer> list =new ArrayList<>();

    public static void main(String[] args) {
        Demo1 demo =new Demo1();
        new Thread(()->{
            for (int i=0;i<10;i++){
                synchronized (demo.list){
                    if(demo.list.size()%2==1){
                        try {
                            demo.list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    demo.list.add(i);
                    System.out.print(Thread.currentThread().getName());
                    System.out.println(demo.list);
                    demo.list.notify();
                }
            }

        }).start();

        new Thread(()->{
            for (int i=0;i<10;i++){
                synchronized (demo.list){
                    if(demo.list.size()%2==0){
                        try {
                            demo.list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    demo.list.add(i);
                    System.out.print(Thread.currentThread().getName());
                    System.out.println(demo.list);
                    demo.list.notify();
                }
            }
        }).start();
    }
}

```
**Lock、Condition：**       

```
public class Task {
    private final Lock lock = new ReentrantLock();

    private final Condition addConditon = lock.newCondition();
    private final Condition subConditon = lock.newCondition();

    private volatile int num = 0;
    private List<String> list = new ArrayList<>();

    public void add() {
        for (int i = 0; i < 10; i++) {
            lock.lock();

            try {
                if (list.size() == 10) {
                    addConditon.await();
                }
                num++;
                Thread.sleep(100);
                list.add("add " + num);
                System.out.println("The list size is " + list.size());
                System.out.println("The add thread is " + Thread.currentThread().getName());
                System.out.println("-------------");
                subConditon.signal();

            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }

    public void sub() {
        for (int i = 0; i < 10; i++) {
            lock.lock();

            try {

                if (list.size() == 0) {
                    subConditon.await();
                }
                num--;
                Thread.sleep(100);
                list.remove(0);
                System.out.println("The list size is " + list.size());
                System.out.println("The sub thread is " + Thread.currentThread().getName());
                System.out.println("-------------");
                addConditon.signal();

            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }


    public static void main(String[] args) {
        Task task = new Task();
        new Thread(task::add).start();
        new Thread(task::sub).start();
    }
}

```
**二、PipedInputStream、PipedOutputStream**     
这里用流在两个线程间通信，但是Java中的Stream是单向的，所以在两个线程中分别建了一个input和output。

```
public class PipedDemo {

    private final PipedInputStream inputStream1;
    private final PipedOutputStream outputStream1;
    private final PipedInputStream inputStream2;
    private final PipedOutputStream outputStream2;

    public PipedDemo(){
        inputStream1 = new PipedInputStream();
        outputStream1 = new PipedOutputStream();
        inputStream2 = new PipedInputStream();
        outputStream2 = new PipedOutputStream();
        try {
            inputStream1.connect(outputStream2);
            inputStream2.connect(outputStream1);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
    /**程序退出时，需要关闭stream*/
    public void shutdown() throws IOException {
        inputStream1.close();
        inputStream2.close();
        outputStream1.close();
        outputStream2.close();
    }


    public static void main(String[] args) throws IOException {
        PipedDemo demo =new PipedDemo();
        new Thread(()->{
            PipedInputStream in = demo.inputStream2;
            PipedOutputStream out = demo.outputStream2;

            for (int i = 0; i < 10; i++) {
                try {
                    byte[] inArr = new byte[2];
                    in.read(inArr);
                    System.out.print(Thread.currentThread().getName()+": "+i+" ");
                    System.out.println(new String(inArr));
                    while(true){
                        if("go".equals(new String(inArr)))
                            break;
                    }
                    out.write("ok".getBytes());
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        }).start();

        new Thread(()->{
            PipedInputStream in = demo.inputStream1;
            PipedOutputStream out = demo.outputStream1;

            for (int i = 0; i < 10; i++) {
                try {
                    out.write("go".getBytes());
                    byte[] inArr = new byte[2];
                    in.read(inArr);
                    System.out.print(Thread.currentThread().getName()+": "+i+" ");
                    System.out.println(new String(inArr));
                    while(true){
                        if("ok".equals(new String(inArr)))
                            break;
                    }

                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        }).start();
//        demo.shutdown();
    }
}

```
**三、BlockingQueue**       

```
public class BlockingQueueDemo {

    public static void main(String[] args) {
        LinkedBlockingQueue<String> queue = new LinkedBlockingQueue<>();
        //读线程
        new Thread(() -> {
            int i =0;
            while (true) {
                try {
                    String item = queue.take();
                    System.out.print(Thread.currentThread().getName() + ": " + i + " ");
                    System.out.println(item);
                    i++;
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
        //写线程
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    String item = "go"+i;
                    System.out.print(Thread.currentThread().getName() + ": " + i + " ");
                    System.out.println(item);
                    queue.put(item);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}

```


## 9.线程创建的三种方式及区别       
链接：https://www.cnblogs.com/htyj/p/10848646.html      

* 继承Thread类      

**1**)定义Thread类的子类，并重写该类的run方法，该run方法的方法体就代表了线程要完成的任务。因此把run()方法称为执行体。       

**2**)创建Thread子类的实例，即创建了线程对象。       

**3**)调用线程对象的start()方法来启动该线程。        

```
package Thread;

import java.util.concurrent.*;

public class TestThread {
    public static void main(String[] args) throws Exception {
        testExtends();
    }

    public static void testExtends() throws Exception {
        Thread t1 = new MyThreadExtends();
        Thread t2 = new MyThreadExtends();
        t1.start();
        t2.start();
    }
}

class MyThreadExtends extends Thread {
    @Override
    public void run() {
        System.out.println("通过继承Thread，线程号:" + currentThread().getName());
    }
}
```

* 实现Runnable接口      

**1**)定义runnable接口的实现类，并重写该接口的run()方法，该run()方法的方法体同样是该线程的线程执行体。      

**2**)创建 Runnable实现类的实例，并依此实例作为Thread的target来创建Thread对象，该Thread对象才是真正的线程对象。        

**3**)调用线程对象的start()方法来启动该线程。


```
package Thread;

import java.util.concurrent.*;
//测试类
public class TestThread {
    public static void main(String[] args) throws Exception {
         testImplents();
    }

    public static void testImplents() throws Exception {
        MyThreadImplements myThreadImplements = new MyThreadImplements();
        Thread t1 = new Thread(myThreadImplements);
        Thread t2 = new Thread(myThreadImplements, "my thread -2");
        t1.start();
        t2.start();
    }
}
//线程类
class MyThreadImplements implements Runnable {
    @Override
    public void run() {
        System.out.println("通过实现Runable，线程号:" + Thread.currentThread().getName());
    }
}

```

* 实现Callable接口      

**1**)创建Callable接口的实现类，并实现call()方法，该call()方法将作为线程执行体，并且有返回值。

**2**)创建Callable实现类的实例，使用FutureTask类来包装Callable对象，该FutureTask对象封装了该Callable对象的call()方法的返回值。

**3**)使用FutureTask对象作为Thread对象的target创建并启动新线程。

**4**)调用FutureTask对象的get()方法来获得子线程执行结束后的返回值


```
package Thread;

import java.util.concurrent.*;

public class TestThread {
    public static void main(String[] args) throws Exception {
        testCallable();
    }

    public static void testCallable() throws Exception {
        Callable callable = new MyThreadCallable();
        FutureTask task = new FutureTask(callable);
        new Thread(task).start();
        System.out.println(task.get());
        Thread.sleep(10);//等待线程执行结束
        //task.get() 获取call()的返回值。若调用时call()方法未返回，则阻塞线程等待返回值
        //get的传入参数为等待时间，超时抛出超时异常；传入参数为空时，则不设超时，一直等待
        System.out.println(task.get(100L, TimeUnit.MILLISECONDS));
    }
}

class MyThreadCallable implements Callable {

    @Override
    public Object call() throws Exception {
        System.out.println("通过实现Callable，线程号:" + Thread.currentThread().getName());
        return 10;
    }
}
```
**三种方式的优缺点**　      

**采用继承Thread类方式**:      

（1）**优点：编写简单，如果需要访问当前线程，无需使用Thread.currentThread()方法，直接使用this，即可获得当前线程**。        

（2）**缺点：因为线程类已经继承了Thread类，所以不能再继承其他的父类**。

**采用实现Runnable接口方式**：      

（1）**优点：线程类只是实现了Runable接口，还可以继承其他的类**。在这种方式下，可以多个线程共享同一个目标对象，所以非常适合多个相同线程来处理同一份资源的情况，从而可以将CPU代码和数据分开，形成清晰的模型，较好地体现了面向对象的思想。        

（2）**缺点：编程稍微复杂，如果需要访问当前线程，必须使用Thread.currentThread()方法**。

**Runnable和Callable的区别**：      

(1)**Callable规定的方法是call(),Runnable规定的方法是run()**.        

(2)**Callable的任务执行后可返回值，而Runnable的任务是不能返回值得**     
(3)**call方法可以抛出异常，run方法不可以**，因为run方法本身没有抛出异常，所以自定义的线程类在重写run的时候也无法抛出异常       

(4)**运行Callable任务可以拿到一个Future对象，表示异步计算的结果**。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。

**start（）和run（）的区别**:       

* start()方法用来，开启线程，但是线程开启后并没有立即执行，他需要获取cpu的执行权才可以执行     

* **run()方法是由jvm创建完本地操作系统级线程后回调的方法，不可以手动调用**（否则就是普通方法）

## 10.LinkedBlockingQueue和ArrayBlockingQueue区别       
链接：https://www.cnblogs.com/silyvin/p/9106628.html        

* 队列大小有所不同，**ArrayBlockingQueue是有界的初始化必须指定大小，而LinkedBlockingQueue可以是有界的也可以是无界的**(Integer.MAX_VALUE)，（而且不会初始化就占用一大片内存）对于后者而言，当添加速度大于移除速度时，在无界的情况下，可能会造成内存溢出等问题。

* 数据存储容器不同，**ArrayBlockingQueue采用的是数组作为数据存储容器，而LinkedBlockingQueue采用的则是以Node节点作为连接对象的链表**。

* 由于**ArrayBlockingQueue采用的是数组的存储容器，因此在插入或删除元素时不会产生或销毁任何额外的对象实例，而LinkedBlockingQueue则会生成一个额外的Node对象**。这可能在**长时间内需要高效并发地处理大批量数据的时，对于GC可能存在较大影响**。

* 两者的实现队列添加或移除的锁不一样，**ArrayBlockingQueue实现的队列中的锁是没有分离的，即添加操作和移除操作采用的同一个ReenterLock锁，而LinkedBlockingQueue实现的队列中的锁是分离的，其添加采用的是putLock，移除采用的则是takeLock，这样能大大提高队列的吞吐量**，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。












# JVM

## 1.Class.forName()和ClassLoader.loadClass区别     

1).Class.forName(className)方法，内部实际调用的方法是  **Class.forName(className,true,classloader)**;

第2个boolean参数表示类是否需要初始化，  **Class.forName(className)默认是需要初始化**。

一旦初始化，就会触发目标对象的 static块代码执行，static参数也也会被再次初始化。


2).ClassLoader.loadClass(className)方法，内部实际调用的方法是  **ClassLoader.loadClass(className,false)**;

第2个 boolean参数，表示目标对象是否进行链接，**false表示不进行链接**，由上面介绍可以，

**不进行链接意味着不进行包括初始化等一些列步骤，那么静态块和静态对象就不会得到执行**






# Java基础
## 1.Java8新特性
链接：https://www.cnblogs.com/xichji/p/11570387.html

### (1)Lambda 表达式和函数式接口

在 Java8 以前，我们想要让一个方法可以与用户进行交互，比如说使用方法内的局部变量；这时候就只能使用接口做为参数，让用户实现这个接口或使用匿名内部类的形式，把局部变量通过接口方法传给用户。

传统匿名内部类缺点：代码臃肿，难以阅读

Lambda 表达式
Lambda 表达式将函数当成参数传递给某个方法，或者把代码本身当作数据处理；

语法格式：

用逗号分隔的参数列表
-> 符号
和 语句块 组成


```
Arrays.asList( "a", "b", "d" ).forEach( e -> System.out.println( e ) );
等价于

List<String> list = Arrays.asList( "a", "b", "d" );
for(String e:list){
    System.out.println(e);
}
如果语句块比较复杂，使用 {} 包起来

Arrays.asList( "a", "b", "d" ).forEach( e -> {
    String m = "9420 "+e;
    System.out.print( m );
});
Lambda 本质上是匿名内部类的改装，所以它使用到的变量都会隐式的转成 final 的

String separator = ",";
Arrays.asList( "a", "b", "d" ).forEach( 
    e -> System.out.print( e + separator ) );
等价于

final String separator = ",";
Arrays.asList( "a", "b", "d" ).forEach( 
    e -> System.out.print( e + separator ) );
Lambda 的返回值和参数类型由编译器推理得出，不需要显示定义，如果只有一行代码可以不写 return 语句

Arrays.asList( "a", "b", "d" ).sort( ( e1, e2 ) -> e1.compareTo( e2 ) );
等价于

List<String> list = Arrays.asList("a", "b", "c");
Collections.sort(list, new Comparator<String>() {
    @Override
    public int compare(String o1, String o2) {
        return o1.compareTo(o2);
    }
});
```
### （2）JVM新特性

使用Metaspace（JEP 122）代替持久代（PermGen space）。在JVM参数方面，使用-XX:MetaSpaceSize和-XX:MaxMetaspaceSize代替原来的-XX:PermSize和-XX:MaxPermSize

## 2.接口和抽象类区别

1.接口的方法默认是 public                         abstract，所有方法在接口中不能有实现（Java 8开始接口方法可以有默认实现），抽象类可以有非抽象的方法；        
2.接口中的实例变量默认是 final 类型的，而抽象类中则不一定；     
3.一个类可以实现多个接口，但最多只能实现一个抽象类；        
4.一个类实现接口的话要实现接口的所有方法，而抽象类不一定；      
5.接口不能用 new 实例化，但可以声明，但是必须引用一个实现该接口的对象；        
6.抽象类里可以有构造方法，而接口内不能有构造方法；      
7.从设计层面来说，抽象是对类的抽象，是一种模板设计，接口是行为的抽象，是一种行为的规范。

## 3.Java如何实现继承多个类的多个方法？
链接：https://www.cnblogs.com/gzhbk/p/11512559.html

使用内部类就可以多继承，严格来说，还不是实现多继承，但是这种方法可以实现多继承所需的功能，所以把它称为实现了多继承。


```
下面就举个例子：
假如有一个打电话类Call，里面实现了一个可以打电话的功能的方法callSomebody(String phoneNum);
一个发信息类SendMessage，里面实现了一个可以发信息功能的方法sendToSomebody(String phoneNum);
还有一个手机类Phone，这个手机类想实现打电话和发信息的功能;我们知道可以用继承来获得父类的方法，
但是只可以单继承呀，也就是说只可以实现其中一个类里面的方法，这并不满足我们的需求。
接下来，我们就使用内部类，达到我们所需的目标了。

public class Call {
    
    public void callSomebody(String phoneNum){
        System.out.println("我在打电话喔，呼叫的号码是：" + phoneNum);
    }

}


class SendMessage {
   public void sendToSomebody(String phoneNum){
       System.out.println("我在发短信喔，发送给 ：" + phoneNum);
  }
}


public class Phone {
private class MyCall extends Call{

}
private class MySendMessage extends SendMessage{

}

private MyCall call = new MyCall();
private MySendMessage send = new MySendMessage();

public void phoneCall(String phoneNum){
call.callSomebody(phoneNum);
}

public void phoneSend(String phoneNum){
send.sendToSomebody(phoneNum);
}

public static void main(String[] args) {
Phone phone = new Phone();
phone.phoneCall("110");
phone.phoneSend("119");
}
}
```
## 4.default关键字
链接：https://blog.csdn.net/SnailMann/article/details/80231593

两种用法：      
1.在switch语句的时候使用default  

```
		int day = 8;
		String dayString;
		switch (day) {
			case 1:	dayString = "Monday";
					break;
			case 2: dayString = "Tuesday";
					break;
			case 3: dayString = "Wednesday";
					break;
			case 4: dayString = "Thursday";
					break;
			case 5: dayString = "Friday";
					break;
			case 6: dayString = "Saturday";
					break;
			//如果case没有匹配的值，那么肯定是星期日
			default: dayString = "Sunday";
					 break;
		}
		System.out.println(dayString);

总结：
使用比较简单，就是当case里的值与switch里的key没有匹配的时候，执行default里的方法。
在这里的例子中就是key为8，所以key与所有的case的值都不匹配，所以输出星期天Sunday.
```

2.在定义接口的时候使用default来修饰具体的方法

```
public interface IntefercaeDemo {

	//具体方法
	default void showDefault(){
		System.out.println("this is showDefault method");
	}
	static void showStatic(){
		System.out.println("this is showStatic method");
	}
	
	//没有实现的抽象方法
	void sayHi();
}


public class LearnDefault implements IntefercaeDemo{
	//实现抽象方法
	@Override
	public void sayHi() {
		System.out.println("this is sayHi mehtod");
	}
	
	public static void main(String[] args) {
		//接口中被static所修饰的具体方法
		IntefercaeDemo.showStatic();
		
		//将实现了IntefercaeDemo的类实例化
		LearnDefault learnDefault = new LearnDefault();
		//被Default所修饰的具体方法可以通过引用变量来调用
		learnDefault.showDefault();

	}
}

```
说明：      
JDK1.8中为了加强接口的能力，使得接口可以存在具体的方法，前提是方法需要被default或static关键字所修饰。

总结：      
1.default修饰的目的是让接口可以拥有具体的方法，让接口内部包含了一些默认的方法实现。        
2.被default修饰的方法是接口的默认方法。既只要实现该接口的类，都具有这么一个默认方法，默认方法也可以被重写。        
3.我们可以想象这么一个场景，既实现某个接口的类都具有某个同样的功能，如果像Java8以前的版本，那么每个实现类都需要写一段重复的代码去实现那个功能，显得没有必要。这就是存在的意义。

注意：      
另外这里既然提到了接口的修饰符default，那么就要注意一点，如果一个类实现了两个接口（可以看做是“多继承”），这两个接口又同时都包含了一个名字相同的default方法，那么会发生什么情况？ 在这样的情况下，编译器是会报错，得到一个编译器错误，因为编译器不知道应该在两个同名的default方法中选择哪一个，因此产生了二义性。

容易混淆地方：      

```
关键字default不要跟平时我们在类中定义方法时，没有加任何访问修饰符时的(default)相混淆，它们的意义是不一样的
public class Demo{
	//没有访问修饰符修饰，所以默认为(default)
	String name;
	void show(){}
}
这里的(default)指的是一种场景，既类中成员没有被访问修饰符修饰，所以属于(default)的情况，效果是(default)情况的成员只能在Demo类所在的包内被访问。
```
    
## 5. char可以存一个汉字吗？string存中文呢？     
java采用unicode，2个字节（16位）来表示一个字符，无论是汉字还是数字字母，或其他语言。char 在java中是2个字节。所以可以存储中文！
string存储一句汉语，其长度与存字符长度相同。        

## 6.throw与throws的区别？
链接：https://blog.csdn.net/xsj_blog/article/details/83030450
```
throw：
throw是语句抛出一个异常，一般是在代码块的内部，
当程序出现某种逻辑错误时由程序员主动抛出某种特定类型的异常
public static void main(String[] args) { 
    String s = "abc"; 
    if(s.equals("abc")) { 
      throw new NumberFormatException(); 
    } else { 
      System.out.println(s); 
    } 
    //function(); 
}

运行结果：Exception in thread "main" java.lang.NumberFormatException at......

throws：
当某个方法可能会抛出某种异常时用于throws 声明可能抛出的异常，然后交给上层调用它的方法程序处理

public class testThrows(){
public static void function() throws NumberFormatException { 
	String s = "abc"; 
	System.out.println(Double.parseDouble(s)); 
} 
 
public static void main(String[] args) { 
	try { 
		function(); 
	} catch (NumberFormatException e) { 
		System.err.println("非数据类型不能强制类型转换。"); 
		//e.printStackTrace(); 
	} 
}

运行结果：非数据类型不能强制类型转换。

```
throw与throws的比较

1.throws出现在方法函数头；而throw出现在函数体。     
2.throws表示出现异常的一种可能性，并不一定会发生这些异常；throw   则是抛出了异常，执行throw则一定抛出了某种异常对象。     
3.两者都是消极处理异常的方式（这里的消极并不是说这种方式不好），只是抛出或者可能抛出异常，但是不会由函数去处理异常，真正的处理异常由函数的上层调用处理。


## 7.内存泄漏与ThreadLocal
链接：https://mp.weixin.qq.com/s/XvTV3VuEn94i9ApJFPA2hA     
ThreadLocal的key是弱引用，那么在 threadLocal.get() 的时候,发生GC之后，key是否为null？      
Thread类有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals，也就是说每个线程有一个自己的ThreadLocalMap。

ThreadLocalMap有自己的独立实现，可以简单地将它的key视作ThreadLocal，value为代码中放入的值（实际上key并不是ThreadLocal本身，而是它的一个弱引用）。

每个线程在往ThreadLocal里放值的时候，都会往自己的ThreadLocalMap里存，读也是以ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。

ThreadLocalMap有点类似HashMap的结构，只是HashMap是由数组+链表实现的，而ThreadLocalMap中并没有链表结构。

我们还要注意Entry， 它的key是ThreadLocal<?> k ，继承自WeakReference， 也就是我们常说的弱引用类型。        
**GC之后key是否为null？**       
回应开头的那个问题， ThreadLocal 的key是弱引用，那么在threadLocal.get()的时候,发生GC之后，key是否是null？        
这个问题刚开始看，如果没有过多思考，弱引用，还有垃圾回收，那么肯定会觉得是null。

其实是不对的，因为题目说的是在做 threadlocal.get() 操作，证明其实还是有强引用存在的，所以 key 并不为 null，如下图所示，ThreadLocal的强引用仍然是存在的。     
如果我们的强引用不存在的话，那么 key 就会被回收，也就是会出现我们 value 没被回收，key 被回收，导致 value 永远存在，出现内存泄漏。     


## 8.final修饰变量和类
**修饰变量：**      
1.基本数据类型的变量：数值一旦在初始化之后便不能更改。      
2.引用类型的变量：在对其初始化之后便不能再让其指向另一个对象，但引用的对象内容可变。  
**修饰方法：**      
1.不能被继承，不能被子类修改。      
**使用 final 方法的原因**：
1.把方法锁定，以防任何继承类修改它的含义；      
2.final 方法效率高，因为在早期的Java实现版本中，会将 final 方法转为内嵌调用，这样就节省了方法调用的开销。但是如果方法过于庞大，可能看不到内嵌调用带来的任何性能提升（现在的 Java 版本已经不需要使用 final 方法进行这些优化了）。类中所有的 private 方法都隐式地指定为 final。      
**修饰类：** 不能被继承，final 类中的所有成员方法都会被隐式地指定为final方法。      
**修饰形参：** final 形参不可变。       
final 一定程度上可以**用来防止指令的重排序**，比如：        

```
public class Holder {
    private int n;
    // private final int n; // 这句可以解决这个问题

    public Holder(int n) {
        // 一个线程执行到这，另一个线程进来了，这时候虽然对象还没有构造好，但另一个线程已经拿到对象了
        this.n = n;
    }

    public void assertSanity() {
        // 会成立原因：某个线程在n != n时，第一次读取到 n 是一个失效值，
        // 第二次读取到的是更新后的值，就抛异常了。
        if (n != n) { 
            throw new AssertionError("n != n");
        }
    }
}
这个问题发生的主要原因就是对对象初始引用的调用被重排序排到了构造函数前面，用了 final 修饰需要在构造函数中修饰的变量可以解决这个问题，因为：

1.对于含有 final 域的对象，JVM 必须保证对象的初始引用在构造函数之后执行，不能乱序执行；
2.也就是说，一旦得到了对象的引用，那么这个对象的 final 域一定是已经完成了初始化的。
```     
## 9.static修饰变量和类     
**修饰变量：** 静态变量随着类加载时被完成初始化，内存中只有一个，且 JVM 也只会为它分配一次内存，该类的所有实例共享静态变量。        
**修饰方法：** 在类加载的时候就存在，不依赖任何实例；static 方法必须实现，不能用 abstract 修饰。     
**修饰代码块：** 在类加载过程中的初始化阶段会执行静态代码块中的内容，静态代码中的内容会和 static 变量的赋值操作一起被组合到 <client>() 方法中，并在初始化阶段被执行。

## 10.父子类构造函数、静态、非静态代码块加载顺序？      
**执行顺序：** 父类静态代码块 -> 子类静态代码块 -> 父类非静态代码块 -> 父类构造方法 -> 子类非静态代码块 -> 子类构造方法        


## 11.BIO、NIO再到Netty     

链接：https://mp.weixin.qq.com/s/FBB_ia2xAFBLw_kTuoaeDQ     
      https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/BIO-NIO-AIO.md

**BIO:**        
传统的阻塞式通信流程        
早期的 Java 网络相关的 API(java.net包) 使用 Socket(套接字)进行网络通信，不过只支持阻塞函数使用。        

要通过互联网进行通信，至少需要一对套接字：      
1.运行于服务器端的 Server Socket        
2.运行于客户机端的 Client Socket      

Socket 网络通信过程简单来说分为下面 4 步：      

建立服务端并且监听客户端请求
客户端请求，服务端和客户端建立连接
两端之间可以传递数据
关闭资源
对应到服务端和客户端的话，是下面这样的。

**服务器端：**

1.创建 ServerSocket 对象并且绑定地址（ip）和端口号(port)：server.bind(new InetSocketAddress(host, port))      
2.通过 accept()方法监听客户端请求       
3.连接建立后，通过输入流读取客户端发送的请求信息        
4.通过输出流向客户端发送响应信息        
5.关闭相关资源        

**客户端：**

1.创建Socket 对象并且连接指定的服务器的地址（ip）和端口号(port)：socket.connect(inetSocketAddress)     
2.连接建立后，通过输出流向服务器端发送请求信息      
3.通过输入流获取服务器响应的信息        
4.关闭相关资源

**服务端：**
```
public class HelloServer {
    private static final Logger logger = LoggerFactory.getLogger(HelloServer.class);

    public void start(int port) {
        //1.创建 ServerSocket 对象并且绑定一个端口
        try (ServerSocket server = new ServerSocket(port);) {
            Socket socket;
            //2.通过 accept()方法监听客户端请求， 这个方法会一直阻塞到有一个连接建立
            while ((socket = server.accept()) != null) {
                logger.info("client connected");
                try (ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
                     ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream())) {
                   //3.通过输入流读取客户端发送的请求信息
                    Message message = (Message) objectInputStream.readObject();
                    logger.info("server receive message:" + message.getContent());
                    message.setContent("new content");
                    //4.通过输出流向客户端发送响应信息
                    objectOutputStream.writeObject(message);
                    objectOutputStream.flush();
                } catch (IOException | ClassNotFoundException e) {
                    logger.error("occur exception:", e);
                }
            }
        } catch (IOException e) {
            logger.error("occur IOException:", e);
        }
    }

    public static void main(String[] args) {
        HelloServer helloServer = new HelloServer();
        helloServer.start(6666);
    }
}

```
ServerSocket 的 **accept（） 方法是阻塞方法，也就是说 ServerSocket 在调用 accept（)等待客户端的连接请求时会阻塞，直到收到客户端发送的连接请求才会继续往下执行代码**，因此我们需要要为每个 Socket 连接开启一个线程（可以通过线程池来做）。

上述服务端的代码只是为了演示，并没有考虑多个客户端连接并发的情况。        

**客户端：**        


```
public class HelloClient {

    private static final Logger logger = LoggerFactory.getLogger(HelloClient.class);

    public Object send(Message message, String host, int port) {
        //1. 创建Socket对象并且指定服务器的地址和端口号
        try (Socket socket = new Socket(host, port)) {
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
            //2.通过输出流向服务器端发送请求信息
            objectOutputStream.writeObject(message);
            //3.通过输入流获取服务器响应的信息
            ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
            return objectInputStream.readObject();
        } catch (IOException | ClassNotFoundException e) {
            logger.error("occur exception:", e);
        }
        return null;
    }

    public static void main(String[] args) {
        HelloClient helloClient = new HelloClient();
        helloClient.send(new Message("content from client"), "127.0.0.1", 6666);
        System.out.println("client receive message:" + message.getContent());
    }
}
```


```
服务端:

[main] INFO github.javaguide.socket.HelloServer - client connected
[main] INFO github.javaguide.socket.HelloServer - server receive message:content from client

客户端：

client receive message:new content

```
**资源消耗严重的问题**
很明显，我上面演示的代码片段有一个很严重的问题：**只能同时处理一个客户端的连接，如果需要管理多个客户端的话，就需要为我们请求的客户端单独创建一个线程**。

我们知道线程是很宝贵的资源，如果我们**为每一次连接都用一个线程处理的话，就会导致线程越来越多，最后达到了极限之后**，就无法再创建线程处理请求了。处理的不好的话，**甚至可能直接就宕机掉了**。

那有没有改进的方法呢？

线程池虽可以改善，但终究未从根本解决问题
当然有！**比较简单并且实际的改进方法就是使用线程池。线程池还可以让线程的创建和回收成本相对较低**，并且我们可以指定线程池的可创建线程的最大数量，这样就不会导致线程创建过多，机器资源被不合理消耗。        
但是，即使你**再怎么优化和改变。也改变不了它的底层仍然是同步阻塞的 BIO 模型的事实**，因此无法从根本上解决问题。

        
        

**NIO:**

**NIO简介：**       
NIO是一种同步非阻塞的I/O模型，在Java 1.4 中引入了 NIO 框架，对应 java.nio 包，**提供了 Channel , Selector，Buffer等抽象**。

NIO中的N可以理解为**Non-blocking**，不单纯是New。**它支持面向缓冲的，基于通道的I/O操作方法**。 NIO提供了与传统BIO模型中的 Socket 和 ServerSocket 相对应的 SocketChannel 和 ServerSocketChannel 两种不同的套接字通道实现,两种通道都支持阻塞和非阻塞两种模式。  


1.阻塞模式 : 基本不会被使用到。使用起来就像传统的网络编程一样，**比较简单，但是性能和可靠性都不好。对于低负载、低并发的应用程序**，勉强可以用一下以提升开发速率和更好的维护性        
2.非阻塞模式 ：与阻塞模式正好相反，非阻塞模式**对于高负载、高并发的（网络）应用来说非常友好，但是编程麻烦**，这个是大部分人诟病的地方。所以， 也就导致了 Netty 的诞生。          

**NIO特性：**       
如果是在面试中回答这个问题，我觉得首先肯定要从 NIO 流是非阻塞 IO 而 IO 流是阻塞 IO 说起。然后，可以从 NIO 的3个核心组件/特性为 NIO 带来的一些改进来分析。如果，你把这些都回答上了我觉得你对于 NIO 就有了更为深入一点的认识，面试官问到你这个问题，你也能很轻松的回答上来了。

1)Non-blocking IO（非阻塞IO）       
**IO流是阻塞的，NIO流是不阻塞的**。

Java **NIO使我们可以进行非阻塞IO操作。比如说，单线程中从通道读取数据到buffer，同时可以继续做别的事情，当数据读取到buffer中后，线程再继续处理数据。写数据也是一样的**。另外，非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。

Java IO的各种流是阻塞的。这意味着，当一个线程调用 read() 或 write() 时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了

2)Buffer(缓冲区)        
**IO 面向流(Stream oriented)，而 NIO 面向缓冲区(Buffer oriented)**。       

Buffer是一个对象，它包含一些要写入或者要读出的数据。在NIO类库中加入Buffer对象，体现了新库与原I/O的一个重要区别。在面向流的I/O中可以将数据直接写入或者将数据直接读到 Stream 对象中。虽然 Stream 中也有 Buffer 开头的扩展类，但只是流的包装类，还是从流读到缓冲区，而 NIO 却是直接读到 Buffer 中进行操作。

**在NIO厍中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的; 在写入数据时，写入到缓冲区中**。任何时候访问NIO中的数据，都是通过缓冲区进行操作。

最常用的缓冲区是 ByteBuffer,一个 ByteBuffer 提供了一组功能用于操作 byte 数组。除了ByteBuffer,还有其他的一些缓冲区，事实上，每一种Java基本类型（除了Boolean类型）都对应有一种缓冲区。

3)Channel (通道)        
**NIO 通过Channel（通道） 进行读写**。

**通道是双向的，可读也可写，而流的读写是单向的**。无论读写，通道只能和Buffer交互。因为 Buffer，通道可以异步地读写。

4)Selector (选择器)     
**NIO有选择器，而IO没有**。     

**选择器用于使用单个线程处理多个通道**。因此，它需要较少的线程来处理这些通道。**线程之间的切换对于操作系统来说是昂贵的**。 因此，为了提高系统效率选择器是有用的。Selector（选择器，也可以理解为多路复用器）是 NIO（非阻塞 IO）实现的关键。它使用了事件通知相关的 API 来实现选择已经就绪也就是能够进行 I/O 相关的操作的任务的能力。      
简单来说，整个过程是这样的：        

1.将 Channel 注册到 Selector 中。       
2.调用 Selector 的 select() 方法，这个方法会阻塞；      
3.到注册在 Selector 中的某个 Channel 有新的 TCP 连接或者可读写事件的话，这个 Channel 就会处于就绪状态，会被 Selector 轮询出来。          
4.然后通过 SelectionKey 可以获取就绪 Channel 的集合，进行后续的 I/O 操作。      

相比于传统的 BIO 模型来说， NIO 模型的最大改进是：      

**1.使用比较少的线程便可以管理多个客户端的连接，提高了并发量并且减少的资源消耗（减少了线程的上下文切换的开销）        
2.在没有 I/O 操作相关的事情的时候，线程可以被安排在其他任务上面，以让线程资源得到充分利用。**

**使用 NIO 编写代码太难了**     
一个使用 NIO 编写的 Server 端如下，可以看出还是整体还是比较复杂的，并且代码读起来不是很直观，并且还可能由于 NIO 本身会存在 Bug。     

很少使用 NIO，很大情况下也是因为使用 NIO 来创建正确并且安全的应用程序的开发成本和维护成本都比较大。所以，一般情况下我们都会使用 Netty 这个比较成熟的高性能框架来做

**Netty:**      
简单用 3 点概括一下 Netty 吧！

1.Netty 是一个基于 NIO 的 client-server(客户端服务器)框架，使用它可以快速简单地开发网络应用程序。        
2.它极大地简化并简化了 TCP 和 UDP 套接字服务器等网络编程,并且性能以及安全性等很多方面甚至都要更好。      
3.支持多种协议如 FTP，SMTP，HTTP 以及各种二进制和基于文本的传统协议。        

用官方的总结就是：**Netty 成功地找到了一种在不妥协可维护性和性能的情况下实现易于开发，性能，稳定性和灵活性的方法。**

Netty能做什么？     
使用 Netty 都可以做并且更好。Netty 主要用来做**网络通信** :     
1.**作为 RPC 框架的网络通信工具** ：我们在分布式系统中，不同服务节点之间经常需要相互调用，这个时候就需要 RPC 框架了。不同服务指点的通信是如何做的呢？可以使用 Netty 来做。比如我调用另外一个节点的方法的话，至少是要让对方知道我调用的是哪个类中的哪个方法以及相关参数吧！      
2.**实现一个自己的 HTTP 服务器** ：通过 Netty 我们可以自己实现一个简单的 HTTP 服务器，这个大家应该不陌生。说到 HTTP 服务器的话，作为 Java 后端开发，我们一般使用 Tomcat 比较多。一个最基本的 HTTP 服务器可要以处理常见的 HTTP Method 的请求，比如 POST 请求、GET 请求等等。      
3.**实现一个即时通讯系统** ：使用 Netty 我们可以实现一个可以聊天类似微信的即时通讯系统，这方面的开源项目还蛮多的，可以自行去 Github 找一找。      
4.**消息推送系统** ：市面上有很多消息推送系统都是基于 Netty 来做的。

## 12.Object类有哪些方法        

```
// native方法，用于返回当前运行时对象的Class对象，使用了final关键字修饰，故不允许子类重写。
public final native Class<?> getClass();

// native方法，用于返回对象的哈希码，主要使用在哈希表中，比如JDK中的HashMap。
public native int hashCode();

// naitive方法，用于创建并返回当前对象的一份拷贝。
// 一般情况下，对于任何对象 x，表达式 x.clone() != x 为true，x.clone().getClass() == x.getClass() 为true。
// Object本身没有实现Cloneable接口，所以不重写clone方法并且进行调用的话会发生CloneNotSupportedException异常。
protected native Object clone() throws CloneNotSupportedException;

// 用于比较2个对象的内存地址是否相等，String类对该方法进行了重写用户比较字符串的值是否相等。
public boolean equals(Object obj);

// 返回类的名字@实例的哈希码的16进制的字符串。建议Object所有的子类都重写这个方法。
public String toString();

// native方法，并且不能重写。唤醒一个在此对象监视器上等待的线程(监视器相当于就是锁的概念)。
// 如果有多个线程在等待只会任意唤醒一个。
public final native void notify();

// native方法，并且不能重写。跟notify一样，唯一的区别就是会唤醒在此对象监视器上等待的所有线程，而不是一个线程。
public final native void notifyAll();

// native方法，并且不能重写。暂停线程的执行。注意：sleep方法没有释放锁，而wait方法释放了锁 。timeout是等待时间。
public final native void wait(long timeout) throws InterruptedException;

// 多了nanos参数，这个参数表示额外时间（以毫微秒为单位，范围是 0-999999）。 所以超时的时间还需要加上nanos毫秒。
public final void wait(long timeout, int nanos) throws InterruptedException;

// 跟之前的2个wait方法一样，只不过该方法一直等待，没有超时时间这个概念
public final void wait() throws InterruptedException;

// 实例被垃圾回收器回收的时候触发的操作，它能干的finally块都能干，所以不要用，因为虚拟机并不保证这个方法执行完成
protected void finalize() throws Throwable { }
```
## 13.try-catch-finally块       
**try 块**：用于捕获异常。其后可接零个或多个 catch 块，如果没有 catch 块，则必须跟一个 finally 块。     
**catch 块**：用于处理在 try 块捕获到的异常。       
**finally 块**：无论是否捕获或处理异常，finally 块里的语句都会被执行。      
**1.当在 try 块或 catch 块中遇到 return 语句时**，比如 return a，虚拟机会先将这个 return 的结果保存起来，假设现在 a = 1，**finally 语句块将在方法返回之前被执行，然后再执行这个 return 语句**，不过即使我们在 finally 块中将 a 的值改为了 2，这里 return 的还是 1。      
**2.如果在 finally 块中遇到了 return 语句**，它就不管前面的 return 语句，真的 return 了。不过最好不要这么干，在 finally 块中写 return 是很糟糕的写法。**当 try 语句和 finally 语句中都有 return 语句时，在方法返回之前，finally 语句的内容将被执行，并且 finally 语句的返回值将会覆盖原始的返回值。**        
**3**.在以下 4 种特殊情况下，**finally 块不会被执行或不会被从头到尾执行完**：        
（1）. 在 finally 语句块中发生了异常。       
（2）. 在前面的代码中用了 System.exit() 退出程序。       
（3）. 程序所在的线程死亡。      
（4）. 关闭 CPU。



## 14.final、finally、finalize区别      
链接：https://www.cnblogs.com/wisefulman/p/10584515.html        
https://blog.csdn.net/a4171175/article/details/90749839     

**final**

用于修饰类、成员变量和成员方法。final修饰的类，不能被继承（String、StrngBuilder、StringBuffer、Math，不可变类），其中所有的方法都不能被重写，所有不能同时用abstract和final修饰（abstract修饰的是抽象类，抽象类是用于被子类继承的，和final起相反的作用）；final修饰的方法不能被重写，但是子类可以用父类中final修饰的方法；final修饰的成员变量是不可变的，如果成员变量是基本数据类型，初始化之后成员变量的值不能被改变，如果成员变量是引用类型，那么它只能指向初始化时指向的那个对象，不能再指向别的对象，但是对象中的内容是允许改变的。final修饰形参，在构造函数初始化中，可以防止重排序在对属性变量初始化前，进行变量操作。

**finally**

finally是在异常处理时提供finally块来执行任何清除操作。不管有没有异常被抛出、捕获都会被执行。try块中的内容是在无异常时执行到结束。catch块中的内容，是在try块内容发生catch所声明的异常时，跳转到catch块中执行。finally块则是无论异常是否发生都会执行finally块的内容，所以在代码逻辑中有需要无论发生什么都必须执行的代码，可以放在finally块中。

**finalize**

finalize是方法名，java技术允许使用finalize（）方法在垃圾收集器将对象从内存中清楚出去之前做必要的清理工作。这个方法是由垃圾收集器在确定这个对象没有被引用时对这个对象调用的，它是在Object类中定义的，因此所有的类都继承了它。子类覆盖finalize（）方法以整理系统资源或者执行其他清理工作。finalize（）方法是在垃圾收集器删除对象之前对这个对象调用的。

finalize()只会在对象内存回收前被调用一次；finalize()的调用具有不确定行，只保证方法会调用，但不保证方法里的任务会被执行完（比如一个对象手脚不够利索，磨磨叽叽，还在自救的过程中，被杀死回收了）。        
finalize()的作用往往被认为是用来做最后的资源回收。
基于在自我救赎中的表现来看，此方法有很大的不确定性（不保证方法中的任务执行完）而且运行代价较高。所以用来回收资源也不会有什么好的表现。


## 15.类文件结构        
Class 文件是一组以 8 位字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在 Class 文件中，中间没有任何分隔符。Java 虚拟机规范规定 Class 文件采用一种类似 C 语言结构体的伪结构来存储数据，这种**伪结构中只有两种数据类型：无符号数和表**，我们之后也主要对这两种类型的数据类型进行解析。

**1.无符号数： 无符号数属于基本数据类型，以 u1、u2、u4、u8** 分别代表 1 个字节、2 个字节、4 个字节和 8 个字节的无符号数，可以用它来描述数字、索引引用、数量值或 utf-8 编码的字符串值。        

**2.表： 表是由多个无符号数或其他表为数据项构成的复合数据类型，名称上都以 _info 结尾**。

**Class 文件的头 8 个字节**     

Class 文件的头 8 个字节是魔数和版本号，**其中头 4 个字节是魔数，也就是 0xCAFEBABE，它可以用来确定这个文件是否为一个能被虚拟机接受的 Class 文件**（这通过扩展名来识别文件类型要安全，毕竟扩展名是可以随便修改的）。

**后 4 个字节则是当前 Class 文件的版本号，其中第 5、6 个字节是次版本号，第 7、8 个字节是主版本号**。高版本号的JDK能够向下兼容曾经版本号的Class文件，可是无法执行以后版本号的Class文件，即使文件格式并未发生变化，虚拟机也必须拒绝执行超过其版本号号的Class文件。     

**常量池**      

**从第 9 个字节开始，就是常量池的入口**，常量池是 Class 文件中：

与其他项目关联最多的的数据类型；        
占用 Class 文件空间最大的数据项目；     
Class 文件中第一个出现的表类型数据项目。        

**常量池的开始的两个字节，也就是第 9、10 个字节，放置一个 u2 类型的数据，标识常量池中常量的数量 cpc** (constant_pool_count)，这个计数值**有一个十分特殊的地方，就是它是从 1 开始而不是从 0 开始的，也就是说如果 cpc = 22，那么代表常量池中有 21 项常量，索引值为 1 ~ 21，第 0 项常量被空出来，为了满足后面某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”时，将让这个索引值指向 0 即可**。        


常量池中记录的是代码出现过的所有 token（类名，成员变量名等，也是我们接下来要修改的地方）以及符号引用（方法引用，成员变量引用等），主要包括以下两大类常量：      


**字面量**：接近于 Java 语言层面的常量概念，包括        
1.文本字符串      
2.声明为 final 的常量值    

**符号引用**：以一组符号来描述所引用的目标，包括        
1.类和接口的全限定名      
2.字段的名称和描述符      
3.方法的名称和描述符      


```
常量池中的每一项常量都通过一个表来存储。目前一共有 14 种常量，不过麻烦的地方就在于，这 14 种常量类型每一种都有自己的结构，我们在这里只详细介绍两种：CONSTANT_Class_info 和 CONSTANT_Utf8_info。

CONSTANT_Class_info 的存储结构为：

... [ tag=7 ] [ name_index ] ...
... [  1位  ] [     2位    ] ...
其中，tag 是标志位，用来区分常量类型的，tag = 7 就表示接下来的这个表是一个 CONSTANT_Class_info，name_index 是一个索引值，指向常量池中的一个 CONSTANT_Utf8_info 类型的常量所在的索引值，CONSTANT_Utf8_info 类型常量一般被用来描述类的全限定名、方法名和字段名。它的存储结构如下：

... [ tag=1 ] [ 当前常量的长度 len ] [ 常量的符号引用的字符串值 ] ...
... [  1位  ] [        2位        ] [         len位         ] ...
```

在本项目中，我们需要修改的就是值为 java/lang/System 的 CONSTANT_Utf8_info 的常量，因为**在类加载的过程中，虚拟机会将常量池中的“符号引用”替换为“直接引用”，而 java/lang/System 就是用来寻找其方法的直接引用的关键所在，我们只要将 java/lang/System 修改为我们的类的全限定名，就可以在运行时将通过 System.xxx 运行的方法偷偷的替换为我们的方法**。


```
public class ByteUtils {

    public static int byte2Int(byte[] b, int start, int len) {
        int res = 0;
        int end = start + len;
        for (int i = start; i < end; i++) {
            int cur = ((int) b[i]) & 0xff;
            cur <<= (--len) * 8;
            res += cur;
        }
        return res;
    }

    public static byte[] int2Byte(int num, int len) {
        byte[] b = new byte[len];
        for (int i = 0; i < len; i++) {
            b[len - i - 1] = (byte) ((num >> (8 * i)) & 0xff);
        }
        return b;
    }

    public static String byte2String(byte[] b, int start, int len) {
        return new String(b, start, len);
    }

    public static byte[] string2Byte(String str) {
        return str.getBytes();
    }

    public static byte[] byteReplace(byte[] oldBytes, int offset, int len, byte[] replaceBytes) {
        byte[] newBytes = new byte[oldBytes.length + replaceBytes.length - len];
        System.arraycopy(oldBytes, 0, newBytes, 0, offset);
        System.arraycopy(replaceBytes, 0, newBytes, offset, replaceBytes.length);
        System.arraycopy(oldBytes, offset + len, newBytes, offset + replaceBytes.length,
                oldBytes.length - offset - len);
        return newBytes;
    }
}
```
实现字节码修改器
介绍完会用到的基础知识，接下来就是本篇的重头戏：实现字节码修改器。通过之前的说明，我们可以通过以下流程完成我们的字节码修改器：

1.取出常量池中的常量的个数 cpc；        
2.遍历常量池中 cpc 个常量，检查 tag = 1 的 CONSTANT_Utf8_info 常量；      
3.找到存储的常量值为 java/lang/System 的常量，把它替换为 org/olexec/execute/HackSystem；     
4.因为只可能有一个值为 java/lang/System 的 CONSTANT_Utf8_info 常量，所以找到之后可以立即返回修改后的字节码。      


```
public class ClassModifier {
    /**
     * Class文件中常量池的起始偏移
     */
    private static final int CONSTANT_POOL_COUNT_INDEX = 8;

    /**
     * CONSTANT_UTF8_INFO常量的tag
     */
    private static final int CONSTANT_UTF8_INFO = 1;

    /**
     * 常量池中11种常量的长度，CONSTANT_ITEM_LENGTH[tag]表示它的长度
     */
    private static final int[] CONSTANT_ITEM_LENGTH = {-1, -1, -1, 5, 5, 9, 9, 3, 3, 5, 5, 5, 5};

    /**
     * 1个和2个字节的符号数，用来在classByte数组中取tag和len
     * tag用u1个字节表示
     * len用u2个字节表示
     */
    private static final int u1 = 1;
    private static final int u2 = 2;

    /**
     * 要被修改的字节码文件
     */
    private byte[] classByte;

    public ClassModifier(byte[] classByte) {
        this.classByte = classByte;
    }

    /**
     * 从0x00000008开始向后取2个字节，表示的是常量池中常量的个数
     * @return 常量池中常量的个数
     */
    public int getConstantPoolCount() {
        return ByteUtils.byte2Int(classByte, CONSTANT_POOL_COUNT_INDEX, u2);
    }

    /**
     * 字节码修改器，替换字节码常量池中 oldStr 为 newStr
     * @param oldStr
     * @param newStr
     * @return 修改后的字节码字节数组
     */
    public byte[] modifyUTF8Constant(String oldStr, String newStr) {
        int cpc = getConstantPoolCount();
        int offset = CONSTANT_POOL_COUNT_INDEX + u2;  // 真实的常量起始位置
        for (int i = 1; i < cpc; i++) {
            int tag = ByteUtils.byte2Int(classByte, offset, u1);
            if (tag == CONSTANT_UTF8_INFO) {
                int len = ByteUtils.byte2Int(classByte, offset + u1, u2);
                offset += u1 + u2;
                String str = ByteUtils.byte2String(classByte, offset, len);
                if (str.equals(oldStr)) {
                    byte[] strReplaceBytes = ByteUtils.string2Byte(newStr);
                    byte[] intReplaceBytes = ByteUtils.int2Byte(strReplaceBytes.length, u2);
                    // 替换新的字符串的长度
                    classByte = ByteUtils.byteReplace(classByte, offset - u2, u2, intReplaceBytes);
                    // 替换字符串本身
                    classByte = ByteUtils.byteReplace(classByte, offset, len, strReplaceBytes);
                    return classByte;  // 就一个地方需要改，改完就可以返回了
                } else {
                    offset += len;
                }
            } else {
                offset += CONSTANT_ITEM_LENGTH[tag];
            }
        }
        return classByte;
    }

}
```

## 16.Java是编译型语言还是解释型语言？      
        
链接：https://blog.csdn.net/qzc70919700/article/details/72515022

* 你可以说它是编译型的。因为所有的Java代码都是要编译的，.java不经过编译就什么用都没有。 
* 你可以说它是解释型的。因为java代码编译后不能直接运行，它是解释运行在JVM上的，所以它是解释运行的，那也就算是解释的了。 
* 但是，现在的JVM为了效率，都有一些JIT优化。它又会把.class的二进制代码编译为本地的代码直接运行，所以，又是编译的。        

**定义**：      

**编译型语言**：把做好的源程序全部编译成二进制代码的可运行程序。然后，可直接运行这个程序。        

**解释型语言**：把做好的源程序翻译一句，然后执行一句，直至结束！       

**区别**：      

编译型语言，执行速度快、效率高；依靠编译器、跨平台性差些。      

解释型语言，执行速度慢、效率低；依靠解释器、跨平台性好。        

java是解释型的语言，因为**虽然java也需要编译，编译成.class文件，但是并不是机器可以识别的语言，而是字节码，最终还是需要 jvm的解释，才能在各个平台执行，这同时也是java跨平台的原因**。所以可是说java即是编译型的，也是解释型，但是假如非要归类的话，从概念上的定义，恐怕java应该归到解释型的语言中。 














# Spring
## 1.Spring MVC如何拦截ip
链接：https://www.cnblogs.com/huanmin/p/7810467.html        
注解实现

```
public class AuthorityIPInterceptor extends HandlerInterceptorAdapter {
    private final static Logger logger = LoggerFactory.getLogger(AuthorityIPInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        if (handler instanceof HandlerMethod) {
            IPFilter ipFilter = ((HandlerMethod) handler).getMethodAnnotation(IPFilter.class);
            if (ipFilter == null) {
                return true;
            }
            String ipAddress = getIpAddress(request);
            ArrayList<String> deny = Lists.newArrayList(ipFilter.deny());
            ArrayList<String> allow = Lists.newArrayList(ipFilter.allow());

            //设置了黑名单,  黑名单内ip不给通过
            if (CollectionUtils.isNotEmpty(deny)) {
                if (deny.contains(ipAddress)) {
                    logger.info("IP: "+ipAddress+" 被拦截");
                    response.setStatus(500);
                    return false;
                }
            }
            //设置了白名单,  只有白名单内的ip给通过
            if (CollectionUtils.isNotEmpty(allow)) {
                if (allow.contains(ipAddress)) {
                    logger.info("IP: "+ipAddress+" 被放行");
                    return true;
                }else {
                    logger.info("IP: "+ipAddress+" 没有放行权利");
                    response.setStatus(500);
                    return false;
                }
            }
        }
        return true;
    }

通过继承springmvc的拦截器，对所有访问当前加了注解的controller接口和方法进行拦截
```

## 2.Spring MVC拦截器      
链接：https://www.cnblogs.com/lukelook/p/11079113.html#t1
**1.拦截器概念：**        
SpringMVC 中的Interceptor 拦截器的**主要作用就是拦截用户的 url 请求**,并在执行 handler 方法的前中后加入某些特殊请求,类似于 servlet 里面的过滤器.

**SpringMVC 中的Interceptor 拦截器也是相当重要和相当有用的，它的主要作用是拦截用户的请求并进行相应的处理**。比如通过它来进行权限验证，或者是来判断用户是否登陆，或者是像12306 那样子判断当前时间是否是购票时间。        

**2.定义Interceptor实现类**    
SpringMVC 中的Interceptor 拦截请求是通过HandlerInterceptor 来实现的。在SpringMVC 中定义一个Interceptor 非常简单，主要有**两种方式**：

第一种方式是要**定义的Interceptor类要实现了Spring 的HandlerInterceptor 接口**，或者是这个类**继承实现了HandlerInterceptor 接口的类，比如Spring 已经提供的实现了HandlerInterceptor 接口的抽象类HandlerInterceptorAdapter** ；

第二种方式是**实现Spring的WebRequestInterceptor接口**，或者是继承实现了WebRequestInterceptor的类。

**3.实现HandlerInterceptor接口**        
**HandlerInterceptor 接口中定义了三个方法**，我们就是通过这三个方法来对用户的请求进行拦截处理的。

（1 ）**preHandle** (HttpServletRequest request, HttpServletResponse response, Object handle) 方法，顾名思义，该方法将在请求处理之前进行调用。SpringMVC 中的Interceptor 是链式的调用的，在一个应用中或者说是在一个请求中可以同时存在多个Interceptor 。**每个Interceptor 的调用会依据它的声明顺序依次执行，而且最先执行的都是Interceptor 中的preHandle 方法，所以可以在这个方法中进行一些前置初始化操作或者是对当前请求的一个预处理**，也可以在这个方法中进行一些判断来决定请求是否要继续进行下去。**该方法的返回值是布尔值Boolean 类型的，当它返回为false 时，表示请求结束，后续的Interceptor 和Controller 都不会再执行；当返回值为true 时就会继续调用下一个Interceptor 的preHandle 方法，如果已经是最后一个Interceptor 的时候就会是调用当前请求的Controller 方法**。

（2 ）**postHandle** (HttpServletRequest request, HttpServletResponse response, Object handle, ModelAndView modelAndView) 方法，由preHandle 方法的解释我们知道**这个方法包括后面要说到的afterCompletion 方法都只能是在当前所属的Interceptor 的preHandle 方法的返回值为true 时才能被调用**。postHandle 方法，顾名思义**就是在当前请求进行处理之后，也就是Controller 方法调用之后执行，但是它会在DispatcherServlet 进行视图返回渲染之前被调用，所以我们可以在这个方法中对Controller 处理之后的ModelAndView 对象进行操作**。postHandle 方法被调用的方向跟preHandle 是相反的，也就是说先声明的Interceptor 的postHandle 方法反而会后执行，这和Struts2 里面的Interceptor 的执行过程有点类型。Struts2 里面的Interceptor 的执行过程也是链式的，只是在Struts2 里面需要手动调用ActionInvocation 的invoke 方法来触发对下一个Interceptor 或者是Action 的调用，然后每一个Interceptor 中在invoke 方法调用之前的内容都是按照声明顺序执行的，而invoke 方法之后的内容就是反向的。

（3 ）**afterCompletion**(HttpServletRequest request, HttpServletResponse response, Object handle, Exception ex) 方法，**该方法也是需要当前对应的Interceptor 的preHandle 方法的返回值为true 时才会执行**。顾名思义，**该方法将在整个请求结束之后，也就是在DispatcherServlet 渲染了对应的视图之后执行。这个方法的主要作用是用于进行资源清理工作的。**


```
package com.my.dm.interceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

public class TestInterceptor implements HandlerInterceptor {

        private Logger logger =LogManager.getLogger(TestInterceptor.class);
                                                     
    @Override
    public void afterCompletion(HttpServletRequest arg0, HttpServletResponse arg1, Object arg2, Exception arg3)
            throws Exception {
        // TODO Auto-generated method stub
        logger.error("afterCompletion");
    }

    @Override
    public void postHandle(HttpServletRequest arg0, HttpServletResponse arg1, Object arg2, ModelAndView arg3)
            throws Exception {
        // TODO Auto-generated method stub
        logger.error("postHandle");
    }

    @Override
    public boolean preHandle(HttpServletRequest arg0, HttpServletResponse arg1, Object arg2) throws Exception {
        // TODO Auto-generated method stub
        logger.error("preHandle");
        return true;
    }

}

多个拦截器的配置文件：
<mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/employee/**" />
            <mvc:mapping path="/trainning/**" />
            <mvc:mapping path="/manage/**" /> 
            <mvc:exclude-mapping path="/**/fonts/*" />
            <mvc:exclude-mapping path="/**/*.css" />
            <mvc:exclude-mapping path="/**/*.js" />
            <mvc:exclude-mapping path="/**/*.png" />
            <mvc:exclude-mapping path="/**/*.gif" />
            <mvc:exclude-mapping path="/**/*.jpg" />
            <mvc:exclude-mapping path="/**/*.jpeg" />
            <bean class="com.pmo.interceptor.PageInterceptor" />
        </mvc:interceptor>
        <mvc:interceptor>  
                <mvc:mapping path="/**"/>
                <bean class="com.pmo.interceptor.LoginInterceptor"></bean>  
        </mvc:interceptor>
        <mvc:interceptor>  
                <mvc:mapping path="/**"/>
                <bean class="com.pmo.interceptor.UserAuthorityInterceptor"></bean>  
        </mvc:interceptor>
    </mvc:interceptors>
    
```
 
## 3.Spring中循环依赖问题       
链接：https://mp.weixin.qq.com/s/G3FdvM_mAgbmiFricvyLKQ     
https://mp.weixin.qq.com/s/wL23OgJN-RDc-Hq8c5UbsQ       

**一、什么是循环依赖？**      
循环，依赖，那是不是 A 引用了 B ，而此时 B 引用了 C,而 C 呢又引用了A，于是一个三角恋的关系出现了。       

**二、循环依赖会带来怎样的问题？**        
循环依赖最直接的问题就是会出现在对象的实例化上面，创建对象的时候，如果在Spring的配置中加入这种 A 依赖 B ，B 依赖 C，C 依赖 A 的话，那么最终创建 A 的实例对象的时候，会出现错误。
而**如果这种循环调用的依赖不去终结掉他的话，那么就相当于一个死循环。**

**三、循环依赖解决方法**        

**（1）构造器参数循环依赖**     

**Spring容器会将每一个正在创建的Bean标识符放在一个“当前创建Bean池”中，Bean标识符在创建过程中将一直保持在这个池中。     
因此如果在创建Bean过程中发现自己已经在“当前创建Bean池”里时将抛出BeanCurrentlyInCreationException异常表示循环依赖；而对于创建完毕的Bean将从“当前创建Bean池”中清除掉。**      

```
先初始化三个Bean
public class StudentA {
    private StudentB studentB ;
    public void setStudentB(StudentB studentB) {
        this.studentB = studentB;
    }
    public StudentA() {
    }
    public StudentA(StudentB studentB) {
        this.studentB = studentB;
    }
}

public class StudentB {
    private StudentC studentC ;
    public void setStudentC(StudentC studentC) {
        this.studentC = studentC;
    }
    public StudentB() {
    }
    public StudentB(StudentC studentC) {
        this.studentC = studentC;
    }
}

public class StudentC {
    private StudentA studentA ;
    public void setStudentA(StudentA studentA) {
        this.studentA = studentA;
    }
    public StudentC() {
    }
    public StudentC(StudentA studentA) {
        this.studentA = studentA;
    }
}
上面是很基本的3个类，StudentA有参构造是StudentB。StudentB的有参构造是StudentC，
StudentC的有参构造是StudentA ，这样就产生了一个循环依赖的情况。

我们都把这三个Bean交给 Spring 管理，并用有参构造实例化。

<bean id="a" class="com.zfx.student.StudentA">
  <constructor-arg index="0" ref="b"></constructor-arg>
</bean>
<bean id="b" class="com.zfx.student.StudentB">
  <constructor-arg index="0" ref="c"></constructor-arg>
</bean>
<bean id="c" class="com.zfx.student.StudentC">
  <constructor-arg index="0" ref="a"></constructor-arg>
</bean>

测试：
public class Test {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("com/zfx/student/applicationContext.xml");
        //System.out.println(context.getBean("a", StudentA.class));
    }
}

报错：
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException:
  Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
  
```
Spring容器先创建单例StudentA，StudentA依赖StudentB，然后将A放在“当前创建Bean池”中。
此时创建StudentB,StudentB依赖StudentC ,然后将B放在“当前创建Bean池”中,此时创建StudentC，StudentC又依赖StudentA。
但是，此时StudentA已经在池中，所以会报错，因为在池中的Bean都是未初始化完的，所以会依赖错误 ，初始化完的Bean会从池中移除。
    
**（2）setter方式单例，默认方式**       
根据bean的生命周期可知，Spring是先将Bean对象实例化之后再设置对象属性的。

```
修改配置文件为set方式注入

<!--scope="singleton"(默认就是单例方式) -->
<bean id="a" class="com.zfx.student.StudentA" scope="singleton">
  <property name="studentB" ref="b"></property>
</bean>
<bean id="b" class="com.zfx.student.StudentB" scope="singleton">
  <property name="studentC" ref="c"></property>
</bean>
<bean id="c" class="com.zfx.student.StudentC" scope="singleton">
  <property name="studentA" ref="a"></property>
</bean>

测试类：
public class Test {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("com/zfx/student/applicationContext.xml");
        System.out.println(context.getBean("a", StudentA.class));
    }
}

结果：
com.zfx.student.StudentA@1fbfd6

```
**为什么用set方式就不报错了？**         

**Spring先是用构造实例化Bean对象 ，此时 Spring 会将这个实例化结束的对象放到一个Map中，并且 Spring 提供了获取这个未设置属性的实例化对象引用的方法。** 
结合我们的实例来看，当Spring实例化了StudentA、StudentB、StudentC后，紧接着会去设置对象的属性，此时StudentA依赖StudentB，就会去Map中取出存在里面的单例StudentB对象，以此类推，不会出来循环的问题。（单例模式下，spring容器会对bean进行缓存）

## 4.Spring中bean为什么默认单例？      
链接：https://mp.weixin.qq.com/s?__biz=MzI3ODcxMzQzMw==&mid=2247492186&idx=3&sn=cbb8fda8f97f28a24939134e711b51a9&chksm=eb50676cdc27ee7afead02ae8c3c0fb6611681c119fa0b3f6dac238676aeb2660a9d77dde392&scene=21#wechat_redirect     
        
**单例bean与原型bean的区别**

如果一个bean**被声明为单例的时候，在处理多次请求的时候在Spring容器里只实例化出一个bean**，后续的请求都公用这个对象，**这个对象会保存在一个map里面，当有请求来的时候会先从缓存(map)里查看有没有，有的话直接使用这个对象，没有的话才实例化一个新的对象**，所以这是个单例的。      
但是对于**原型(prototype)bean来说当每次请求来的时候直接实例化新的bean，没有缓存以及从缓存查的过程**。      

单例bean的优势：        
**1.减少了新生成实例的消耗**
新生成实例消耗包括两方面，第一，Spring会通过反射或者cglib来生成bean实例这都是耗性能的操作，其次给对象分配内存也会涉及复杂算法。

**2.减少jvm垃圾回收**
由于不会给每个请求都新生成bean实例，所以自然回收的对象少了。

**3.可以快速获取到bean**
因为单例的获取bean操作除了第一次生成之外其余的都是从缓存里获取的所以很快。

**单例bean的劣势：**

单例的bean一个很大的劣势就是他不能做到线程安全！！！

由于所有请求都共享一个bean实例，所以这个bean要是有状态的一个bean的话可能在并发场景下出现问题，而原型的bean则不会有这样问题（但也有例外，比如他被单例bean依赖），因为给每个请求都新创建实例。


## 4.Spring并发之-有状态bean和无状态bean        
链接：https://www.cnblogs.com/smilepup-hhr/p/11484752.html      
        
**一、有状态和无状态**      

**有状态会话bean**：     
每个用户有自己特有的一个实例，在用户的生存期内，bean保持了用户的信息，即“有状态”；一旦用户灭亡（调用结束或实例结束），bean的生命期也告结束。即每个用户最初都会得到一个初始的bean。简单来说，**有状态就是有数据存储功能。有状态对象(Stateful Bean)，就是有实例变量的对象 ，可以保存数据，是非线程安全的**。  

**无状态会话bean**：         
bean一旦实例化就被加进会话池中，各个用户都可以共用。**即使用户已经消亡，bean   的生命期也不一定结束，它可能依然存在于会话池中，供其他用户调用**。由于没有特定的用户，那么也就不能保持某一用户的状态，所以叫无状态bean。但无状态会话bean   并非没有状态，如果它有自己的属性（变量），那么这些变量就会受到所有调用它的用户的影响，这是在实际应用中必须注意的。简单来说，**无状态就是一次操作，不能保存数据。无状态对象(Stateless Bean)，就是没有实例变量的对象 不能保存数据，是不变类，是线程安全的**。        

```
有状态bean：

package com.sw;

public class TestManagerImpl implements TestManager{
    private User user;    //有一个记录信息的实例

    public void deleteUser(User e) throws Exception {
        user = e ;           //步骤1
        prepareData(e);
    }

    public void prepareData(User e) throws Exception {
        user = getUserByID(e.getId());            //步骤2
        .....
        //使用user.getId();                       //步骤3
        .....
        .....
    }    
}

```

**二、解决有状态bean的线程安全问题**        
Spring对bean的配置中有四种配置方式，我们只说其中两种：singleton单例模式、prototype原型模式。       

```
<bean id="testManager" class="com.sw.TestManagerImpl" scope="singleton" />

<bean id="testManager" class="com.sw.TestManagerImpl" scope="prototype" />

```
默认的配置是singleton。**singleton表示该bean全局只有一个实例。prototype表示该bean在每次被注入的时候，都要重新创建一个实例**，这种情况适用于有状态的Bean。如果**对有状态的bean使用了singleton的话会出现线程安全问题**。      

比如针对上面所举的例子，如果有两个用户同时访问假定为user1,user2
当user1 调用到程序中的1步骤的时候，该Bean的私有变量user被付值为user1当user1的程序走到2步骤的时候，该Bean的私有变量user被重新付值为user1_create理想的状况，当user1走到3步骤的时候，私有变量user应该为user1_create;但如果在user1调用到3步骤之前，user2开始运行到了1步骤了，由于单态的资源共享，则私有变量user被修改为user2这种情况下，user1的步骤3用到的user.getId()实际用到是user2的对象。

**解决方法：**      

1.将**有状态的bean配置成prototype模式**，让每一个线程都创建一个prototype实例。但是这样会产生很多的实例消耗较多的内存空间。

2.使用**ThreadLocal变量，为每一条线程设置变量副本**。       

以ThreadLocal为例：     

例如，我们有一个银行的BankDAO类和一个个人账户的PeopleDAO类，现在需要个人向银行进行转账，在PeopleDAO类中有一个账户减少的方法，BankDAO类中有一个账户增加的方法，那么**这两个方法在调用的时候必须使用同一个Connection数据库连接对象，如果他们使用两个Connection对象，则会开启两段事务，可能出现个人账户减少而银行账户未增加的现象**。使用同一个Connection对象的话，在应用程序中可能会设置为一个全局的数据库连接对象，从而避免在调用每个方法时都传递一个Connection对象。**问题是当我们把Connection对象设置为全局变量时，你不能保证是否有其他线程会将这个Connection对象关闭，这样就会出现线程安全问题。解决办法就是在进行转账操作这个线程中，使用ThreadLocal中获取Connection对象**，这样，在调用个人账户减少和银行账户增加的线程中，就能从ThreadLocal中取到同一个Connection对象，并且这个Connection对象为转账操作这个线程独有，不会被其他线程影响，保证了线程安全性。        


```
public class ConnectionHolder {
    public static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
    };
    
    public static Connection getConnection(){
        Connection connection = connectionHolder.get();
        if(null == connection){
            connection = DriverManager.getConnection(DB_URL);
            connectionHolder.set(connection);
        }
        return connection;
    }
}
```











# 操作系统
## 1.什么是系统调用？
根据进程访问资源的特点，我们可以把进程在系统上的运行分为两个级别：

1.用户态(user mode) : 用户态运行的进程或可以直接读取用户程序的数据。      
2.系统态(kernel mode):可以简单的理解系统态运行的进程或程序几乎可以访问计算机的任何资源，不受限制。      

说了用户态和系统态之后，那么什么是系统调用呢？

我们运行的程序基本都是运行在用户态，如果我们调用操作系统提供的系统态级别的子功能咋办呢？那就需要系统调用了！

也就是说在我们运行的用户程序中，凡是与系统态级别的资源有关的操作（如文件管理、进程控制、内存管理等)，都必须通过系统调用方式向操作系统提出服务请求，并由操作系统代为完成。      

这些系统调用按功能大致可分为如下几类：

1.设备管理。完成设备的请求或释放，以及设备启动等功能。      
2.文件管理。完成文件的读、写、创建及删除等功能。        
3.进程控制。完成进程的创建、撤销、阻塞及唤醒等功能。        
4.进程通信。完成进程之间的消息传递或信号传递等功能。        
5.内存管理。完成内存的分配、回收以及获取作业占用内存区大小及地址等功能。


## 2.进程、线程、协程区别及通信方式       
链接：https://www.jianshu.com/p/6dde7f92951e        
https://www.cnblogs.com/fanguangdexiaoyuer/p/10834737.html#_label6

**协程（Coroutines）是一种比线程更加轻量级的存在**，正如一个进程可以拥有多个线程一样，**一个线程可以拥有多个协程**。      

**协程不是被操作系统内核所管理的，而是完全由程序所控制，也就是在用户态执行**。这样带来的**好处是性能大幅度的提升，因为不会像线程切换那样消耗资源**。

**协程**不是进程也不是线程，而**是一个特殊的函数，这个函数可以在某个地方挂起，并且可以重新在挂起处外继续运行**。所以说，协程与进程、线程相比并不是一个维度的概念。

一个进程可以包含多个线程，一个线程也可以包含多个协程。简单来说，一个线程内可以由多个这样的特殊函数在运行，但是有一点必须明确的是，一个线程的多个协程的运行是串行的。如果**是多核CPU，多个进程或一个进程内的多个线程是可以并行运行的，但是一个线程内协程却绝对是串行的，无论CPU有多少个核**。毕竟协程虽然是一个特殊的函数，但仍然是一个函数。**一个线程内可以运行多个函数，但这些函数都是串行运行的。当一个协程运行时，其它协程必须挂起**。


**进程**        

进程（Process）是计算机中的程序关于某数据集合上的一次运行活动，**是系统进行资源分配和调度的基本单位**，是操作系统结构的基础。**在早期面向进程设计的计算机结构中，进程是程序的基本执行实体；在当代面向线程设计的计算机结构中，进程是线程的容器**。程序是指令、数据及其组织形式的描述，进程是程序的实体。

进程的概念主要有两点：**第一，进程是一个实体。每一个进程都有它自己的地址空间，一般情况下，包括文本区域（text region）、数据区域（data region）和堆栈（stack region）**。文本区域存储处理器执行的代码；数据区域存储变量和进程执行期间使用的动态分配的内存；堆栈区域存储着活动过程调用的指令和本地变量。第二，进程是一个“执行中的程序”。程序是一个没有生命的实体，只有处理器赋予程序生命时（操作系统执行之），它才能成为一个活动的实体，我们称其为进程。

进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动,进程是系统进行资源分配和调度的一个独立单位。每个进程都有自己的独立内存空间，不同进程通过进程间通信来通信。**由于进程比较重量，占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大，但相对比较稳定安全**。

**线程**        

**线程是进程的一个实体,是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位**.线程自己基本上不拥有系统资源,**只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈),但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源。线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据**。

一个线程可以创建和撤消另一个线程，**同一进程中的多个线程之间可以并发执行**。由于线程之间的相互制约，致使线程 在运行中呈现出间断性。线程也有就绪、阻塞和运行三种基本状态。就绪状态是指线程具备运行的所有条件，逻辑上可以运行，在等待处理机；运行状态是指线程占有处理机正在运行；阻塞状态是指线程在等待一个事件（如某个信号量），逻辑上不可执行。每一个程序都至少有一个线程，若程序只有一个线程，那就是程序本身。
**线程是程序中一个单一的顺序控制流程**。进程内一个相对独立的、可调度的执行单元，**是系统独立调度和分派CPU的基本单位指运行中的程序的调度单位。在单个程序中同时运行多个线程完成不同的工作，称为多线程**。

**进程通信**        

**管道(pipe)**  

管道是一种半双工的通信方式，数据只能单向流动，而且只能在具有亲缘关系的进程间使用。进程的亲缘关系通常是指父子进程关系。

**有名管道 (namedpipe)**   

有名管道也是半双工的通信方式，但是它允许无亲缘关系进程间的通信。

**信号量(semaphore)**    

信号量是一个计数器，可以用来控制多个进程对共享资源的访问。它常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。

**消息队列(messagequeue)**   

消息队列是由消息的链表，存放在内核中并由消息队列标识符标识。消息队列克服了信号传递信息少、管道只能承载无格式字节流以及缓冲区大小受限等缺点。

**信号 (sinal)**   

信号是一种比较复杂的通信方式，用于通知接收进程某个事件已经发生。

**共享内存(shared memory)**     

共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。共享内存是最快的 IPC 方式，它是针对其他进程间通信方式运行效率低而专门设计的。它往往与其他通信机制，如信号量，配合使用，来实现进程间的同步和通信。

**套接字(socket)**     

套接口也是一种进程间通信机制，与其他通信机制不同的是，它可用于不同设备及其间的进程通信。


**线程间的通信方式**        

**锁机制**：包括互斥锁、条件变量、读写锁 

互斥锁提供了以排他方式防止数据结构被并发修改的方法。 
读写锁允许多个线程同时读共享数据，而对写操作是互斥的。 
条件变量可以以原子的方式阻塞进程，直到某个特定条件为真为止。对条件的测试是在互斥锁的保护下进行的。条件变量始终与互斥锁一起使用。

wait/notify 等待

Volatile 内存共享

CountDownLatch 并发工具

CyclicBarrier 并发工具

**信号量机制(Semaphore)**  

包括无名线程信号量和命名线程信号量。

**信号机制(Signal)**   

类似进程间的信号处理。

线程间的通信目的主要是用于线程同步，所以线程没有像进程通信中的用于数据交换的通信机制。


    

上下文切换：        

1.进程的切换者是操作系统，切换时机是根据操作系统自己的切换策略，用户是无感知的。进程的切换内容包括页全局目录、内核栈、硬件上下文，切换内容保存在内存中。进程切换过程是由“用户态到内核态到用户态”的方式，切换效率低。

2.线程的切换者是操作系统，切换时机是根据操作系统自己的切换策略，用户无感知。线程的切换内容包括内核栈和硬件上下文。线程切换内容保存在内核栈中。线程切换过程是由“用户态到内核态到用户态”， 切换效率中等。

3.协程的切换者是用户（编程者或应用程序），切换时机是用户自己的程序所决定的。协程的切换内容是硬件上下文，切换内存保存在用户自己的变量（用户栈或堆）中。协程的切换过程只有用户态，即没有陷入内核态，因此切换效率高。

## 3.select、poll、epoll之间区别        
链接：https://www.cnblogs.com/aspirant/p/9166944.html       

**(1)select==>时间复杂度O(n)**

它**仅仅知道了，有I/O事件发生了，却并不知道是哪那几个流**（可能有一个，多个，甚至全部），我们**只能无差别轮询所有流**，找出能读出数据，或者写入数据的流，对他们进行操作。所以select具有O(n)的无差别轮询复杂度，同时处理的流越多，无差别轮询时间就越长。**select的底层是一个fd_set的数据结构，本质上是一个long类型的数组，数组中每一个元素都对应于一个文件描述符，通过轮询所有的文件描述符来检查是否有事件发生**。

**(2)poll==>时间复杂度O(n)**

**poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态， 但是它没有最大连接数的限制，原因是它是基于链表来存储的**.

**(3)epoll==>时间复杂度O(1)**

epoll可以理解为event poll，不同于忙轮询和无差别轮询，**因为epoll_wait返回的就是有事件发生的文件描述符。epoll会把哪个流发生了怎样的I/O事件通知我们**。所以我们说**epoll实际上是事件驱动**（每个事件关联上fd）的，此时我们对这些流的操作都是有意义的。（复杂度降低到了O(1)）

select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但**select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。  

epoll跟select都能提供多路I/O复用的解决方案。在现在的Linux内核里有都能够支持，其中epoll是Linux所特有，而select则应该是POSIX所规定，一般操作系统均有实现。      

**select：**

**select本质上是通过设置或者检查存放fd标志位的数据结构**来进行下一步处理。这样所带来的**缺点**是：

**1、 单个进程可监视的fd数量被限制，即能监听端口的大小有限**

一般来说这个数目和系统内存关系很大，具体数目可以cat /proc/sys/fs/file-max察看。**32位机默认是1024个。64位机默认是2048.**

**2、 对socket进行扫描时是线性扫描，即采用轮询的方法，效率较低**

当套接字比较多的时候，**每次select()都要通过遍历FD_SETSIZE个Socket来完成调度,不管哪个Socket是活跃的,都遍历一遍。这会浪费很多CPU时间**。如果能给套接字注册某个回调函数，当他们活跃时，自动完成相关操作，那就避免了轮询，这正是epoll与kqueue做的。

**3、需要维护一个用来存放大量fd的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大**

**poll：**

poll本质上和select没有区别，它**将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态，如果设备就绪则在设备等待队列中加入一项并继续遍历，如果遍历完所有fd后没有发现就绪设备，则挂起当前进程，直到设备就绪或者主动超时，被唤醒后它又要再次遍历fd**。这个过程经历了多次无谓的遍历。

**它没有最大连接数的限制，原因是它是基于链表来存储的**，但是同样有一个**缺点**：

**1、大量的fd的数组被整体复制于用户态和内核地址空间之间，而不管这样的复制是不是有意义。**                   

**2、poll还有一个特点是“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd。**

**epoll:**

**epoll有EPOLLLT和EPOLLET两种触发模式**，LT是默认的模式，ET是“高速”模式。**LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作，而在ET（边缘触发）模式中，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无 论fd中是否还有数据可读**。所以**在ET模式下，read一个fd的时候一定要把它的buffer读光**，也就是说一直读到read的返回值小于请求值，或者 遇到EAGAIN错误。还有一个特点是，**epoll使用“事件”的就绪通知方式，通过epoll_ctl注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活该fd，epoll_wait便可以收到通知**。

**epoll为什么要有EPOLLET触发模式？**

如果**采用EPOLLLT模式的话，系统中一旦有大量你不需要读写的就绪文件描述符，它们每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率.**。而采用**EPOLLET这种边沿触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你**！！！这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符

**epoll的优点：**

**1、没有最大并发连接的限制**，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）；        
**2、效率提升，不是轮询的方式**，不会随着FD数目的增加效率下降。**只有活跃可用的FD才会调用callback函数；
即Epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关**，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll。
**3、 内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递**；即epoll使用mmap减少复制开销。

**select、poll、epoll 区别总结：**

**1、支持一个进程所能打开的最大连接数**

select

单个进程所能打开的最大连接数有FD_SETSIZE宏定义，其大小是32个整数的大小（在32位的机器上，大小就是3232，同理64位机器上FD_SETSIZE为3264），当然我们可以对进行修改，然后重新编译内核，但是性能可能会受到影响，这需要进一步的测试。

poll

poll本质上和select没有区别，但是它没有最大连接数的限制，原因是它是基于链表来存储的

epoll

虽然连接数有上限，但是很大，1G内存的机器上可以打开10万左右的连接，2G内存的机器可以打开20万左右的连接

**2、FD剧增后带来的IO效率问题**

select

因为每次调用时都会对连接进行线性遍历，所以随着FD的增加会造成遍历速度慢的“线性下降性能问题”。

poll

同上

epoll

因为epoll内核中实现是根据每个fd上的callback函数来实现的，只有活跃的socket才会主动调用callback，所以在活跃socket较少的情况下，使用epoll没有前面两者的线性下降的性能问题，但是所有socket都很活跃的情况下，可能会有性能问题。

**3、消息传递方式**

select

内核需要将消息传递到用户空间，都需要内核拷贝动作

poll

同上

epoll

epoll通过内核和用户空间共享一块内存来实现的，利用mmap()文件映射内存加速与内核空间的消息传递。

**总结：**

综上，在选择select，poll，epoll时要根据具体的使用场合以及这三种方式的自身特点。

1、表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。

2、select低效是因为每次它都需要轮询。但低效也是相对的，视情况而定，也可通过良好的设计改善 


















# Linux     
## 1.Linux查看端口占用情况      
lsof -i:port(端口号)
## 2.Linux查看一个文件有多少行      
　wc命令的功能为统计指定文件中的字节数、字数、行数, 并将统计结果显示输出。      
- c 统计字节数。
- l 统计行数。
- w 统计字数。

如：wc -l filename 就是查看文件里有多少行       
wc -w filename 看文件里有多少个word。       
wc -L filename 文件里最长的那一行是多少个字。

## 3.列出所有进程的信息     
ps aux  # 列出所有进程的信息

## 4.查看占用CPU最多的进程      
top

## 5.查看当前系统的端口使用     
netstat -an     

## 6.查看文件指定行数内容       

```
     tail date.log           输出文件末尾的内容，默认10行

     tail -20  date.log      输出最后20行的内容

     tail -n -20  date.log   输出倒数第20行到文件末尾的内容

     tail -n +20  date.log   输出第20行到文件末尾的内容
     
     head date.log           输出文件开头的内容，默认10行

     head -15  date.log      输出开头15行的内容

     head -n +15 date.log    输出开头到第15行的内容

     head -n -15 date.log    输出开头到倒数第15行的内容

     sed -n "开始行，结束行p" 文件名    

     sed -n '70,75p' date.log   输出第70行到第75行的内容

     sed -n '6p;260,400p; ' 文件名    输出第6行 和 260到400行

     sed -n 5p 文件名      输出第5行

tail 和 head 加上 -n参数后 都代表输出到指定行数，tail 是指定行数到结尾，head是开头到指定行数

+数字 代表整数第几行， -数字代表倒数第几行
```

## 7.查看文件大小并排序、磁盘剩余可用空间大小     

* 按字节排序，按兆（M）加参数 ‘h’
```
du -s /usr/* | sort -rn   从大到小

du -s /usr/* | sort -n    从小到大


du -h #以K  M  G为单位显示，提高可读性

```

* 选择部分列出
```
du -s /usr/* | sort -rn | head     前面的10个

du -s /usr/* | sort -rn | tail     后面的10个

```

* 显示磁盘分区上可以使用的磁盘空间

```
df -h   #使用-h选项以KB、MB、GB的单位来显示，可读性高
```



## 8.孤儿进程与僵尸进程       

**孤儿进程**        

**一个父进程退出，而它的一个或多个子进程还在运行，那么这些子进程将成为孤儿进程**。

孤儿进程将被 init 进程（进程号为 1）所收养，并由 init 进程对它们完成状态收集工作。

**由于孤儿进程会被 init 进程收养，所以孤儿进程不会对系统造成危害**。
    
**僵尸进程**        

**一个子进程的进程描述符在子进程退出时不会释放，只有当父进程通过 wait() 或 waitpid() 获取了子进程信息后才会释放**。如果子进程退出，而父进程并没有调用 wait() 或 waitpid()，那么子进程的进程描述符仍然保存在系统中，这种进程称之为僵尸进程。

**僵尸进程通过 ps 命令显示出来的状态为 Z**（zombie）。

系统所能使用的进程号是有限的，**如果产生大量僵尸进程，将因为没有可用的进程号而导致系统不能产生新的进程**。

要**消灭系统中大量的僵尸进程，只需要将其父进程杀死，此时僵尸进程就会变成孤儿进程，从而被 init 进程所收**养，这样 init 进程就会释放所有的僵尸进程所占有的资源，从而结束僵尸进程。

**wait()**      


```
pid_t wait(int *status)    

```
 
**父进程调用 wait() 会一直阻塞，直到收到一个子进程退出的 SIGCHLD 信号，之后 wait() 函数会销毁子进程并返回**。

如果成功，返回被收集的子进程的进程 ID；如果调用进程没有子进程，调用就会失败，此时返回 -1，同时 errno 被置为 ECHILD。

参数 status 用来保存被收集的子进程退出时的一些状态，如果对这个子进程是如何死掉的毫不在意，只想把这个子进程消灭掉，可以设置这个参数为 NULL。


**waitpid()**       

```
pid_t waitpid(pid_t pid, int *status, int options)   

```

作用和 wait() 完全相同，但是多了两个可由用户控制的参数 pid 和 options。

**pid 参数指示一个子进程的 ID，表示只关心这个子进程退出的 SIGCHLD 信号**。如果 **pid=-1 时，那么和 wait() 作用相同，都是关心所有子进程退出的 SIGCHLD 信号**。

options 参数主要有 WNOHANG 和 WUNTRACED 两个选项，WNOHANG 可以使 waitpid() 调用变成非阻塞的，也就是说它会立即返回，父进程可以继续执行其它任务。

**SIGCHLD**     

当**一个子进程改变了它的状态时（停止运行，继续运行或者退出）**，有两件事会发生在父进程中：

* 得到 SIGCHLD 信号；
* waitpid() 或者 wait() 调用会返回。
其中**子进程发送的 SIGCHLD 信号包含了子进程的信息，比如进程 ID、进程状态、进程使用 CPU 的时间等**。

在**子进程退出时，它的进程描述符不会立即释放，这是为了让父进程得到子进程信息**，父进程通过 wait() 和 waitpid() 来获得一个已经退出的子进程的信息。



# 计算机网络
## 1.Http方法
链接：https://blog.csdn.net/wuyoudeyuer/article/details/80509901
```
1.GET:获取资源

GET方法用来请求URI识别的资源。指定的资源经服务器端解析后返回响应内容。也就是说，如果请求的资源是文本，那就保持原样返回。

请求	
GET /index.html HTTP/1.1
Host: www.hackr.cn	
响应：
返回index.html的页面资源

2.POST:传输实体主题

POST方法用来传输实体的主体。

请求	
POST /submit.cgi HTTP/1.1
Host:www.hackr.cn
Content-Length:1560	
响应：
返回submit.cgi接收数据的处理结果

3.PUT:传输文件

PUT方法用来传输文件。就像FTP协议的文件上传一样，要求在请求报文主体中包含文件的内容，然后保存到请求URI指定的位置。

请求	
POST /index.cgi HTTP/1.1
Host:www.hackr.cn
Content-Length:text/hml
Content-Length:1560
响应：
响应返回状态码204 No Content(比如：该html已存在从该服务器上)
 
4.HEAD：获取报文首部

HEAD方法和GET方法一样，只是不返回报文主体部分。用于确认URI的有效性及资源更新的日期时间等。

请求
HEAD /index.html HTTP/1.1
Host:www.hackr.cn	
响应：
返回index.html有关的响应首部

5.DELETE：删除文件

DELETE方法用来删除文件，是PUT的相反方法。DELETE方法按请求URL删除指定的资源。也不常用。

请求
DELETE /example.html HTTP/1.1
Host:www.hackr.cn	
响应：
响应返回状态码204 No Content(比如：该html已从该服务器上删除)

6.OPTIONS:询问支持的方法

OPTIONS方法用来查询针对请求URL指定的资源支持的方法。

请求
OPTIONS * HTTP/1.1
Host:www.hackr.cn	
响应：
HTTP/1.1 200 OK 
Allow:GET,POST,HEAD,OPTIONS

7.TRACE:追踪路径

TRACE方法是让Web服务器端将之前的请求通信环回给客户端方法。
客户端可以用TRACE方法查询发送出去的请求时怎样被加工修改的。
不常用，还容易引发XST攻击

请求	
TRACE / HTTP/1.1
Host:hackr.cn
Max-Forwards:2	
响应：
HTTP:/1.1 200 OK
Content-Type:message/http
Content-Length:1024

TRACE / HTTP/1.1
Host:hackr.cn
Max-Forwards:2

8.CONNECT:要求用隧道协议链接代理

CONNECT方法要求在与代理服务器通信时建立隧道，实现用隧道协议进行TCP通信。主要使用SSL和TSL协议把通信内容加密后经网络隧道传输。

请求	
CONNECT proxy.hackr.cn:8080 HTTP/1.1
响应：
Host:proxy.hacky.cn	HTTP/1.1 200 OK(之后进入网络隧道)
```

## 2.IP数据包如何判断是TCP还是UDP连接的？       
查看**IP包头中的协议字段**，如果该字段**数值为6就表示，里面的数据是TCP的。如果是17则表示是UDP的**。        

## 3.ping是哪个协议？ping的过程？












# MySQL     
## 1.为什么使用B+树？如何减少io次数？       


一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘I/O消耗，相对于内存存取，I/O存取的消耗要高几个数量级，**所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂度**。换句话说，索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数。而B-/+/*Tree，经过改进可以有效的利用系统对磁盘的块读取特性，**在读取相同磁盘块的同时，尽可能多的加载索引数据，来提高索引命中效率，从而达到减少磁盘IO的读取次数**。      

**1.B+树的磁盘读写代价更低**：**B+树的内部节点并没有指向关键字具体信息的指针，因此其内部节点相对B树更小**，如果把所有同一内部节点的关键字存放在同一盘块中，那么盘块所能容纳的关键字数量也越多，一次性读入内存的需要查找的关键字也就越多，相对IO读写次数就降低了。**B+树的索引页中全部是都是索引,这样一个数据页中能查询到很多索引降低了下一次去磁盘再拿索引页的可能性,这样就降低了磁盘的IO了**。       
**2.B+树的查询效率更加稳定**：由于非终结点并不是最终指向文件内容的结点，而只是叶子结点中关键字的索引。所以**任何关键字的查找必须走一条从根结点到叶子结点的路。所有关键字查询的路径长度相同，导致每一个数据的查询效率相当。**      
**3**.B树在提高了IO性能的同时并没有解决元素遍历的我效率低下的问题，正是为了解决这个问题，B+树应用而生。**B+树只需要去遍历叶子节点就可以实现整棵树的遍历。而且在数据库中基于范围的查询是非常频繁的，而B树不支持这样的操作或者说效率太低。**     
**4**.**B+树的特点，只有叶子节点存储数据。**        

## 2.数据库的三范式         

第一范式: 每个列都不可以再拆分.     
第二范式: 非主键列完全依赖于主键,而不能是依赖于主键的一部分.       
第三范式: 非主键列只依赖于主键,不依赖于其他非主键.

## 3.MySQL的几种日志文件        
链接：https://www.cnblogs.com/myseries/p/10728533.html

1：重做日志（redo log）

2：回滚日志（undo log）

3：二进制日志（binlog）

4：错误日志（errorlog）

5：慢查询日志（slow query log）

6：一般查询日志（general log）

7：中继日志（relay log）        

**一、重做日志（redo log）**        

**作用：**

　　确保事务的持久性。**redo日志记录事务执行后的状态，用来恢复未写入data file的已成功事务更新的数据。防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行重做，从而达到事务的持久性这一特性**。

**内容：**

　　**物理格式的日志，记录的是物理数据页面的修改的信息**，其redo log是顺序写入redo log file的物理文件中去的。

**什么时候产生：**

　　**事务开始之后就产生redo log，redo log的落盘并不是随着事务的提交才写入的，而是在事务的执行过程中，便开始写入redo log文件中**。

**什么时候释放：**

　　**当对应事务的脏页写入到磁盘之后，redo log的使命也就完成了，重做日志占用的空间就可以重用（被覆盖）**。

**对应的物理文件：**

　　默认情况下，对应的物理文件位于数据库的data目录下的ib_logfile1&ib_logfile2

　　innodb_log_group_home_dir 指定日志文件组所在的路径，默认./ ，表示在数据库的数据目录下。

　　innodb_log_files_in_group 指定重做日志文件组中文件的数量，默认2

**关于文件的大小和数量，由以下两个参数配置：**

　　innodb_log_file_size 重做日志文件的大小。

　　innodb_mirrored_log_groups 指定了日志镜像文件组的数量，默认1

**其他：**

　　很重要一点，**redo log是什么时候写盘的？前面说了是在事物开始之后逐步写盘的**。

　　之所以说重做日志是在事务开始之后逐步写入重做日志文件，而不一定是事务提交才写入重做日志缓存，**原因就是，重做日志有一个缓存区Innodb_log_buffer，Innodb_log_buffer的默认大小为8M(这里设置的16M),Innodb存储引擎先将重做日志写入innodb_log_buffer中**。     

然后会**通过以下三种方式将innodb日志缓冲区的日志刷新到磁盘**：

**1.Master Thread 每秒一次执行刷新Innodb_log_buffer到重做日志文件**。

**2.每个事务提交时会将重做日志刷新到重做日志文件**。

**3.当重做日志缓存可用空间 少于一半时，重做日志缓存被刷新到重做日志文件**。

由此可以看出，重做日志通过不止一种方式写入到磁盘，尤其是对于第一种方式，Innodb_log_buffer到重做日志文件是Master Thread线程的定时任务。

因此重做日志的写盘，并不一定是随着事务的提交才写入重做日志文件的，而是随着事务的开始，逐步开始的。

另外引用《MySQL技术内幕 Innodb 存储引擎》（page37）上的原话：

**即使某个事务还没有提交，Innodb存储引擎仍然每秒会将重做日志缓存刷新到重做日志文件**。

**这一点是必须要知道的，因为这可以很好地解释再大的事务的提交（commit）的时间也是很短暂的**。       


**二、回滚日志（undo log）**        

**作用：**

**保证数据的原子性，保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读**。

**内容：**

**逻辑格式的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于redo log的**。

**什么时候产生：**

**事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性**。

**什么时候释放：**

**当事务提交之后，undo log并不能立马被删除，而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间**。

**对应的物理文件：**

MySQL5.6之前，undo表空间位于共享表空间的回滚段中，共享表空间的默认的名称是ibdata，位于数据文件目录中。

MySQL5.6之后，undo表空间可以配置成独立的文件，但是提前需要在配置文件中配置，完成数据库初始化后生效且不可改变undo log文件的个数

如果初始化数据库之前没有进行相关配置，那么就无法配置成独立的表空间了。

**关于MySQL5.7之后的独立undo 表空间配置参数如下：**

innodb_undo_directory = /data/undospace/ –undo独立表空间的存放目录 innodb_undo_logs = 128 –回滚段为128KB innodb_undo_tablespaces = 4 –指定有4个undo log文件

如果undo使用的共享表空间，这个共享表空间中又不仅仅是存储了undo的信息，共享表空间的默认为与MySQL的数据目录下面，其属性由参数innodb_data_file_path配置。

**其他：**

**undo是在事务开始之前保存的被修改数据的一个版本，产生undo日志的时候，同样会伴随类似于保护事务持久化机制的redolog的产生**。

默认情况下undo文件是保持在共享表空间的，也即ibdatafile文件中，当数据库中发生一些大的事务性操作的时候，要生成大量的undo信息，全部保存在共享表空间中的。

**因此共享表空间可能会变的很大，默认情况下，也就是undo 日志使用共享表空间的时候，被“撑大”的共享表空间是不会也不能自动收缩的。因此，mysql5.7之后的“独立undo 表空间”的配置就显得很有必要了**。 

**三、二进制日志（binlog）：**      

**作用：**

**用于复制，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步**。        
用于数据库的基于时间点的还原。

**内容：**

**逻辑格式的日志，可以简单认为就是执行过的事务中的sql语句**。

但**又不完全是sql语句这么简单，而是包括了执行的sql语句（增删改）反向的信息，也就意味着delete对应着delete本身和其反向的insert；update对应着update执行前后的版本的信息；insert对应着delete和insert本身的信息**。

在使用mysqlbinlog解析binlog之后一些都会真相大白。

因此可以基于binlog做到类似于oracle的闪回功能，其实都是依赖于binlog中的日志记录。

**什么时候产生：**

**事务提交的时候，一次性将事务中的sql语句（一个事物可能对应多个sql语句）按照一定的格式记录到binlog中**。

**这里与redo log很明显的差异就是redo log并不一定是在事务提交的时候刷新到磁盘，redo log是在事务开始之后就开始逐步写入磁盘**。

因此**对于事务的提交，即便是较大的事务，提交（commit）都是很快的，但是在开启了bin_log的情况下，对于较大事务的提交，可能会变得比较慢一些**。

这是**因为binlog是在事务提交的时候一次性写入的造成的**，这些可以通过测试验证。

**什么时候释放：**

**binlog的默认是保持时间由参数expire_logs_days配置，也就是说对于非活动的日志文件，在生成时间超过expire_logs_days配置的天数之后，会被自动删除**。        

**对应的物理文件：**

配置文件的路径为log_bin_basename，binlog日志文件按照指定大小，当日志文件达到指定的最大的大小之后，进行滚动更新，生成新的日志文件。

对于每个binlog日志文件，通过一个统一的index文件来组织。     

**其他：**

**二进制日志的作用之一是还原数据库的，这与redo log很类似**，很多人混淆过，但是**两者有本质的不同**

**作用不同：redo log是保证事务的持久性的，是事务层面的，binlog作为还原的功能，是数据库层面的**（当然也可以精确到事务层面的），虽然都有还原的意思，但是其保护数据的层次是不一样的。

**内容不同：redo log是物理日志，是数据页面的修改之后的物理记录，binlog是逻辑日志，可以简单认为记录的就是sql语句**。

另外，**两者日志产生的时间，可以释放的时间，在可释放的情况下清理机制，都是完全不同的**。

**恢复数据时候的效率，基于物理日志的redo log恢复数据的效率要高于语句逻辑日志的binlog**。

**关于事务提交时，redo log和binlog的写入顺序，为了保证主从复制时候的主从一致**（当然也包括使用binlog进行基于时间点还原的情况），是要严格一致的，**MySQL通过两阶段提交过程来完成事务的一致性的**，也即redo log和binlog的一致性的，**理论上是先写redo log，再写binlog，两个日志都提交成功（刷入磁盘），事务才算真正的完成**。        

**四、错误日志**        

**错误日志记录着mysqld启动和停止,以及服务器在运行过程中发生的错误的相关信息**。在默认情况下，系统记录错误日志的功能是关闭的，错误信息被输出到标准错误输出。      
指定日志路径两种方法:       
编辑my.cnf 写入 log-error=[path]        
通过命令参数错误日志 mysqld_safe –user=mysql –log-error=[path] &


**五、普通查询日志 general query log**

**记录了服务器接收到的每一个查询或是命令，无论这些查询或是命令是否正确甚至是否包含语法错误，general log 都会将其记录下来** ，记录的格式为 {Time ，Id ，Command，Argument }。也正**因为mysql服务器需要不断地记录日志，开启General log会产生不小的系统开销。 因此，Mysql默认是把General log关闭的**。

查看日志的存放方式：show variables like ‘log_output’;       


**六、慢查询日志**           
**慢日志记录执行时间过长和没有使用索引的查询语句**，报错select、update、delete以及insert语句，**慢日志只会记录执行成功的语句**。      
1. 查看慢查询时间：      
show variables like “long_query_time”;默认10s       
2. 查看慢查询配置情况：         
show status like “%slow_queries%”;      
3. 查看慢查询日志路径：      
show variables like “%slow%”;       
4. 开启慢日志       
set global slow_query_log=1     



## 4.数据库调优连招     
链接：https://mp.weixin.qq.com/s/e0CqJG2-PCDgKLjQfh02tw         
https://www.cnblogs.com/-mrl/p/13230006.html

#### 排除缓存干扰       
因为在MySQL8.0之前我们的数据库是存在缓存这样的情况的，我之前就被坑过，因为存在缓存，我发现我sql怎么执行都是很快，当然第一次其实不快但是我没注意到，以至于上线后因为缓存经常失效，导致rt（Response time）时高时低。

后面就发现了是缓存的问题，我们在执行SQL的时候，记得加上SQL NoCache去跑SQL，这样跑出来的时间就是真实的查询时间了。

我说一下**为什么缓存会失效，而且是经常失效**。

**如果我们当前的MySQL版本支持缓存而且我们又开启了缓存，那每次请求的查询语句和结果都会以key-value的形式缓存在内存中的**，大家也看到我们的结构图了，**一个请求会先去看缓存是否存在，不存在才会走解析器**。

**缓存失效比较频繁的原因就是，只要我们一对表进行更新，那这个表所有的缓存都会被清空，其实我们很少存在不更新的表**，特别是我之前的电商场景，可能静态表可以用到缓存，但是我们都走大数据离线分析，缓存也就没用了。

大家如果是8.0以上的版本就不用担心这个问题，如果是8.0之下的版本，记得排除缓存的干扰。      

#### Explain        
最开始提到了用执行计划去分析，我想explain是大家SQL调优都会回答到的吧。

因为这基本上是写SQL的必备操作，那我现在问大家一个我去阿里面试被问过的一个问题：explain你记得哪些字段，分别有什么含义？     

那我再问大家一下，**你们认为统计这个统计的行数就是完全对的么？索引一定会走到最优索引么？**

当然我都这么问了，你们肯定也知道结果了，**行数只是一个接近的数字，不是完全正确的，索引也不一定就是走最优的，是可能走错的**。      

**MySQL中数据的单位都是页，MySQL又采用了采样统计的方法**，采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。

我们**数据是一直在变的，所以索引的统计信息也是会变的，会根据一个阈值，重新做统计**。

至于MySQL索引可能走错也很好理解，如果走A索引要扫描100行，B所有只要20行，但是他可能选择走A索引，你可能会想MySQL是不是有病啊，其实不是的。

**一般走错都是因为优化器在选择的时候发现，走A索引没有额外的代价，比如走B索引并不能直接拿到我们的值，还需要回到主键索引才可以拿到，多了一次回表的过程，这个也是会被优化器考虑进去的**。

他发现走A索引不需要回表，没有额外的开销，所以它选错了。

**如果是上面的统计信息错了，那简单，我们用analyze table tablename 就可以重新统计索引信息了**，所以在实践中，如果你发现explain的结果预估的rows值跟实际情况差距比较大，可以采用这个方法来处理。

**还有一个方法就是force index强制走正确的索引，或者优化SQL，最后实在不行，可以新建索引，或者删掉错误的索引**。        

#### 覆盖索引       

上面我提到了，可能需要回表这样的操作，那我们怎么能做到不回表呢？在自己的索引上就查到自己想要的，不要去主键索引查了。

覆盖索引

**如果在我们建立的索引上就已经有我们需要的字段，就不需要回表了**，在电商里面也是很常见的，我们需要去商品表通过各种信息查询到商品id，id一般都是主键。

由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。        

#### 联合索引       
还是商品表举例，我们需要根据他的名称，去查他的库存，假设这是一个很高频的查询请求，你会怎么建立索引呢？

大家可以思考上面的回表的消耗对SQL进行优化。

是的建立一个，名称和库存的联合索引，这样名称查出来就可以看到库存了，不需要查出id之后去回表再查询库存了，联合索引在我们开发过程中也是常见的，但是并不是可以一直建立的，大家要思考索引占据的空间。        
#### 最左匹配原则       
最左匹配原则就是指在联合索引中，如果你的 SQL 语句中用到了联合索引中的最左边的索引，那么这条 SQL 语句就可以利用这个联合索引去进行匹配。      

```
例如某表现有索引(a,b,c)：

select * from t where a=1 and b=1 and c =1;     #这样可以利用到定义的索引（a,b,c）,用上a,b,c

select * from t where a=1 and b=1;     #这样可以利用到定义的索引（a,b,c）,用上a,b

select * from t where a=1;     #这样也可以利用到定义的索引（a,b,c）,用上a

select * from t where b=1 and c=1;     #这样不可以利用到定义的索引（a,b,c）

select * from t where a=1 and c=1;     #这样可以利用到定义的索引（a,b,c），但只用上a索引，b,c索引用不到
```
值得注意的是，**当遇到范围查询(>、<、between、like)就会停止匹配**。

```
select * from t where a=1 and b>1 and c =1; #这样a,b可以用到（a,b,c），c索引用不到 
这条语句只有 a,b 会用到索引，c 都不能用到索引。这个原因可以从联合索引的结构来解释。

但是如果是建立(a,c,b)联合索引，则a,b,c都可以使用索引，因为优化器会自动改写为最优查询语句

select * from t where a=1 and b >1 and c=1;  #如果是建立(a,c,b)联合索引，则a,b,c都可以使用索引
#优化器改写为
select * from t where a=1 and c=1 and b >1;
```
这也是最左前缀原理的一部分，**索引index1:(a,b,c)，只会走a、a,b、a,b,c 三种类型的查询，其实这里说的有一点问题，a,c也走，但是只走a字段索引，不会走c字段**。

另外还有一个特殊情况说明下，select * from table where a = '1' and b > ‘2’ and c='3' 这种类型的也只会有 a与b 走索引，c不会走。

**像select * from table where a = '1' and b > ‘2’ and c='3' 这种类型的sql语句，在a、b走完索引后，c肯定是无序了，所以c就没法走索引，数据库会觉得还不如全表扫描c字段来的快**。

以**index （a,b,c）为例建立这样的索引相当于建立了索引a、ab、abc三个索引。一个索引顶三个索引当然是好事，毕竟每多一个索引，都会增加写操作的开销和磁盘空间的开销**。        

**最左匹配原则的原理**      

我们都知道索引的底层是一颗 B+ 树，那么**联合索引当然还是一颗 B+ 树，只不过联合索引的健值数量不是一个，而是多个。构建一颗 B+ 树只能根据一个值来构建**，因此数据库依据联合索引最左的字段来构建 B+ 树。        

一个**形如(a,b,c)联合索引的 b+ 树，其中的非叶子节点存储的是第一个关键字的索引 a，而叶子节点存储的是三个关键字的数据**。这里可以看出 a 是有序的，而 b，c 都是无序的。**但是当在 a 相同的时候，b 是有序的，b 相同的时候，c 又是有序的**。        

通过**对联合索引的结构的了解，那么就可以很好的了解为什么最左匹配原则中如果遇到范围查询就会停止了**。以 select * from t where a=5 and b>0 and c =1; #这样a,b可以用到（a,b,c），c不可以 为例子，当查询到 b 的值以后（这是一个范围值），c 是无序的。所以就不能根据联合索引来确定到低该取哪一行。

**总结**：      
1.在 InnoDB 中联合索引只有先确定了前一个（左侧的值）后，才能确定下一个值。**如果有范围查询的话，那么联合索引中使用范围查询的字段后的索引在该条 SQL 中都不会起作用**。     

2.值得注意的是，in 和 = 都可以乱序，比如有索引（a,b,c），语句 select * from t where c =1 and a=1 and b=1，这样的语句也可以用到最左匹配，**因为 MySQL 中有一个优化器，他会分析 SQL 语句，将其优化成索引可以匹配的形式，即 select * from t where a =1 and b=1 and c=1**。

#### 索引下推       
select * from itemcenter where name like '张%' and size=22 and age = 20;       
所以这个语句在搜索索引树的时候，只能用 “敖”，找到第一个满足条件的记录ID1，当然，这还不错，总比全表扫描要好。**在MySQL 5.6之前，只能从ID1开始一个个回表，到主键索引上找出数据行，再对比字段值**。即先找满足张某的数据，再从中筛选size=20，age=20的数据。        

**而MySQL 5.6 引入的索引下推优化（index condition pushdown)， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数**。即先用size=20和age=20过滤掉不满足的数据，再用张某进行匹配！减少回表次数。      

#### 唯一索引普通索引选择难题       
这个在我的面试视频里面其实问了好几次了，核心是需要回答到change buffer，那**change buffer又是个什么东西呢**？

**当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页了**。

**在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行change buffer中与这个页有关的操作，通过这种方式就能保证这个数据逻辑的正确性**。

需要说明的是，虽然名字叫作change buffer，实际上它是可以持久化的数据。也就是说，**change buffer在内存中有拷贝，也会被写入到磁盘上**。

**将change buffer中的操作应用到原数据页，得到最新结果的过程称为merge**。

**除了访问这个数据页会触发merge外，系统有后台线程会定期merge。在数据库正常关闭（shutdown）的过程中，也会执行merge操作**。     

显然，**如果能够将更新操作先记录在change buffer，减少读磁盘，语句的执行速度会得到明显的提升。而且，数据读入内存是需要占用buffer pool的，所以这种方式还能够避免占用内存，提高内存利用率**。

那么，**什么条件下可以使用change buffer呢**？

**对于唯一索引来说**，所有的更新操作都要**先判断这个操作是否违反唯一性约束**。

**要判断表中是否存在这个数据，而这必须要将数据页读入内存才能判断，如果都已经读入到内存了，那直接更新内存会更快，就没必要使用change buffer了**。

因此，唯一索引的更新就不能使用change buffer，**实际上也只有普通索引可以使用**。

**change buffer用的是buffer pool里的内存，因此不能无限增大，change buffer的大小，可以通过参数innodb_change_buffer_max_size来动态设置**，这个参数设置为50的时候，表示change buffer的大小最多只能占用buffer pool的50%。

**将数据从磁盘读入内存涉及随机IO的访问，是数据库里面成本最高的操作之一，change buffer因为减少了随机磁盘访问**，所以对更新性能的提升是会很明显的。

**change buffer的使用场景**     

因为merge的时候是真正进行数据更新的时刻，而**change buffer的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做merge之前，change buffer记录的变更越多**（也就是这个页面上要更新的次数越多），**收益就越大**。

因此，**对于写多读少的业务来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好**，这种业务模型常见的就是账单类、日志类的系统。

反过来，**假设一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在change buffer，但之后由于马上要访问这个数据页，会立即触发merge过程。这样随机访问IO的次数不会减少，反而增加了change buffer的维护代价**，所以，对于这种业务模式来说，change buffer反而起到了副作用。        

#### 前缀索引       

我们存在邮箱作为用户名的情况，每个人的邮箱都是不一样的，那我们是不是可以在邮箱上建立索引，但是邮箱这么长，我们怎么去建立索引呢？

**MySQL是支持前缀索引的，也就是说，你可以定义字符串的一部分作为索引。默认地，如果你创建索引的语句不指定前缀长度，那么索引就会包含整个字符串**。

我们**是否可以建立一个区分度很高的前缀索引，达到优化和节约空间的目的呢**？

使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。

上面说过覆盖索引了，覆盖索引是不需要回表的，但是**前缀索引，即使你的联合索引已经包涵了相关信息，他还是会回表，因为他不确定你到底是不是一个完整的信息**，就算你是www.zhangsan@qq.com一个完整的邮箱去查询，他还是不知道你是否是完整的，所以他需要回表去判断一下。

下面这个也是我在阿里面试面试官问的，**很长的字段，想做索引我们怎么去优化他呢**？

因为**存在一个磁盘占用的问题，索引选取的越长，占用的磁盘空间就越大，相同的数据页能放下的索引值就越少，搜索的效率也就会越低**。

我当时就回答了一个hash，把字段hash为另外一个字段存起来，每次校验hash就好了，hash的索引也不大。

我们都知道只要区分度过高，都可以，那我们可以采用倒序，或者删减字符串这样的情况去建立我们自己的区分度，**不过大家需要注意的是，调用函数也是一次开销哟**。

就比如本来是www.zhangsan@qq,com 其实前面的www.基本上是没任何区分度的，所有人的邮箱都是这么开头的，你一搜一大堆出来，放在索引还浪费内存，你可以substring()函数截取掉前面的，然后建立索引。

我们所有人的身份证都是区域开头的，同区域的人很多，那怎么做良好的区分呢？REVERSE（）函数翻转一下，区分度可能就高了。

这些操作都用到了函数，我就说一下函数的坑。      

#### 条件字段函数操作       
如果对日期字段操作，浮点字符操作等等，大家需要注意的是，如果对字段做了函数计算，就用不上索引了，这是MySQL的规定。

**对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能**。

需要注意的是，**优化器并不是要放弃使用这个索引**。

这个时候大家可以用一些取巧的方法，比如 select * from tradelog where id + 1 = 10000 就走不上索引，select * from tradelog where id = 9999就可以。        

**隐式类型转换**        
select * from t where id = 1

如果id是字符类型的，1是数字类型的，你用explain会发现走了全表扫描，根本用不上索引，为啥呢？

因为MySQL底层会对你的比较进行转换，相当于加了 **CAST( id AS signed int) 这样的一个函数，上面说过函数会导致走不上索引**。       

#### 隐式字符编码转换       
**如果两个表的字符集不一样，一个是utf8mb4，一个是utf8，因为utf8mb4是utf8的超集，所以一旦两个字符比较，就会转换为utf8mb4再比较**。

转换的过程相当于加了CONVERT(id USING utf8mb4)函数，那又回到上面的问题了，**用到函数就用不上索引了**。

还有大家一会可能会遇到mysql突然卡顿的情况，那可能是MySQLflush了。

#### flush      

**redo log**大家都知道，也就是我们对数据库操作的日志，**它是在内存中的，每次操作一旦写了redo log就会立马返回结果，但是这个redo log总会找个时间去更新到磁盘，这个操作就是flush**。

在更新之前，**当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”**。

内存数据写入到磁盘后，**内存和磁盘上的数据页的内容就一致了，称为“干净页“**。       

**那什么时候会flush呢**？

**1.InnoDB的redo log写满了**，这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写。

**2.系统内存不足**，当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。**如果淘汰的是“脏页”，就要先将脏页写到磁盘**。

你一定会说，这时候难道不能直接把内存淘汰掉，下次需要请求的时候，从磁盘读入数据页，然后拿redo log出来应用不就行了？

这里其实是从性能考虑的，**如果刷脏页一定会写盘，就保证了每个数据页有两种状态**：

**一种是内存里存在，内存里就肯定是正确的结果，直接返回；        
另一种是内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回。这样的效率最高**。      
**3.MySQL认为系统“空闲”的时候**，只要有机会就刷一点“脏页”。

**4.MySQL正常关闭**，这时候，MySQL会把内存的脏页都flush到磁盘上，这样下次MySQL启动的时候，就可以直接从磁盘上读数据，启动速度会很快。
        
那我们**怎么做才能把握flush的时机呢**？

**Innodb刷脏页控制策略，我们每个电脑主机的io能力是不一样的，你要正确地告诉InnoDB所在主机的IO能力**，这样InnoDB才能知道需要全力刷脏页的时候，可以刷多快。

这就要**用到innodb_io_capacity这个参数了，它会告诉InnoDB你的磁盘能力，这个值建议设置成磁盘的IOPS**，磁盘的IOPS可以通过fio这个工具来测试。

正确地设置innodb_io_capacity参数，可以有效的解决这个问题。

这中间有个有意思的点，刷脏页的时候，旁边如果也是脏页，会一起刷掉的，并且如果周围还有脏页，这个连带责任制会一直蔓延，这种情况其实在机械硬盘时代比较好，一次IO就解决了所有问题，但是现在都是固态硬盘了，innodb_flush_neighbors=0这个参数可以不产生连带制，在MySQL 8.0中，innodb_flush_neighbors参数的默认值已经是0了。


## 5.一条SQL语句执行很慢的原因      
链接：https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485185&idx=1&sn=66ef08b4ab6af5757792223a83fc0d45&chksm=cea248caf9d5c1dc72ec8a281ec16aa3ec3e8066dbb252e27362438a26c33fbe842b0e0adf47&token=79317275&lang=zh_CN#rd     

**一、分类讨论**

1、大多数情况是正常的，只是偶尔会出现很慢的情况。       
2、在数据量不变的情况下，这条SQL语句一直以来都执行的很慢。      

**二、针对偶尔很慢的情况**      

**1.数据库在刷新脏页（flush）**     

当我们要往数据库插入一条数据、或者要更新一条数据的时候，我们知道数据库会在内存中把对应字段的数据更新了，但是更新之后，这些更新的字段并不会马上同步持久化到磁盘中去，而是把这些更新的记录写入到 redo log 日记中去，等到空闲的时候，在通过 redo log 里的日记把最新的数据同步到磁盘中去。        

**刷脏页有下面4种场景**（后两种不用太关注“性能”问题）：

**1.redolog写满了**：redo log 里的容量是有限的，如果数据库一直很忙，更新又很频繁，这个时候 redo log 很快就会被写满了，这个时候就没办法等到空闲的时候再把数据同步到磁盘的，只能暂停其他操作，全身心来把数据同步到磁盘中去的，而这个时候，就会导致我们平时正常的SQL语句突然执行的很慢，所以说，数据库在在同步数据到磁盘的时候，就有可能导致我们的SQL语句执行的很慢了。

**2.内存不够用了**：如果一次查询较多的数据，恰好碰到所查数据页不在内存中时，需要申请内存，而此时恰好内存不足的时候就需要淘汰一部分内存数据页，如果是干净页，就直接释放，如果恰好是脏页就需要刷脏页。

**3.MySQL 认为系统“空闲”的时候**：这时系统没什么压力。

**4.MySQL 正常关闭的时候**：这时候，MySQL 会把内存的脏页都 flush 到磁盘上，这样下次 MySQL 启动的时候，就可以直接从磁盘上读数据，启动速度会很快。      

**2.拿不到锁**      

我们**要执行的这条语句，刚好这条语句涉及到的表，别人在用，并且加锁了，我们拿不到锁，只能慢慢等待别人释放锁了。或者，表没有加锁，但要使用到的某个一行被加锁了**，这个时候，我也没办法啊。

如果要**判断是否真的在等待锁，我们可以用 show processlist这个命令来查看当前的状态哦**，这里我要提醒一下，有些命令最好记录一下。

**三、针对一直都很慢情况**      

**1.索引没用上**        

select * from t where 100 <c and c < 100000;        

**（1）、字段没有索引**     
刚好你的 c 字段上没有索引，那么抱歉，只能走全表扫描了，你就体验不会索引带来的乐趣了，所以，这回导致这条查询语句很慢。          

**（2）、字段有索引，但是却没用索引**          
好吧，这个时候你给 c 这个字段加上了索引，然后又查询了一条语句

select * from t where c - 1 = 1000;
我想问大家一个问题，这样子在查询的时候会用索引查询吗？

答是不会，如果我们在字段的左边做了运算，那么很抱歉，在查询的时候，就不会用上索引了，所以呢，大家要注意这种字段上有索引，但由于自己的疏忽，导致系统没有使用索引的情况了。

正确的查询应该如下

select * from t where c = 1000 + 1;
        
**（3）、函数操作导致没用上索引**       
如果我们在查询的时候，对字段进行了函数操作，也是会导致没有用上索引的，例如

select * from t where pow(c,2) = 1000;
这里我只是做一个例子，假设函数 pow 是求 c 的 n 次方，实际上可能并没有 pow(c,2)这个函数。其实这个和上面在左边做运算也是很类似的。

所以呢，一条语句执行都很慢的时候，可能是该语句没有用上索引了，不过具体是啥原因导致没有用上索引的呢，你就要会分析了，我上面列举的三个原因，应该是出现的比较多的吧。      

**2.数据库自己选错索引**        

我们在进行查询操作的时候，例如

select * from t where 100 < c and c < 100000;
我们知道，主键索引和非主键索引是有区别的，主键索引存放的值是整行字段的数据，而非主键索引上存放的值不是整行字段的数据，而且存放主键字段的值。

也就是说，我们如果走 c 这个字段的索引的话，最后会查询到对应主键的值，然后，再根据主键的值走主键索引，查询到整行数据返回。

好吧扯了这么多，其实我就是想告诉你，**就算你在 c 字段上有索引，系统也并不一定会走 c 这个字段上的索引，而是有可能会直接扫描扫描全表**，找出所有符合 100 < c and c < 100000 的数据。       

**为什么会这样？**      
其实是这样的，**系统在执行这条语句的时候，会进行预测：究竟是走 c 索引扫描的行数少，还是直接扫描全表扫描的行数少呢？显然，扫描行数越少当然越好了，因为扫描行数越少，意味着I/O操作的次数越少**。

如果是扫描全表的话，那么扫描的次数就是这个表的总行数了，假设为 n；而如果走索引 c 的话，我们通过索引 c 找到主键之后，还得再通过主键索引来找我们整行的数据，也就是说，需要走两次索引。而且，我们也不知道符合 100 c < and c < 10000 这个条件的数据有多少行，万一这个表是全部数据都符合呢？这个时候意味着，走 c 索引不仅扫描的行数是 n，同时还得每行数据走两次索引。       

**所以呢，系统是有可能走全表扫描而不走索引的。那系统是怎么判断呢**？          
**判断来源于系统的预测，也就是说，如果要走 c 字段索引的话，系统会预测走 c 字段索引大概需要扫描多少行。如果预测到要扫描的行数很多，它可能就不走索引而直接扫描全表了**。      

**系统是通过索引的区分度来判断的，一个索引上不同的值越多，意味着出现相同数值的索引越少，意味着索引的区分度越高**。我们也**把区分度称之为基数，即区分度越高，基数越大**。所以呢，基数越大，意味着符合 100 < c and c < 10000 这个条件的行数越少。

所以呢，**一个索引的基数越大，意味着走索引查询越有优势**。      

那么问题来了，**怎么知道这个索引的基数呢**？

系统当然是不会遍历全部来获得一个索引的基数的，代价太大了，**索引系统是通过遍历部分数据，也就是通过采样的方式，来预测索引的基数的**。

重点的来了，**采样，那就有可能出现失误的情况，也就是说，c 这个索引的基数实际上是很大的，但是采样的时候，却很不幸，把这个索引的基数预测成很小**。例如你采样的那一部分数据刚好基数很小，然后就误以为索引的基数很小。**然后就呵呵，系统就不走 c 索引了，直接走全部扫描了**。

得出结论：**由于统计的失误，导致系统没有走索引，而是走了全表扫描，而这，也是导致我们 SQL 语句执行的很慢的原因**。     

系统判断是否走索引，扫描行数的预测其实只是原因之一，这条查询语句是否需要使用使用临时表、是否需要排序等也是会影响系统的选择的。        

```
不过呢，我们有时候也可以通过强制走索引的方式来查询，例如

select * from t force index(a) where c < 100 and c < 100000;
我们也可以通过

show index from t;
来查询索引的基数和实际是否符合，如果和实际很不符合的话，我们可以重新来统计索引的基数，可以用这条命令

analyze table t;
来重新统计分析。
```
**既然会预测错索引的基数，这也意味着，当我们的查询语句有多个索引的时候，系统有可能也会选错索引哦**，这也可能是 SQL 执行的很慢的一个原因。      
        

## 5.B+树的插入过程     
链接：https://www.cnblogs.com/nullzx/p/8729425.html     
（删除过程也包含在此链接中）

1）若**为空树，创建一个叶子结点，然后将记录插入其中**，此时这个叶子结点也是根结点，插入操作结束。

2）针对**叶子类型**结点：根据key值找到叶子结点，向这个叶子结点插入记录。**插入后，若当前结点key的个数小于等于m-1，则插入结束。否则将这个叶子结点分裂成左右两个叶子结点，左叶子结点包含前m/2个记录，右结点包含剩下的记录，将第m/2+1个记录的key进位到父结点中**（父结点一定是索引类型结点），**进位到父结点的key左孩子指针向左结点,右孩子指针向右结点**。将当前结点的指针指向父结点，然后执行第3步。

3）针对**索引类型**结点：若**当前结点key的个数小于等于m-1，则插入结束**。**否则，将这个索引类型结点分裂成两个索引结点，左索引结点包含前(m-1)/2个key，右结点包含m-(m-1)/2个key，将第m/2个key进位到父结点中，进位到父结点的key左孩子指向左结点, 进位到父结点的key右孩子指向右结点**。将当前结点的指针指向父结点，然后重复第3步。

## 5.主键和索引的区别       
链接：https://blog.csdn.net/a200822146085/article/details/88825443
        

主键是一种约束，唯一索引是一种索引，两者在本质上是不同的。

1、主键创建后一定包含一个唯一性索引，唯一性索引并不一定就是主键。      

2、唯一性索引列允许空值，而主键列不允许为空值。     

3、主键可以被其他表引用为外键，而唯一索引不能。     

4、一个表最多只能创建一个主键，但可以创建多个唯一索引。     

5、主键更适合那些不容易更改的唯一标识，如自动递增列、身份证号等。







## 高并发架构       
### 消息队列        

**为什么使用消息队列？消息队列有什么优点和缺点？Kafka、ActiveMQ、RabbitMQ、RocketMQ 都有什么区别，以及适合哪些场景？**     

**一、为什么使用消息队列？**        
其实就是问问你消息队列都有哪些使用场景，然后你项目里具体是什么场景，说说你在这个场景里用消息队列是什么？      
先说一下消息队列常见的使用场景吧，其实场景有很多，但是比较核心的有 3 个：**解耦、异步、削峰**。        

**解耦：**      
A 系统发送数据到 BCD 三个系统，通过接口调用发送。如果 E 系统也要这个数据呢？那如果 C 系统现在不需要了呢？A 系统负责人几乎崩溃......        

在这个场景中，A 系统跟其它各种乱七八糟的系统严重耦合，A 系统产生一条比较关键的数据，很多系统都需要 A 系统将这个数据发送过来。A 系统要时时刻刻考虑 BCDE 四个系统如果挂了该咋办？要不要重发，要不要把消息存起来？头发都白了啊！

如果使用 MQ，A 系统产生一条数据，发送到 MQ 里面去，哪个系统需要数据自己去 MQ 里面消费。如果新系统需要数据，直接从 MQ 里消费即可；如果某个系统不需要这条数据了，就取消对 MQ 消息的消费即可。这样下来，A 系统压根儿不需要去考虑要给谁发送数据，不需要维护这个代码，也不需要考虑人家是否调用成功、失败超时等情况。      
总结：**通过一个 MQ，Pub/Sub 发布订阅消息这么一个模型，A 系统就跟其它系统彻底解耦了**。      

**异步：**          
再来看一个场景，A 系统接收一个请求，需要在自己本地写库，还需要在 BCD 三个系统写库，自己本地写库要 3ms，BCD 三个系统分别写库要 300ms、450ms、200ms。最终请求总延时是 3 + 300 + 450 + 200 = 953ms，接近 1s。用户通过浏览器发起请求，等待个 1s，这几乎是不可接受的。        

一般互联网类的企业，对于用户直接的操作，一般要求是每个请求都必须在 200 ms 以内完成，对用户几乎是无感知的。

如果使用 MQ，那么 A 系统连续发送 3 条消息到 MQ 队列中，假如耗时 5ms，A 系统从接受一个请求到返回响应给用户，总时长是 3 + 5 = 8ms，对于用户而言，其实感觉上就是点个按钮，8ms 以后就直接返回了，爽！网站做得真好，真快！      

**削峰：**      
每天 0:00 到 12:00，A 系统风平浪静，每秒并发请求数量就 50 个。结果每次一到 12:00 ~ 13:00 ，每秒并发请求数量突然会暴增到 5k+ 条。但是系统是直接基于 MySQL 的，大量的请求涌入 MySQL，每秒钟对 MySQL 执行约 5k 条 SQL。

一般的 MySQL，扛到每秒 2k 个请求就差不多了，如果每秒请求到 5k 的话，可能就直接把 MySQL 给打死了，导致系统崩溃，用户也就没法再使用系统了。

但是高峰期一过，到了下午的时候，就成了低峰期，可能也就 1w 的用户同时在网站上操作，每秒中的请求数量可能也就 50 个请求，对整个系统几乎没有任何的压力。      

如果使用 MQ，每秒 5k 个请求写入 MQ，A 系统每秒钟最多处理 2k 个请求，因为 MySQL 每秒钟最多处理 2k 个。A 系统从 MQ 中慢慢拉取请求，每秒钟就拉取 2k 个请求，不要超过自己每秒能处理的最大请求数量就 ok，这样下来，哪怕是高峰期的时候，A 系统也绝对不会挂掉。而 MQ 每秒钟 5k 个请求进来，就 2k 个请求出去，结果就导致在中午高峰期（1 个小时），可能有几十万甚至几百万的请求积压在 MQ 中。

这个短暂的高峰期积压是 ok 的，因为高峰期过了之后，每秒钟就 50 个请求进 MQ，但是 A 系统依然会按照每秒 2k 个请求的速度在处理。所以说，只要高峰期一过，A 系统就会快速将积压的消息给解决掉。      

**二、消息队列的优缺点**        

优点上面已经说了，就是在**特殊场景下有其对应的好处，解耦、异步、削峰**。

缺点有以下几个：

**系统可用性降低**      
系统引入的外部依赖越多，越容易挂掉。本来你就是 A 系统调用 BCD 三个系统的接口就好了，ABCD 四个系统还好好的，没啥问题，你偏加个 MQ 进来，万一 MQ 挂了咋整？MQ 一挂，整套系统崩溃，你不就完了？**如何保证消息队列的高可用？**

**系统复杂度提高**      
硬生生加个 MQ 进来，你**怎么保证消息没有重复消费？怎么处理消息丢失的情况？怎么保证消息传递的顺序性？**

**一致性问题**      
A 系统处理完了直接返回成功了，人都以为你这个请求就成功了；但是问题是，要是 BCD 三个系统那里，BD 两个系统写库成功了，结果 C 系统写库失败了，咋整？你这数据就不一致了。

所以消息队列实际是一种非常复杂的架构，你引入它有很多好处，但是也得针对它带来的坏处做各种额外的技术方案和架构来规避掉，做好之后，你会发现，妈呀，系统复杂度提升了一个数量级，也许是复杂了 10 倍。但是关键时刻，用，还是得用的。      


**三、Kafka、ActiveMQ、RabbitMQ、RocketMQ 有什么优缺点？**      

**RabbitMQ**，但是确实 erlang 语言阻止了大量的 Java 工程师去深入研究和掌控它，对公司而言，几乎处于不可控的状态，但是确实人家是开源的，比较稳定的支持，活跃度也高；

不过现在确实越来越多的公司会去用 **RocketMQ**，确实很不错，毕竟是阿里出品，但社区可能有突然黄掉的风险（目前 RocketMQ 已捐给 Apache，但 GitHub 上的活跃度其实不算高）对自己公司技术实力有绝对自信的，推荐用 RocketMQ，否则回去老老实实用 RabbitMQ 吧，人家有活跃的开源社区，绝对不会黄。

所以中小型公司，技术实力较为一般，技术挑战不是特别高，用 RabbitMQ 是不错的选择；大型公司，基础架构研发实力较强，用 RocketMQ 是很好的选择。

如果是大数据领域的实时计算、日志采集等场景，用 **Kafka** 是业内标准的，绝对没问题，社区活跃度很高，绝对不会黄，何况几乎是全世界这个领域的事实性规范。



**如何保证消息队列的高可用？**      

**一、RabbitMQ 的高可用性**         

RabbitMQ 是比较有代表性的，因为是**基于主从（非分布式）做高可用性的**，我们就以 RabbitMQ 为例子讲解第一种 MQ 的高可用性怎么实现。

RabbitMQ 有三种模式：**单机模式、普通集群模式、镜像集群模式**。     
**单机模式**        

单机模式，就是 Demo 级别的，一般就是你本地启动了玩玩儿的😄，没人生产用单机模式。

**普通集群模式（无高可用性）**      

普通集群模式，意思就是**在多台机器上启动多个 RabbitMQ 实例，每个机器启动一个。你创建的 queue，只会放在一个 RabbitMQ 实例上，但是每个实例都同步 queue 的元数据**（元数据可以认为是 queue 的一些配置信息，通过元数据，可以找到 queue 所在实例）。你消费的时候，实际上如果连接到了另外一个实例，那么那个实例会从 queue 所在实例上拉取数据过来。     

这种方式确实很麻烦，也不怎么好，**没做到所谓的分布式**，就是个普通集群。因为这**导致你要么消费者每次随机连接一个实例然后拉取数据，要么固定连接那个 queue 所在实例消费数据，前者有数据拉取的开销，后者导致单实例性能瓶颈**。

而且如果那个放 queue 的实例宕机了，会导致接下来其他实例就无法从那个实例拉取，如果你开启了消息持久化，让 RabbitMQ 落地存储消息的话，消息不一定会丢，得等这个实例恢复了，然后才可以继续从这个 queue 拉取数据。

所以这个事儿就比较尴尬了，这就**没有什么所谓的高可用性，这方案主要是提高吞吐量的**，就是说让集群中多个节点来服务某个 queue 的读写操作。        

**镜像集群模式（高可用性）**        

这种模式，才是所谓的 RabbitMQ 的高可用模式。跟普通集群模式不一样的是，在**镜像集群模式下，你创建的 queue，无论元数据还是 queue 里的消息都会存在于多个实例上**，就是说，**每个 RabbitMQ 节点都有这个 queue 的一个完整镜像，包含 queue 的全部数据的意思**。**然后每次你写消息到 queue 的时候，都会自动把消息同步到多个实例的 queue 上**。       

那么如何开启这个镜像集群模式呢？其实很简单，RabbitMQ 有很好的管理控制台，就是在后台新增一个策略，这个策略是镜像集群模式的策略，指定的时候是可以要求数据同步到所有节点的，也可以要求同步到指定数量的节点，再次创建 queue 的时候，应用这个策略，就会自动将数据同步到其他的节点上去了。

这样的话，**好处在于，你任何一个机器宕机了，没事儿，其它机器（节点）还包含了这个 queue 的完整数据，别的 consumer 都可以到其它节点上去消费数据**。**坏处**在于，**第一，这个性能开销也太大了吧，消息需要同步到所有机器上，导致网络带宽压力和消耗很重！第二，不是分布式的，就没有扩展性可言了，如果某个 queue 负载很重，你加机器，新增的机器也包含了这个 queue 的所有数据，并没有办法线性扩展你的 queue**。你想，如果这个 queue 的数据量很大，大到这个机器上的容量无法容纳了，此时该怎么办呢？      

**二、Kafka 的高可用性**        

Kafka 一个最基本的架构认识：**由多个 broker 组成，每个 broker 是一个节点；你创建一个 topic，这个 topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据**。

这就是**天然的分布式消息队列，就是说一个 topic 的数据，是分散放在多个机器上的，每个机器就放一部分数据**。

实际上 RabbitMQ 之类的，并不是分布式消息队列，它就是传统的消息队列，只不过提供了一些集群、HA(High Availability, 高可用性) 的机制而已，**因为无论怎么玩儿，RabbitMQ 一个 queue 的数据都是放在一个节点里的，镜像集群下，也是每个节点都放这个 queue 的完整数据**。

Kafka 0.8 以前，是没有 HA 机制的，就是任何一个 broker 宕机了，那个 broker 上的 partition 就废了，没法写也没法读，没有什么高可用性可言。

比如说，我们假设创建了一个 topic，指定其 partition 数量是 3 个，分别在三台机器上。但是，如果第二台机器宕机了，会导致这个 topic 的 1/3 的数据就丢了，因此这个是做不到高可用的。     

**Kafka 0.8 以后，提供了 HA 机制**，就是 replica（复制品） 副本机制。**每个 partition 的数据都会同步到其它机器上，形成自己的多个 replica 副本。所有 replica 会选举一个 leader 出来，那么生产和消费都跟这个 leader 打交道，然后其他 replica 就是 follower。写的时候，leader 会负责把数据同步到所有 follower 上去，读的时候就直接读 leader 上的数据即可**。只能读写 leader？很简单，要是你可以随意读写每个 follower，那么就要 care 数据一致性的问题，系统复杂度太高，很容易出问题。**Kafka 会均匀地将一个 partition 的所有 replica 分布在不同的机器上，这样才可以提高容错性**。            

这么搞，**就有所谓的高可用性了，因为如果某个 broker 宕机了，没事儿，那个 broker上面的 partition 在其他机器上都有副本的。如果这个宕机的 broker 上面有某个 partition 的 leader，那么此时会从 follower 中重新选举一个新的 leader 出来，大家继续读写那个新的 leader 即可**。这就有所谓的高可用性了。

**写数据的时候，生产者就写 leader，然后 leader 将数据落地写本地磁盘，接着其他 follower 自己主动从 leader 来 pull 数据。一旦所有 follower 同步好数据了，就会发送 ack 给 leader，leader 收到所有 follower 的 ack 之后，就会返回写成功的消息给生产者**。（当然，这只是其中一种模式，还可以适当调整这个行为）

**消费的时候，只会从 leader 去读，但是只有当一个消息已经被所有 follower 都同步成功返回 ack 的时候，这个消息才会被消费者读到**。
        
            

**如何保证消息不被重复消费？或者说，如何保证消息消费的幂等性？**       

首先，比如 RabbitMQ、RocketMQ、Kafka，都有可能会出现消息重复消费的问题，正常。因为这问题通常不是 MQ 自己保证的，是由我们开发来保证的。挑一个 Kafka 来举个例子，说说怎么重复消费吧。

**Kafka 实际上有个 offset 的概念，就是每个消息写进去，都有一个 offset，代表消息的序号，然后 consumer 消费了数据之后，每隔一段时间（定时定期），会把自己消费过的消息的 offset 提交一下，表示“我已经消费过了，下次我要是重启啥的，你就让我继续从上次消费到的 offset 来继续消费吧”**。

但是**凡事总有意外**，比如我们之前生产经常遇到的，就是你有时候重启系统，看你怎么重启了，如果碰到点着急的，**直接 kill 进程了，再重启。这会导致 consumer 有些消息处理了，但是没来得及提交 offset，尴尬了。重启之后，少数消息会再次消费一次**。

举个栗子。

有这么个场景。数据 1/2/3 依次进入 kafka，kafka 会给这三条数据每条分配一个 offset，代表这条数据的序号，我们就假设分配的 offset 依次是 152/153/154。消费者从 kafka 去消费的时候，也是按照这个顺序去消费。假如当消费者消费了 offset=153 的这条数据，刚准备去提交 offset 到 zookeeper，此时消费者进程被重启了。那么此时消费过的数据 1/2 的 offset 并没有提交，kafka 也就不知道你已经消费了 offset=153 这条数据。那么重启之后，消费者会找 kafka 说，嘿，哥儿们，你给我接着把上次我消费到的那个地方后面的数据继续给我传递过来。由于之前的 offset 没有提交成功，那么数据 1/2 会再次传过来，如果此时消费者没有去重的话，那么就会导致重复消费。       
**如果消费者干的事儿是拿一条数据就往数据库里写一条，会导致说，你可能就把数据 1/2 在数据库里插入了 2 次，那么数据就错啦**。

其实重复消费不可怕，可怕的是你没考虑到重复消费之后，怎么保证幂等性。

举个例子吧。**假设你有个系统，消费一条消息就往数据库里插入一条数据，要是你一个消息重复两次，你不就插入了两条，这数据不就错了**？但是你要是消费到第二次的时候，自己判断一下是否已经消费过了，若是就直接扔了，这样不就保留了一条数据，从而保证了数据的正确性。

**一条数据重复出现两次，数据库里就只有一条数据，这就保证了系统的幂等性**。

**幂等性，通俗点说，就一个数据，或者一个请求，给你重复来多次，你得确保对应的数据是不会改变的**，不能出错。

所以第二个问题来了，**怎么保证消息队列消费的幂等性**？

其实还是得结合业务来思考，我这里给几个思路：

1.比如你拿个数据要写库，你**先根据主键查一下，如果这数据都有了，你就别插入了，update 一下好吧**。

2.比如你是**写 Redis，那没问题了，反正每次都是 set，天然幂等性**。 

3.比如你不是上面两个场景，那做的稍微复杂一点，**你需要让生产者发送每条数据的时候，里面加一个全局唯一的 id**，类似订单 id 之类的东西，然后**你这里消费到了之后，先根据这个 id 去比如 Redis 里查一下，之前消费过吗？如果没有消费过，你就处理，然后这个 id 写 Redis。如果消费过了，那你就别处理了**，保证别重复处理相同的消息即可。     

4.比如**基于数据库的唯一键来保证重复数据不会重复插入多条。因为有唯一键约束了，重复数据插入只会报错**，不会导致数据库中出现脏数据。       
        
        

**如何保证消息的可靠性传输？或者说，如何处理消息丢失的问题？**      
一、RabbitMQ        

**生产者弄丢了数据**        

生产者将数据发送到 RabbitMQ 的时候，可能数据就在半路给搞丢了，因为网络问题啥的，都有可能。

此时可以**选择用 RabbitMQ提供的事务功能，就是生产者发送数据之前开启 RabbitMQ 事务 channel.txSelect ，然后发送消息，如果消息没有成功被 RabbitMQ 接收到，那么生产者会收到异常报错，此时就可以回滚事务 channel.txRollback，然后重试发送消息；如果收到了消息，那么可以提交事务 channel.txCommit。**     

但是问题是，**RabbitMQ 事务机制（同步）一搞，基本上吞吐量会下来，因为太耗性能**。

所以一般来说，**如果你要确保说写 RabbitMQ 的消息别丢，可以开启 confirm 模式**，在生产者那里设置开启 confirm 模式之后，**你每次写的消息都会分配一个唯一的 id，然后如果写入了 RabbitMQ 中，RabbitMQ 会给你回传一个 ack 消息，告诉你说这个消息 ok 了。如果 RabbitMQ 没能处理这个消息，会回调你的一个 nack 接口，告诉你这个消息接收失败，你可以重试**。而且你可以结合这个机制自己在内存里维护每个消息 id 的状态，如果超过一定时间还没接收到这个消息的回调，那么你可以重发。

事务机制和 confirm 机制最大的不同在于，**事务机制是同步的，你提交一个事务之后会阻塞在那儿，但是 confirm 机制是异步的，你发送个消息之后就可以发送下一个消息，然后那个消息 RabbitMQ 接收了之后会异步回调你的一个接口通知你这个消息接收到了**。

所以一般在生产者这块避免数据丢失，**都是用 confirm 机制的**。       

**RabbitMQ 弄丢了数据**     

就是 RabbitMQ 自己弄丢了数据，**这个必须开启 RabbitMQ 的持久化，就是消息写入之后会持久化到磁盘，哪怕是 RabbitMQ 自己挂了，恢复之后会自动读取之前存储的数据**，一般数据不会丢。除非极其罕见的是，RabbitMQ 还没持久化，自己就挂了，可能导致少量数据丢失，但是这个概率较小。

**设置持久化有两个步骤**：

**1.创建 queue 的时候将其设置为持久化**     
这样就可以保证 RabbitMQ 持久化 queue 的元数据，但是它是不会持久化 queue 里的数据的。

**2.第二个是发送消息的时候将消息的 deliveryMode 设置为 2**      
就是将消息设置为持久化的，此时 RabbitMQ 就会将消息持久化到磁盘上去。

**必须要同时设置这两个持久化才行，RabbitMQ 哪怕是挂了，再次重启，也会从磁盘上重启恢复 queue，恢复这个 queue 里的数据**。

注意，哪怕是你给 RabbitMQ 开启了持久化机制，也**有一种可能，就是这个消息写到了 RabbitMQ 中，但是还没来得及持久化到磁盘上，结果不巧，此时 RabbitMQ 挂了，就会导致内存里的一点点数据丢失**。

所以，**持久化可以跟生产者那边的 confirm 机制配合起来，只有消息被持久化到磁盘之后，才会通知生产者 ack 了**，所以哪怕是在持久化到磁盘之前，RabbitMQ 挂了，数据丢了，生产者收不到 ack ，你也是可以自己重发的。       

**消费端弄丢了数据**        

RabbitMQ 如果丢失了数据，主要是因为你消费的时候，刚消费到，还没处理，结果进程挂了，比如重启了，那么就尴尬了，RabbitMQ 认为你都消费了，这数据就丢了。

这个时候得用 RabbitMQ 提供的 ack 机制，简单来说，就是**你必须关闭 RabbitMQ 的自动 ack ，可以通过一个 api 来调用就行，然后每次你自己代码里确保处理完的时候，再在程序里 ack 一把。这样的话，如果你还没处理完，不就没有 ack 了**？那 RabbitMQ 就认为你还没处理完，**这个时候 RabbitMQ 会把这个消费分配给别的 consumer 去处理，消息是不会丢的**。     




**Kafka**       

**消费端弄丢了数据**        

**唯一可能导致消费者弄丢数据的情况**，就是说，**你消费到了这个消息，然后消费者那边自动提交了 offset**，让 Kafka 以为你已经消费好了这个消息，但其实你才刚准备处理这个消息，你还没处理，你自己就挂了，此时这条消息就丢咯。

这不是**跟 RabbitMQ 差不多吗，大家都知道 Kafka 会自动提交 offset，那么只要关闭自动提交 offset，在处理完之后自己手动提交 offset，就可以保证数据不会丢**。但是此时**确实还是可能会有重复消费，比如你刚处理完，还没提交 offset，结果自己挂了，此时肯定会重复消费一次，自己保证幂等性就好**了。


**Kafka 弄丢了数据**        

这块**比较常见的一个场景，就是 Kafka 某个 broker 宕机，然后重新选举 partition 的 leader**。大家想想，**要是此时其他的 follower 刚好还有些数据没有同步，结果此时 leader 挂了，然后选举某个 follower 成 leader 之后，不就少了一些数据**？这就丢了一些数据啊。

所以此时一般是要求起码设置如下 4 个参数：

1.给 topic 设置 replication.factor 参数：这个值必须大于 1，**要求每个 partition 必须有至少 2 个副本**。     

2.在 Kafka 服务端设置 min.insync.replicas 参数：这个值必须大于 1，这个是**要求一个 leader 至少感知到有至少一个 follower 还跟自己保持联系，没掉队**，这样才能确保 leader 挂了还有一个 follower 吧。        

3.在 producer 端设置 acks=all ：这个是要求**每条数据，必须是写入所有 replica 之后，才能认为是写成功了**。      

4.在 producer 端设置 retries=MAX （很大很大很大的一个值，无限次重试的意思）：这个是**要求一旦写入失败，就无限重试**，卡在这里了。      

我们生产环境就是按照上述要求配置的，这样配置之后，至少在 Kafka broker 端就可以保证在 leader 所在 broker 发生故障，进行 leader 切换时，数据不会丢失。

**生产者会不会弄丢数据？**      

如果**按照上述的思路设置了 acks=all ，一定不会丢，要求是，你的 leader 接收到消息，所有的 follower 都同步到了消息之后，才认为本次写成功了**。如果**没满足这个条件，生产者会自动不断的重试，重试无限次**。        


**如何保证消息的顺序性？**      

以前做过一个 mysql binlog 同步的系统，压力还是非常大的，日同步数据要达到上亿，就是说数据从一个 mysql 库原封不动地同步到另一个 mysql 库里面去（mysql -> mysql）。常见的一点在于说比如大数据 team，就需要同步一个 mysql 库过来，对公司的业务系统的数据做各种复杂的操作。

你在 mysql 里增删改一条数据，对应出来了增删改 3 条 binlog 日志，接着这三条 binlog 发送到 MQ 里面，再消费出来依次执行，起码得保证人家是按照顺序来的吧？不然本来是：增加、修改、删除；你愣是换了顺序给执行成删除、修改、增加，不全错了么。

本来这个数据同步过来，应该最后这个数据被删除了；结果你搞错了这个顺序，最后这个数据保留下来了，数据同步就出错了。      

先看看**顺序会错乱的俩场景**：

**RabbitMQ**：一个 queue，多个 consumer。比如，生产者向 RabbitMQ 里发送了三条数据，顺序依次是 data1/data2/data3，压入的是 RabbitMQ 的一个内存队列。**有三个消费者分别从 MQ 中消费这三条数据中的一条，结果消费者2先执行完操作**，把 data2 存入数据库，然后是 data1/data3。这不明显乱了。      

**Kafka**：比如说我们建了一个 topic，有三个 partition。生产者在写的时候，其实**可以指定一个 key**，比如说我们指定了**某个订单 id 作为 key，那么这个订单相关的数据，一定会被分发到同一个 partition 中去，而且这个 partition 中的数据一定是有顺序的**。
消费者从 partition 中取出来数据的时候，也一定是有顺序的。到这里，顺序还是 ok 的，没有错乱。接着，我们**在消费者里可能会搞多个线程来并发处理消息**。因为如果消费者是单线程消费处理，而处理比较耗时的话，比如处理一条消息耗时几十 ms，那么 1 秒钟只能处理几十条消息，这吞吐量太低了。而**多个线程并发跑的话，顺序可能就乱掉了**。      

**解决方案**：  

**RabbitMQ**        

**拆分多个 queue，每个 queue 一个 consumer，就是多一些 queue 而已，确实是麻烦点**；或者就一个 queue 但是对应一个 consumer，然后这个 consumer 内部用内存队列做排队，然后分发给底层不同的 worker 来处理。

**Kafka**       

1.一个 topic，一个 partition，一个 consumer，内部单线程消费，单线程吞吐量太低，一般不会用这个。        
2.**写 N 个内存 queue，具有相同 key 的数据都到同一个内存 queue；然后对于 N 个线程，每个线程分别消费一个内存 queue 即可，这样就能保证顺序性**。

        
        

**如何解决消息队列的延时以及过期失效问题？消息队列满了以后该怎么处理？有几百万消息持续积压几小时，说说怎么解决？**        

**大量消息在 mq 里积压了几个小时了还没解决？**        

几千万条数据在 MQ 里积压了七八个小时，从下午 4 点多，积压到了晚上 11 点多。这个是我们真实遇到过的一个场景，确实是线上故障了，这个时候要不然就是修复 consumer 的问题，让它恢复消费速度，然后傻傻的等待几个小时消费完毕。这个肯定不能在面试的时候说吧。

一个消费者一秒是 1000 条，一秒 3 个消费者是 3000 条，一分钟就是 18 万条。所以如果你积压了几百万到上千万的数据，即使消费者恢复了，也需要大概 1 小时的时间才能恢复过来。

一般这个时候，**只能临时紧急扩容了**，具体操作步骤和思路如下：

1.先修复 consumer 的问题，确保其恢复消费速度，然后将现有 consumer 都停掉。        
2.新建一个 topic，partition 是原来的 10 倍，临时建立好原先 10 倍的 queue 数量。        
3.然后写一个临时的分发数据的 consumer 程序，这个程序部署上去消费积压的数据，消费之后不做耗时的处理，直接均匀轮询写入临时建立好的 10 倍数量的 queue。        
4.接着临时征用 10 倍的机器来部署 consumer，每一批 consumer 消费一个临时 queue 的数据。这种做法相当于是临时将 queue 资源和 consumer 资源扩大 10 倍，以正常的 10 倍速度来消费数据。     
5.等快速消费完积压数据之后，得恢复原先部署的架构，重新用原先的 consumer 机器来消费消息。       

**MQ 中的消息过期失效了**       

假设你用的是 RabbitMQ，**RabbtiMQ 是可以设置过期时间的，也就是 TTL**。如果消息在 queue 中积压超过一定的时间就会被 RabbitMQ 给清理掉，这个数据就没了。那这就是第二个坑了。这就**不是说数据会大量积压在 mq 里，而是大量的数据会直接搞丢**。

这个情况下，就不是说要增加 consumer 消费积压的消息，因为**实际上没啥积压，而是丢了大量的消息**。我们可以**采取一个方案，就是批量重导**，这个我们之前线上也有类似的场景干过。**就是大量积压的时候，我们当时就直接丢弃数据了，然后等过了高峰期以后**，比如大家一起喝咖啡熬夜到晚上12点以后，用户都睡觉了。这个时候我们就开始写程序，**将丢失的那批数据，写个临时程序，一点一点的查出来，然后重新灌入 MQ 里面去，把白天丢的数据给他补回来**。也只能是这样了。

假设 1 万个订单积压在 mq 里面，没有处理，其中 1000 个订单都丢了，你只能手动写程序把那 1000 个订单给查出来，手动发到 mq 里去再补一次。

**MQ都快写满了**        

如果消息积压在 MQ 里，你很长时间都没有处理掉，此时导致 MQ 都快写满了，咋办？这个还有别的办法吗？没有，谁让你第一个方案执行的太慢了，你**临时写程序，接入数据来消费，消费一个丢弃一个，都不要了，快速消费掉所有的消息。然后走第二个方案，到了晚上再补数据吧**。



**如果让你写一个消息队列，该如何进行架构设计？**

比如说这个消息队列系统，我们从以下几个角度来考虑一下：

1.首先这个 mq 得支持可伸缩性吧，就是需要的时候快速扩容，就可以增加吞吐量和容量，那怎么搞？设计个分布式的系统呗，参照一下 kafka 的设计理念，broker -> topic -> partition，每个 partition 放一个机器，就存一部分数据。如果现在资源不够了，简单啊，给 topic 增加 partition，然后做数据迁移，增加机器，不就可以存放更多数据，提供更高的吞吐量了？

2.其次你得考虑一下这个 mq 的数据要不要落地磁盘吧？那肯定要了，落磁盘才能保证别进程挂了数据就丢了。那落磁盘的时候怎么落啊？顺序写，这样就没有磁盘随机读写的寻址开销，磁盘顺序读写的性能是很高的，这就是 kafka 的思路。

3.其次你考虑一下你的 mq 的可用性啊？这个事儿，具体参考之前可用性那个环节讲解的 kafka 的高可用保障机制。多副本 -> leader & follower -> broker 挂了重新选举 leader 即可对外服务。

4.能不能支持数据 0 丢失啊？可以的，参考我们之前说的那个 kafka 数据零丢失方案。

### 搜索引擎        

**ES 的分布式架构原理能说一下么（ES 是如何实现分布式的啊）？**             

**ElasticSearch 设计的理念就是分布式搜索引擎**，底层其实还是基于 lucene 的。**核心思想就是在多台机器上启动多个 ES 进程实例**，组成了一个 ES 集群。

ES 中**存储数据的基本单位是索引**，比如说你现在要在 ES 中存储一些订单数据，你就应该在 ES 中创建一个索引 order_idx ，所有的订单数据就都写到这个索引里面去，一个索引差不多就是相当于是 mysql 里的一张表。

**index -> type -> mapping -> document -> field**

这样吧，为了做个更直白的介绍，我在这里做个类比。但是切记，不要划等号，类比只是为了便于理解。

index 相当于 mysql 里的一张表。而 type 没法跟 mysql 里去对比，一个 **index 里可以有多个 type，每个 type 的字段都是差不多的**，但是有一些略微的差别。**假设有一个 index，是订单 index，里面专门是放订单数据的。就好比说你在 mysql 中建表，有些订单是实物商品的订单**，比如一件衣服、一双鞋子；**有些订单是虚拟商品的订单**，比如游戏点卡，话费充值。**就两种订单大部分字段是一样的，但是少部分字段可能有略微的一些差别**。

所以**就会在订单 index 里，建两个 type，一个是实物商品订单 type，一个是虚拟商品订单 type，这两个 type 大部分字段是一样的**，少部分字段是不一样的。

很多情况下，**一个 index 里可能就一个 type，但是确实如果说是一个 index 里有多个 type 的情况**（注意， mapping types 这个概念在 ElasticSearch 7. X 已被完全移除，详细说明可以参考官方文档），你可以**认为 index 是一个类别的表，具体的每个 type 代表了 mysql 中的一个表。每个 type 有一个 mapping，如果你认为一个 type 是具体的一个表**，index 就代表多个 type 同属于的一个类型，而** mapping 就是这个 type 的表结构定义**，你在 mysql 中创建一个表，肯定是要定义表结构的，里面有哪些字段，每个字段是什么类型。**实际上你往 index 里的一个 type 里面写的一条数据，叫做一条 document，一条 document 就代表了 mysql 中某个表里的一行，每个 document 有多个 field，每个 field 就代表了这个 document 中的一个字段的值**。        

你**搞一个索引，这个索引可以拆分成多个 shard ，每个 shard 存储部分数据**。**拆分多个 shard 是有好处的，一是支持横向扩展**，比如你数据量是 3T，3 个 shard，每个 shard 就 1T 的数据，若现在数据量增加到 4T，怎么扩展，很简单，重新建一个有 4 个 shard 的索引，将数据导进去；**二是提高性能**，数据分布在多个 shard，即多台服务器上，**所有的操作，都会在多台机器上并行分布式执行，提高了吞吐量和性能**。

接着就是这个 shard 的数据实际是有多个备份，就是说**每个 shard 都有一个 primary shard ，负责写入数据，但是还有几个 replica shard 。 primary shard 写入数据之后，会将数据同步到其他几个 replica shard 上去**。        

通过这个 replica 的方案，每个 shard 的数据都有多个备份，如果某个机器宕机了，没关系啊，还有别的数据副本在别的机器上呢。高可用了吧。

**ES 集群多个节点，会自动选举一个节点为 master 节点，这个 master 节点其实就是干一些管理的工作的，比如维护索引元数据、负责切换 primary shard 和 replica shard 身份等。要是 master 节点宕机了，那么会重新选举一个节点为 master 节点**。

如果是非 master节点宕机了，那么会由 master 节点，让那个宕机节点上的 primary shard 的身份转移到其他机器上的 replica shard。接着你要是修复了那个宕机机器，重启了之后，master 节点会控制将缺失的 replica shard 分配过去，同步后续修改的数据之类的，让集群恢复正常。

说得更简单一点，就是说**如果某个非 master 节点宕机了。那么此节点上的 primary shard 不就没了。那好，master 会让 primary shard 对应的 replica shard（在其他机器上）切换为 primary shard。如果宕机的机器修复了，修复后的节点也不再是 primary shard，而是 replica shard**。

其实上述就是 ElasticSearch 作为分布式搜索引擎最基本的一个架构设计。     

**ES 写入数据的工作原理是什么啊？ES 查询数据的工作原理是什么啊？底层的 Lucene 介绍一下呗？倒排索引了解吗？**      

**es 写数据过程**       

1.**客户端选择一个 node 发送请求过去，这个 node 就是 coordinating node**（协调节点）。     
2.**coordinating node 对 document 进行路由，将请求转发给对应的 node**（有 primary shard）。      
3.**实际的 node 上的 primary shard 处理请求，然后将数据同步到 replica node** 。     
4.**coordinating node 如果发现 primary node 和所有 replica node 都搞定之后，就返回响应结果给客户端**。      

**es 读数据过程**       

可以**通过 doc id 来查询，会根据 doc id 进行 hash，判断出来当时把 doc id 分配到了哪个 shard 上面去，从那个 shard 去查询**。

1.**客户端发送请求到任意一个 node，成为 coordinate node** 。        
2.**coordinate node 对 doc id 进行哈希路由，将请求转发到对应的 node，此时会使用 round-robin 随机轮询算法，在 primary shard 以及其所有 replica 中随机选择一个，让读请求负载均衡**。       
3.**接收请求的 node 返回 document 给 coordinate node** 。       
4.coordinate node **返回 document 给客户端**。      

**es 搜索数据过程**     

es 最强大的是做全文检索，就是比如你有三条数据：

java真好玩儿啊      
java好难学啊        
j2ee特别牛      

**你根据 java 关键词来搜索，将包含 java 的 document 给搜索出来**。es 就会给你返回：java真好玩儿啊，java好难学啊。

1.**客户端发送请求到一个 coordinate node** 。       
2.**协调节点将搜索请求转发到所有的 shard 对应的 primary shard 或 replica shard** ，都可以。        
3.query phase：**每个 shard 将自己的搜索结果（其实就是一些 doc id ）返回给协调节点，由协调节点进行数据的合并、排序、分页等操作**，产出最终结果。      
4.fetch phase：**接着由协调节点根据 doc id 去各个节点上拉取实际的 document 数据，最终返回给客户端**。       

写请求是写入 primary shard，然后同步给所有的 replica shard；读请求可以从 primary shard 或 replica shard 读取，采用的是随机轮询算法。        

**写数据底层原理**      

先写入内存 buffer，在 buffer 里的时候数据是搜索不到的；同时将数据写入 translog 日志文件。

如果 buffer 快满了，或者到一定时间，就会将内存 buffer 数据 refresh 到一个新的 segment file 中，但是此时数据不是直接进入 segment file 磁盘文件，而是先进入 os cache 。这个过程就是 refresh 。

每隔 1 秒钟，es 将 buffer 中的数据写入一个新的 segment file ，每秒钟会产生一个新的磁盘文件 segment file ，这个 segment file 中就存储最近 1 秒内 buffer 中写入的数据。

但是如果 buffer 里面此时没有数据，那当然不会执行 refresh 操作，如果 buffer 里面有数据，默认 1 秒钟执行一次 refresh 操作，刷入一个新的 segment file 中。

操作系统里面，磁盘文件其实都有一个东西，叫做 os cache ，即操作系统缓存，就是说数据写入磁盘文件之前，会先进入 os cache ，先进入操作系统级别的一个内存缓存中去。只要 buffer 中的数据被 refresh 操作刷入 os cache 中，这个数据就可以被搜索到了。

为什么叫 es 是准实时的？ NRT ，全称 near real-time 。默认是每隔 1 秒 refresh 一次的，所以 es 是准实时的，因为写入的数据 1 秒之后才能被看到。可以通过 es 的 restful api 或者 java api ，手动执行一次 refresh 操作，就是手动将 buffer 中的数据刷入 os cache 中，让数据立马就可以被搜索到。只要数据被输入 os cache 中，buffer 就会被清空了，因为不需要保留 buffer 了，数据在 translog 里面已经持久化到磁盘去一份了。

重复上面的步骤，新的数据不断进入 buffer 和 translog，不断将 buffer 数据写入一个又一个新的 segment file 中去，每次 refresh 完 buffer 清空，translog 保留。随着这个过程推进，translog 会变得越来越大。当 translog 达到一定长度的时候，就会触发 commit 操作。

commit 操作发生第一步，就是将 buffer 中现有数据 refresh 到 os cache 中去，清空 buffer。然后，将一个 commit point 写入磁盘文件，里面标识着这个 commit point 对应的所有 segment file ，同时强行将 os cache 中目前所有的数据都 fsync 到磁盘文件中去。最后清空 现有 translog 日志文件，重启一个 translog，此时 commit 操作完成。

这个 commit 操作叫做 flush 。默认 30 分钟自动执行一次 flush ，但如果 translog 过大，也会触发 flush 。flush 操作就对应着 commit 的全过程，我们可以通过 es api，手动执行 flush 操作，手动将 os cache 中的数据 fsync 强刷到磁盘上去。

translog 日志文件的作用是什么？你执行 commit 操作之前，数据要么是停留在 buffer 中，要么是停留在 os cache 中，无论是 buffer 还是 os cache 都是内存，一旦这台机器死了，内存中的数据就全丢了。所以需要将数据对应的操作写入一个专门的日志文件 translog 中，一旦此时机器宕机，再次重启的时候，es 会自动读取 translog 日志文件中的数据，恢复到内存 buffer 和 os cache 中去。

translog 其实也是先写入 os cache 的，默认每隔 5 秒刷一次到磁盘中去，所以默认情况下，可能有 5 秒的数据会仅仅停留在 buffer 或者 translog 文件的 os cache 中，如果此时机器挂了，会丢失 5 秒钟的数据。但是这样性能比较好，最多丢 5 秒的数据。也可以将 translog 设置成每次写操作必须是直接 fsync 到磁盘，但是性能会差很多。

实际上你在这里，**如果面试官没有问你 es 丢数据的问题，你可以在这里给面试官炫一把，你说，其实 es 第一是准实时的，数据写入 1 秒后可以搜索到；可能会丢失数据的。有 5 秒的数据，停留在 buffer、translog os cache、segment file os cache 中，而不在磁盘上，此时如果宕机，会导致 5 秒的数据丢失**。

总结一下，**数据先写入内存 buffer，然后每隔 1s，将数据 refresh 到 os cache，到了 os cache 数据就能被搜索到（所以我们才说 es 从写入到能被搜索到，中间有 1s 的延迟）。每隔 5s，将数据写入 translog 文件（这样如果机器宕机，内存数据全没，最多会有 5s 的数据丢失），translog 大到一定程度，或者默认每隔 30mins，会触发 commit 操作，将缓冲区的数据都 flush 到 segment file 磁盘文件中**。

**数据写入 segment file 之后，同时就建立好了倒排索引**。        

**删除/更新数据底层原理**       

如果是**删除操作，commit 的时候会生成一个 .del 文件，里面将某个 doc 标识为 deleted 状态，那么搜索的时候根据 .del 文件就知道这个 doc 是否被删除了**。

如果是**更新操作，就是将原来的 doc 标识为 deleted 状态，然后新写入一条数据**。

**buffer 每 refresh 一次，就会产生一个 segment file ，所以默认情况下是 1 秒钟一个 segment file** ，这样下来 segment file 会越来越多，此时**会定期执行 merge。每次 merge 的时候，会将多个 segment file 合并成一个，同时这里会将标识为 deleted 的 doc 给物理删除掉，然后将新的 segment file 写入磁盘，这里会写一个 commit point ，标识所有新的 segment file ，然后打开 segment file 供搜索使用，同时删除旧的 segment file** 。

**底层 lucene**     

简单来说，**lucene 就是一个 jar 包，里面包含了封装好的各种建立倒排索引的算法代码**。我们用 Java 开发的时候，引入 lucene jar，然后基于 lucene 的 api 去开发就可以了。

通过 lucene，我们可以将已有的数据建立索引，lucene 会在本地磁盘上面，给我们组织索引的数据结构。        

**倒排索引**        

**在搜索引擎中，每个文档都有一个对应的文档 ID，文档内容被表示为一系列关键词的集合**。例如，文档 1 经过分词，提取了 20 个关键词，每个关键词都会记录它在文档中出现的次数和出现位置。

那么，**倒排索引就是关键词到文档 ID 的映射，每个关键词都对应着一系列的文件**，这些文件中都出现了关键词。      

另外，实用的倒排索引还可以记录更多的信息，比如文档频率信息，表示在文档集合中有多少个文档包含某个单词。

那么，**有了倒排索引，搜索引擎可以很方便地响应用户的查询**。比如**用户输入查询 Facebook ，搜索系统查找倒排索引，从中读出包含这个单词的文档，这些文档就是提供给用户的搜索结果**。

要注意倒排索引的两个重要细节：

1.倒排索引中的所有词项对应一个或多个文档；        
2.倒排索引中的词项根据字典顺序升序排列                
上面只是一个简单的栗子，并没有严格按照字典顺序升序排列。        


**ES 在数据量很大的情况下（数十亿级别）如何提高查询效率啊？**       
**性能优化的杀手锏——filesystem cache**      

你**往 es 里写的数据，实际上都写到磁盘文件里去了，查询的时候，操作系统会将磁盘文件里的数据自动缓存到 filesystem cache 里面去**。      

**es 的搜索引擎严重依赖于底层的 filesystem cache ，你如果给 filesystem cache 更多的内存，尽量让内存可以容纳所有的 idx segment file 索引数据文件，那么你搜索的时候就基本都是走内存的，性能会非常高**。

性能差距究竟可以有多大？我们之前很多的测试和压测，如果走磁盘一般肯定上秒，搜索性能绝对是秒级别的，1秒、5秒、10秒。但如果是走 filesystem cache ，是走纯内存的，那么一般来说性能比走磁盘要高一个数量级，基本上就是毫秒级的，从几毫秒到几百毫秒不等。

这里有个真实的案例。某个公司 es 节点有 3 台机器，每台机器看起来内存很多，64G，总内存就是 64 * 3 = 192G 。每台机器给 es jvm heap 是 32G ，那么剩下来留给 filesystem cache 的就是每台机器才 32G ，总共集群里给 filesystem cache 的就是 32 * 3 = 96G 内存。而此时，整个磁盘上索引数据文件，在 3 台机器上一共占用了 1T 的磁盘容量，es 数据量是 1T ，那么每台机器的数据量是 300G 。这样性能好吗？ filesystem cache 的内存才 100G，十分之一的数据可以放内存，其他的都在磁盘，然后你执行搜索操作，大部分操作都是走磁盘，性能肯定差。

归根结底，你**要让 es 性能要好，最佳的情况下，就是你的机器的内存，至少可以容纳你的总数据量的一半**。

根据我们自己的生产环境实践经验，**最佳的情况下，是仅仅在 es 中就存少量的数据，就是你要用来搜索的那些索引**，如果内存留给 filesystem cache 的是 100G，那么你就将索引数据控制在 100G 以内，**这样的话，你的数据几乎全部走内存来搜索，性能非常之高**，一般可以在 1 秒以内。

比如说你现在有一行数据。 **id,name,age .... 30 个字段。但是你现在搜索，只需要根据 id,name,age 三个字段来搜索**。如果你傻乎乎往 es 里写入一行数据所有的字段，就会导致说 90% 的数据是不用来搜索的，结果硬是占据了 es 机器上的 filesystem cache 的空间，**单条数据的数据量越大，就会导致 filesystem cahce 能缓存的数据就越少。其实，仅仅写入 es 中要用来检索的少数几个字段就可以了，比如说就写入 es id,name,age 三个字段，然后你可以把其他的字段数据存在 mysql/hbase 里，我们一般是建议用 es + hbase 这么一个架构**。

**hbase 的特点是适用于海量数据的在线存储**，就是对 hbase 可以写入海量数据，但是不要做复杂的搜索，做很简单的一些根据 id 或者范围进行查询的这么一个操作就可以了。**从 es 中根据 name 和 age 去搜索，拿到的结果可能就 20 个 doc id ，然后根据 doc id 到 hbase 里去查询每个 doc id 对应的完整的数据，给查出来，再返回给前端**。

写入 es 的数据最好小于等于，或者是略微大于 es 的 filesystem cache 的内存容量。然后你从 es 检索可能就花费 20ms，然后再根据 es 返回的 id 去 hbase 里查询，查 20 条数据，可能也就耗费个 30ms，可能你原来那么玩儿，1T 数据都放 es，会每次查询都是 5~10s，现在可能性能就会很高，每次查询就是 50ms。        

**数据预热**        

假如说，哪怕是你就按照上述的方案去做了，es 集群中每个机器写入的数据量还是超过了 filesystem cache 一倍，比如说你写入一台机器 60G 数据，结果 filesystem cache 就 30G，还是有 30G 数据留在了磁盘上。

其实可以做数据预热。

举个例子，拿微博来说，你可以把一些大V，平时看的人很多的数据，你自己**提前后台搞个系统，每隔一会儿，自己的后台系统去搜索一下热数据，刷到 filesystem cache 里去，后面用户实际上来看这个热数据的时候，他们就是直接从内存里搜索了，很快**。

或者是电商，你可以将平时查看最多的一些商品，比如说 iphone 8，热数据提前后台搞个程序，每隔 1 分钟自己主动访问一次，刷到 filesystem cache 里去。

对于**那些你觉得比较热的、经常会有人访问的数据，最好做一个专门的缓存预热子系统，就是对热数据每隔一段时间，就提前访问一下，让数据进入 filesystem cache 里面去**。这样下次别人访问的时候，性能一定会好很多。      

**冷热分离**        

**es 可以做类似于 mysql 的水平拆分**，就是说将大量的访问很少、频率很低的数据，单独写一个索引，然后将访问很频繁的热数据单独写一个索引。**最好是将冷数据写入一个索引中，然后热数据写入另外一个索引中，这样可以确保热数据在被预热之后，尽量都让他们留在 filesystem os cache 里**，别让冷数据给冲刷掉。

你看，假设你有 6 台机器，2 个索引，一个放冷数据，一个放热数据，每个索引 3 个 shard。3 台机器放热数据 index，另外 3 台机器放冷数据 index。然后这样的话，**你大量的时间是在访问热数据 index，热数据可能就占总数据量的 10%，此时数据量很少，几乎全都保留在 filesystem cache 里面了，就可以确保热数据的访问性能是很高的**。但是对于冷数据而言，是在别的 index 里的，跟热数据 index 不在相同的机器上，大家互相之间都没什么联系了。**如果有人访问冷数据，可能大量数据是在磁盘上的，此时性能差点，就 10% 的人去访问冷数据，90% 的人在访问热数据，也无所谓了**。      

### 缓存        

**项目中缓存是如何使用的？为什么要用缓存？缓存使用不当会造成什么后果？**      

**为什么要用缓存？**        

用缓存，主要有两个用途：高性能、高并发。

**高性能**      

假设这么个场景，你有个操作，一个请求过来，吭哧吭哧你各种乱七八糟操作 mysql，半天查出来一个结果，耗时 600ms。但是这个结果可能接下来几个小时都不会变了，或者变了也可以不用立即反馈给用户。那么此时咋办？

缓存啊，折腾 600ms 查出来的结果，扔缓存里，一个 key 对应一个 value，下次再有人查，别走 mysql 折腾 600ms 了，直接从缓存里，通过一个 key 查出来一个 value，2ms 搞定。性能提升 300 倍。

就是说**对于一些需要复杂操作耗时查出来的结果，且确定后面不怎么变化，但是有很多读请求，那么直接将查询出来的结果放在缓存中，后面直接读缓存就好**。

**高并发**      

mysql 这么重的数据库，压根儿设计不是让你玩儿高并发的，虽然也可以玩儿，但是天然支持不好。mysql 单机支撑到 2000QPS 也开始容易报警了。

所以要是你有个系统，高峰期一秒钟过来的请求有 1万，那一个 mysql 单机绝对会死掉。你这个时候就只能上缓存，把很多数据放缓存，别放 mysql。**缓存功能简单，说白了就是 key-value 式操作，单机支撑的并发量轻松一秒几万十几万，支撑高并发 so easy。单机承载并发量是 mysql 单机的几十倍**。

**缓存是走内存的，内存天然就支撑高并发**。      

**用了缓存之后会有什么不良后果？**      

1.缓存与数据库双写不一致        
2.缓存雪崩、缓存穿透、缓存击穿      
3.缓存并发竞争      

**如何保证缓存与数据库的双写一致性？**      

**Cache Aside Pattern**     

最经典的缓存+数据库读写的模式，就是 Cache Aside Pattern。

**1.读的时候，先读缓存，缓存没有的话，就读数据库，然后取出数据后放入缓存，同时返回响应。        
2.更新的时候，先更新数据库，然后再删除缓存**。      

**为什么是删除缓存，而不是更新缓存？**

原因很简单，**很多时候，在复杂点的缓存场景，缓存不单单是数据库中直接取出来的值**。

比如可能更新了某个表的一个字段，然后其对应的缓存，是需要查询另外两个表的数据并进行运算，才能计算出缓存最新的值的。

另外**更新缓存的代价有时候是很高的**。是不是说，每次修改数据库的时候，都一定要将其对应的缓存更新一份？也许有的场景是这样，但是**对于比较复杂的缓存数据计算的场景**，就不是这样了。如果你频繁修改一个缓存涉及的多个表，缓存也频繁更新。但是问题在于，**这个缓存到底会不会被频繁访问到**？

举个栗子，一个缓存涉及的表的字段，在 1 分钟内就修改了 20 次，或者是 100 次，那么缓存更新 20 次、100 次；但是这个缓存在 1 分钟内只被读取了 1 次，有大量的冷数据。实际上，如果你只是删除缓存的话，那么在 1 分钟内，这个缓存不过就重新计算一次而已，开销大幅度降低。用到缓存才去算缓存。

**其实删除缓存，而不是更新缓存，就是一个 lazy 计算的思想，不要每次都重新做复杂的计算，不管它会不会用到，而是让它到需要被使用的时候再重新计算**。像 mybatis，hibernate，都有懒加载思想。查询一个部门，部门带了一个员工的 list，没有必要说每次查询部门，都把里面的 1000 个员工的数据也同时查出来啊。80% 的情况，查这个部门，就只是要访问这个部门的信息就可以了。先查部门，同时要访问里面的员工，那么这个时候只有在你要访问里面的员工的时候，才会去数据库里面查询 1000 个员工。      

**最初级的缓存不一致问题及解决方案**        

问题：先更新数据库，再删除缓存。如果删除缓存失败了，那么会导致数据库中是新数据，缓存中是旧数据，数据就出现了不一致。      

解决思路：先删除缓存，再更新数据库。如果数据库更新失败了，那么数据库中是旧数据，缓存中是空的，那么数据不会不一致。因为读的时候缓存没有，所以去读了数据库中的旧数据，然后更新到缓存中。      

**比较复杂的数据不一致问题分析**        

数据发生了变更，先删除了缓存，然后要去修改数据库，此时还没修改。一个请求过来，去读缓存，发现缓存空了，去查询数据库，查到了修改前的旧数据，放到了缓存中。随后数据变更的程序完成了数据库的修改。完了，数据库和缓存中的数据不一样了...

为什么上亿流量高并发场景下，缓存会出现这个问题？

**只有在对一个数据在并发的进行读写的时候，才可能会出现这种问题**。其实如果说你的并发量很低的话，特别是读并发很低，每天访问量就 1 万次，那么很少的情况下，会出现刚才描述的那种不一致的场景。但是问题是，如果每天的是上亿的流量，每秒并发读是几万，每秒只要有数据更新的请求，就可能会出现上述的数据库+缓存不一致的情况。

解决方案如下：

**更新数据的时候，根据数据的唯一标识，将操作路由之后，发送到一个 jvm 内部队列中。读取数据的时候，如果发现数据不在缓存中，那么将重新执行“读取数据+更新缓存”的操作，根据唯一标识路由之后，也发送到同一个 jvm 内部队列中**。

**一个队列对应一个工作线程，每个工作线程串行拿到对应的操作，然后一条一条的执行**。这样的话，一个数据变更的操作，先删除缓存，然后再去更新数据库，但是还没完成更新。此时如果一个读请求过来，没有读到缓存，那么可以先将缓存更新的请求发送到队列中，此时会在队列中积压，然后同步等待缓存更新完成。

这里**有一个优化点，一个队列中，其实多个更新缓存请求串在一起是没意义的，因此可以做过滤，如果发现队列中已经有一个更新缓存的请求了，那么就不用再放个更新请求操作进去了，直接等待前面的更新操作请求完成即可**。

待那个队列对应的工作线程完成了上一个操作的数据库的修改之后，才会去执行下一个操作，也就是缓存更新的操作，此时会从数据库中读取最新的值，然后写入缓存中。

**如果请求还在等待时间范围内，不断轮询发现可以取到值了，那么就直接返回；如果请求等待的时间超过一定时长，那么这一次直接从数据库中读取当前的旧值**。      
        

**了解什么是 Redis 的雪崩、穿透和击穿？Redis 崩溃之后会怎么样？系统该如何应对这种情况？如何处理 Redis 的穿透？**

**缓存雪崩**        

对于系统 A，假设每天高峰期每秒 5000 个请求，本来缓存在高峰期可以扛住每秒 4000 个请求，**但是缓存机器意外发生了全盘宕机。缓存挂了**，此时 1 秒 5000 个**请求全部落数据库，数据库必然扛不住，它会报一下警，然后就挂了**。此时，如果没有采用什么特别的方案来处理这个故障，DBA 很着急，重启数据库，但是数据库立马又被新的流量给打死了。

缓存雪崩的事前事中事后的解决方案如下：

* **事前：Redis 高可用，主从+哨兵，Redis cluster**，避免全盘崩溃。
* **事中：本地 ehcache 缓存 + hystrix 限流&降级**，避免 MySQL 被打死。
* **事后：Redis 持久化**，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。      


用户发送一个请求，**系统 A 收到请求后，先查本地 ehcache 缓存，如果没查到再查 Redis。如果 ehcache 和 Redis 都没有，再查数据库，将数据库中的结果，写入 ehcache 和 Redis 中**。

**限流组件，可以设置每秒的请求，有多少能通过组件，剩余的未通过的请求，怎么办？走降级！可以返回一些默认的值，或者友情提示，或者空值**。

好处：

* 数据库绝对不会死，限流组件确保了每秒只有多少个请求能通过。
* 只要数据库不死，就是说，对用户来说，2/5 的请求都是可以被处理的。
* 只要有 2/5 的请求可以被处理，就意味着你的系统没死，对用户来说，可能就是点击几次刷不出来页面，但是多点几次，就可以刷出来了。


**缓存穿透**        

对于系统A，假设一秒 5000 个请求，结果其中 4000 个请求是黑客发出的恶意攻击。

黑客发出的那 4000 个攻击，缓存中查不到，每次你去数据库里查，也查不到。

举个栗子。数据库 id 是从 1 开始的，结果黑客发过来的请求 id 全部都是负数。这样的话，缓存中不会有，请求每次都“视缓存于无物”，直接查询数据库。这种恶意攻击场景的缓存穿透就会直接把数据库给打死。

解决方案：布隆过滤器        


**缓存击穿**        

缓存击穿，就是**说某个 key 非常热点，访问非常频繁，处于集中式高并发访问的情况，当这个 key 在失效的瞬间，大量的请求就击穿了缓存，直接请求数据库，就像是在一道屏障上凿开了一个洞**。

不同场景下的解决方式可如下：

* 若缓存的数据是**基本不会发生更新的**，则可尝试将该热点数据**设置为永不过期**。
* 若缓存的**数据更新不频繁**，且缓存刷新的整个流程耗时较少的情况下，则可以**采用基于 Redis、zookeeper 等分布式中间件的分布式互斥锁**，或者本地互斥锁以保证仅少量的请求能请求数据库并重新构建缓存，其余线程则在锁释放后能访问到新缓存。
* 若缓存的**数据更新频繁或者在缓存刷新的流程耗时较长的情况**下，可以利用**定时线程在缓存过期前主动地重新构建缓存或者延后缓存的过期时间**，以保证所有的请求能一直访问到对应的缓存。      


**Redis 的并发竞争问题是什么？如何解决这个问题？了解 Redis 事务的 CAS 方案吗**？      

某个时刻，多个系统实例都去更新某个 key。可以基于 zookeeper 实现分布式锁。每个系统通过 zookeeper 获取分布式锁，确保同一时间，只能有一个系统实例在操作某个 key，别人都不允许读和写。   

你要写入缓存的数据，都是从 mysql 里查出来的，都得写入 mysql 中，写入 mysql 中的时候必须保存一个时间戳，从 mysql 查出来的时候，时间戳也查出来。

每次要写之前，先判断一下当前这个 value 的时间戳是否比缓存里的 value 的时间戳要新。如果是的话，那么可以写，否则，就不能用旧的数据覆盖新的数据。        


**Redis 和 Memcached 有什么区别？Redis 的线程模型是什么？为什么 Redis 单线程却能支撑高并发？**      

**Redis 和 Memcached 有啥区别？**
**Redis 支持复杂的数据结构**        

Redis 相比 Memcached 来说，拥有更多的数据结构，能支持更丰富的数据操作。如果需要缓存能够支持更复杂的结构和操作， Redis 会是不错的选择。

**Redis 原生支持集群模式**      

在 Redis3.x 版本中，便能支持 cluster 模式，而 Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据。

**性能对比**        

由于 Redis 只使用单核，而 Memcached 可以使用多核，所以平均每一个核上 Redis 在存储小数据时比 Memcached 性能更高。而在 100k 以上的数据中，Memcached 性能要高于 Redis。虽然 Redis 最近也在存储大数据的性能上进行优化，但是比起 Memcached，还是稍有逊色。       

**Redis 的线程模型**        

**Redis 内部使用文件事件处理器 file event handler ，这个文件事件处理器是单线程的**，所以 Redis 才叫做单线程的模型。**它采用 IO 多路复用机制同时监听多个 socket，将产生事件的 socket 压入内存队列中，事件分派器根据 socket 上的事件类型来选择对应的事件处理器进行处理**。

文件事件处理器的结构包含 4 个部分：

* 多个 socket
* IO 多路复用程序
* 文件事件分派器
* 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）      

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将产生事件的 socket 放入队列中排队，事件分派器每次从队列中取出一个 socket，根据 socket 的事件类型交给对应的事件处理器进行处理。        

**为啥 Redis 单线程模型也能效率这么高？**       

* 纯内存操作。
* 核心是基于非阻塞的 IO 多路复用机制。
* C 语言实现，一般来说，C 语言实现的程序“距离”操作系统更近，执行速度相对会更快。
* 单线程反而避免了多线程的频繁上下文切换问题，预防了多线程可能产生的竞争问题。        

**Redis 6.0 开始引入多线程**        

注意！ Redis 6.0 之后的版本抛弃了单线程模型这一设计，原本使用单线程运行的 Redis 也开始选择性地使用多线程模型。

前面还在强调 Redis 单线程模型的高效性，现在为什么又要引入多线程？这其实说明 Redis 在有些方面，单线程已经不具有优势了。**因为读写网络的 Read/Write 系统调用在 Redis 执行期间占用了大部分 CPU 时间，如果把网络读写做成多线程的方式对性能会有很大提升**。

**Redis 的多线程部分只是用来处理网络数据的读写和协议解析，执行命令仍然是单线程**。 之所以这么设计是不想 Redis 因为多线程而变得复杂，需要去控制 key、lua、事务、LPUSH/LPOP 等等的并发问题。        

**总结**        

Redis **选择使用单线程模型处理客户端的请求主要还是因为 CPU 不是 Redis 服务器的瓶颈**，所以使用多线程模型带来的性能提升并不能抵消它带来的开发成本和维护成本，**系统的性能瓶颈也主要在网络 I/O 操作上**；而 Redis 引入多线程操作也是出于性能上的考虑，对于一些大键值对的删除操作，通过多线程非阻塞地释放内存空间也能减少对 Redis 主线程阻塞的时间，提高执行的效率。      

**Redis 都有哪些数据类型？分别在哪些场景下使用比较合适？**      

**Redis 主要有以下几种数据类型**：

* Strings
* Hashes
* Lists
* Sets
* Sorted Sets

**Redis 的过期策略都有哪些？内存淘汰机制都有哪些？手写一下 LRU 代码实现？**        

**Redis 过期策略**      

Redis 过期策略是：**定期删除+惰性删除**。

所谓**定期删除，指的是 Redis 默认是每隔 100ms 就随机抽取一些设置了过期时间的 key，检查其是否过期，如果过期就删除**。

假设 Redis 里放了 10w 个 key，都设置了过期时间，你每隔几百毫秒，就检查 10w 个 key，那 Redis 基本上就死了，cpu 负载会很高的，消耗在你的检查过期 key 上了。注意，这里可不是每隔 100ms 就遍历所有的设置过期时间的 key，那样就是一场性能上的灾难。**实际上 Redis 是每隔 100ms 随机抽取一些 key 来检查和删除的**。

但是问题是，**定期删除可能会导致很多过期 key 到了时间并没有被删除掉**，那咋整呢？所以就是**惰性删除了。这就是说，在你获取某个 key 的时候，Redis 会检查一下 ，这个 key 如果设置了过期时间那么是否过期了？如果过期了此时就会删除**，不会给你返回任何东西。

获取 key 的时候，如果此时 key 已经过期，就删除，不会返回任何东西。

但是实际上这还是有问题的，如果**定期删除漏掉了很多过期 key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期 key 堆积在内存里，导致 Redis 内存块耗尽了**，咋整？

答案是：**走内存淘汰机制**。        

**内存淘汰机制**        

Redis 内存淘汰机制有以下几个：

noeviction: 当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧，实在是太恶心了。        

**allkeys-lru**：当内存不足以容纳新写入数据时，**在键空间中，移除最近最少使用的 key**（这个是最常用的）。      

allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个 key，这个一般没人用吧，为啥要随机，肯定是把最近最少使用的 key 给干掉啊。      

volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 key（这个一般不太合适）。      

volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 key。      

volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 key 优先移除。       

**如何保证 redis 的高并发和高可用？redis 的主从复制原理能介绍一下么？redis 的哨兵原理能介绍一下么？**        

**redis 实现高并发主要依靠主从架构，一主多从**，一般来说，很多项目其实就足够了，单主用来写入数据，单机几万 QPS，多从用来查询数据，多个从实例可以提供每秒 10w 的 QPS。

如果**想要在实现高并发的同时，容纳大量的数据，那么就需要 redis 集群**，使用 redis 集群之后，可以提供每秒几十万的读写并发。

**redis 高可用，如果是做主从架构部署，那么加上哨兵就可以了，就可以实现，任何一个实例宕机，可以进行主备切换**。        

**Redis 主从架构**      

单机的 Redis，能够承载的 QPS 大概就在上万到几万不等。**对于缓存来说，一般都是用来支撑读高并发的**。因此架构做成主从(master-slave)架构，**一主多从，主负责写，并且将数据复制到其它的 slave 节点，从节点负责读。所有的读请求全部走从节点。这样也可以很轻松实现水平扩容，支撑读高并发**。      

Redis replication -> 主从架构 -> 读写分离 -> 水平扩容支撑读高并发      
**Redis replication 的核心机制**        

* Redis 采用异步方式复制数据到 slave 节点，不过 Redis2.8 开始，slave node 会周期性地确认自己每次复制的数据量；
* 一个 master node 是可以配置多个 slave node 的；
* slave node 也可以连接其他的 slave node；
* slave node 做复制的时候，不会 block master node 的正常工作；
* slave node 在做复制的时候，也不会 block 对自己的查询操作，它会用旧的数据集来提供服务；但是复制完成的时候，需要删除旧数据集，加载新数据集，这个时候就会暂停对外服务了；
* slave node 主要用来进行横向扩容，做读写分离，扩容的 slave node 可以提高读的吞吐量。        


注意，如果**采用了主从架构，那么建议必须开启 master node 的持久化，不建议用 slave node 作为 master node 的数据热备**，因为那样的话，如果**你关掉 master 的持久化，可能在 master 宕机重启的时候数据是空的，然后可能一经过复制， slave node 的数据也丢了**。

另外，**master 的各种备份方案，也需要做**。万一本地的所有文件丢失了**，从备份中挑选一份 rdb 去恢复 master，这样才能确保启动的时候，是有数据的**，即使采用了后续讲解的高可用机制，slave node 可以自动接管 master node，但也可能 sentinel 还没检测到 master failure，master node 就自动重启了，还是可能导致上面所有的 slave node 数据被清空。

**Redis 主从复制的核心原理**        

当**启动一个 slave node 的时候，它会发送一个 PSYNC 命令给 master node**。

如果这是 **slave node 初次连接到 master node，那么会触发一次 full resynchronization 全量复制。此时 master 会启动一个后台线程，开始生成一份 RDB 快照文件，同时还会将从客户端 client 新收到的所有写命令缓存在内存中。 RDB 文件生成完毕后， master 会将这个 RDB 发送给 slave，slave 会先写入本地磁盘，然后再从本地磁盘加载到内存中，接着 master 会将内存中缓存的写命令发送到 slave，slave 也会同步这些数据。slave node 如果跟 master node 有网络故障，断开了连接，会自动重连，连接之后 master node 仅会复制给 slave 部分缺少的数据**。        

**主从复制的断点续传**      

从 **Redis2.8 开始，就支持主从复制的断点续传**，如果主从复制过程中，网络连接断掉了，那么可以接着上次复制的地方，继续复制下去，而不是从头开始复制一份。

**master node 会在内存中维护一个 backlog，master 和 slave 都会保存一个 replica offset 还有一个 master run id，offset 就是保存在 backlog 中的。如果 master 和 slave 网络连接断掉了，slave 会让 master 从上次 replica offset 开始继续复制，如果没有找到对应的 offset，那么就会执行一次 resynchronization** 。

如果根据 host+ip 定位 master node，是不靠谱的，如果 master node 重启或者数据出现了变化，那么 slave node 应该根据不同的 run id 区分。      


**无磁盘化复制**        

**master 在内存中直接创建 RDB ，然后发送给 slave，不会在自己本地落地磁盘了**。只需要在配置文件中开启 repl-diskless-sync yes 即可。


repl-diskless-sync yes

等待 5s 后再开始复制，因为要等更多 slave 重新连接过来       

repl-diskless-sync-delay 5

**过期 key 处理**       

**slave 不会过期 key，只会等待 master 过期 key。如果 master 过期了一个 key，或者通过 LRU 淘汰了一个 key，那么会模拟一条 del 命令发送给 slave**。        

**复制的完整流程**      

**slave node 启动时，会在自己本地保存 master node 的信息**，包括 master node 的 host 和 ip ，但是复制流程没开始。

**slave node 内部有个定时任务，每秒检查是否有新的 master node 要连接和复制，如果发现，就跟 master node 建立 socket 网络连接。然后 slave node 发送 ping 命令给 master node。如果 master 设置了 requirepass，那么 slave node 必须发送 masterauth 的口令过去进行认证。master node 第一次执行全量复制，将所有数据发给 slave node。而在后续，master node 持续将写命令，异步复制给 slave node**。        

**全量复制**        

* master 执行 bgsave ，在本地生成一份 rdb 快照文件。
* master node 将 rdb 快照文件发送给 slave node，如果 rdb 复制时间超过 60秒（repl-timeout），那么 slave node 就会认为复制失败，可以适当调大这个参数(对于千兆网卡的机器，一般每秒传输 100MB，6G 文件，很可能超过 60s)
* master node 在生成 rdb 时，会将所有新的写命令缓存在内存中，在 slave node 保存了 rdb 之后，再将新的写命令复制给 slave node。
* 如果在复制期间，内存缓冲区持续消耗超过 64MB，或者一次性超过 256MB，那么停止复制，复制失败。     

client-output-buffer-limit slave 256MB 64MB 60      

* **slave node 接收到 rdb 之后，清空自己的旧数据，然后重新加载 rdb 到自己的内存中，同时基于旧的数据版本对外提供服务**。
* 如果 slave node 开启了 AOF，那么会立即执行 BGREWRITEAOF，重写 AOF。


**增量复制**        

* 如果**全量复制过程中，master-slave 网络连接断掉，那么 slave 重新连接 master 时，会触发增量复制**。
* master 直接从自己的 backlog 中获取部分丢失的数据，发送给 slave node，默认 backlog 就是 1MB。
* **master 就是根据 slave 发送的 psync 中的 offset 来从 backlog 中获取数据的**。      

**heartbeat**       

主从节点互相都会发送 heartbeat 信息。
master 默认每隔 10秒 发送一次 heartbeat，slave node 每隔 1秒 发送一个 heartbeat。        

**异步复制**        

**master 每次接收到写命令之后，先在内部写入数据，然后异步发送给 slave node**。      

**Redis 如何才能做到高可用**        

如果系统在 365 天内，有 99.99% 的时间，都是可以哗哗对外提供服务的，那么就说系统是高可用的。

一个 slave 挂掉了，是不会影响可用性的，还有其它的 slave 在提供相同数据下的相同的对外的查询服务。

但是，**如果 master node 死掉了，会怎么样？没法写数据了，写缓存的时候，全部失效了**。slave node 还有什么用呢，**没有 master 给它们复制数据了，系统相当于不可用了**。

**Redis 的高可用架构，叫做 failover 故障转移，也可以叫做主备切换**。

**master node 在故障时，自动检测，并且将某个 slave node 自动切换为 master node 的过程，叫做主备切换**。这个过程，实现了 Redis 的**主从架构下的高可用**。      

**Redis 哨兵集群实现高可用**        


哨兵的介绍
sentinel，中文名是哨兵。哨兵是 Redis 集群架构中非常重要的一个组件，主要有以下功能：

* 集群监控：负责监控 Redis master 和 slave 进程是否正常工作。
* 消息通知：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
* 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
* 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。      

**哨兵用于实现 Redis 集群的高可用，本身也是分布式的**，作为一个哨兵集群去运行，互相协同工作。

* 故障转移时，判断一个 master node 是否宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题。
* 即使部分哨兵节点挂掉了，哨兵集群还是能正常工作的，因为如果一个作为高可用机制重要组成部分的故障转移系统本身是单点的，那就很坑爹了。


**哨兵的核心知识**      

* 哨兵至少需要 3 个实例，来保证自己的健壮性。
* 哨兵 + Redis 主从的部署架构，**是不保证数据零丢失的**，只能保证 Redis 集群的高可用性。
* 对于哨兵 + Redis 主从这种复杂的部署架构，尽量在测试环境和生产环境，都进行充足的测试和演练。        

哨兵集群必须部署 2 个以上节点，如果哨兵集群仅仅部署了 2 个哨兵实例，quorum = 1。

配置 quorum=1 ，如果 master 宕机， s1 和 s2 中只要有 1 个哨兵认为 master 宕机了，就可以进行切换，同时 s1 和 s2 会选举出一个哨兵来执行故障转移。但是同时这个时候，需要 majority，也就是大多数哨兵都是运行的。  

如果此时仅仅是 M1 进程宕机了，哨兵 s1 正常运行，那么故障转移是 OK 的。但是如果是整个 M1 和 S1 运行的机器宕机了，那么哨兵只有 1 个，此时就没有 majority 来允许执行故障转移，虽然另外一台机器上还有一个 R1，但是故障转移不会执行。


```
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+
```
配置 quorum=2 ，如果 M1 所在机器宕机了，那么三个哨兵还剩下 2 个，S2 和 S3 可以一致认为 master 宕机了，然后选举出一个来执行故障转移，同时 3 个哨兵的 majority 是 2，所以还剩下的 2 个哨兵运行着，就可以允许执行故障转移。

**Redis 哨兵主备切换的数据丢失问题**        

**导致数据丢失的两种情况**      

主备切换的过程，可能会导致数据丢失：

* 异步复制导致的数据丢失        

因为**master->slave 的复制是异步的，所以可能有部分数据还没复制到 slave，master 就宕机了**，此时这部分数据就丢失了。

* 脑裂导致的数据丢失        

脑裂，也就是说，**某个 master 所在机器突然脱离了正常的网络，跟其他 slave 机器不能连接，但是实际上 master 还运行着。此时哨兵可能就会认为 master 宕机了，然后开启选举，将其他 slave 切换成了 master。这个时候，集群里就会有两个 master ，也就是所谓的脑裂**。

此时**虽然某个 slave 被切换成了 master，但是可能 client 还没来得及切换到新的 master，还继续向旧 master 写数据。因此旧 master 再次恢复的时候，会被作为一个 slave 挂到新的 master 上去，自己的数据会清空，重新从新的 master 复制数据。而新的 master 并没有后来 client 写入的数据，因此，这部分数据也就丢失了**。        

**数据丢失问题的解决方案**      

进行如下配置：

min-slaves-to-write 1       
min-slaves-max-lag 10
表示，**要求至少有 1 个 slave，数据复制和同步的延迟不能超过 10 秒**。

如果说一旦所有的 slave，数据复制和同步的延迟都超过了 10 秒钟，那么这个时候，master 就不会再接收任何请求了。     

*减少异步复制数据的丢失     

有了 min-slaves-max-lag 这个配置，就可以确保说，一旦 slave 复制数据和 ack 延时太长，就认为可能 master 宕机后损失的数据太多了，那么就拒绝写请求，这样可以把 master 宕机时由于部分数据未同步到 slave 导致的数据丢失降低的可控范围内。

*减少脑裂的数据丢失     

如果一个 master 出现了脑裂，跟其他 slave 丢了连接，那么上面两个配置可以确保说，如果不能继续给指定数量的 slave 发送数据，而且 slave 超过 10 秒没有给自己 ack 消息，那么就直接拒绝客户端的写请求。因此在脑裂场景下，最多就丢失 10 秒的数据。          

**sdown 和 odown 转换机制**     

***sdown 是主观宕机，就一个哨兵如果自己觉得一个 master 宕机了，那么就是主观宕机**
***odown 是客观宕机，如果 quorum 数量的哨兵都觉得一个 master 宕机了，那么就是客观宕机**      

**sdown 达成的条件很简单，如果一个哨兵 ping 一个 master，超过了 is-master-down-after-milliseconds 指定的毫秒数之后，就主观认为 master 宕机了**；如果一个哨兵在指定时间内，收到了 quorum 数量的其它哨兵也认为那个 master 是 sdown 的，那么就认为是 odown 了。

**哨兵集群的自动发现机制**      

**哨兵互相之间的发现，是通过 Redis 的 pub/sub 系统实现的，每个哨兵都会往 __sentinel__:hello 这个 channel 里发送一个消息**，这时候所有其他哨兵都可以消费到这个消息，并感知到其他的哨兵的存在。

每隔两秒钟，每个哨兵都会往自己监控的某个 master+slaves 对应的 __sentinel__:hello channel 里发送一个消息，内容是自己的 host、ip 和 runid 还有对这个 master 的监控配置。

**每个哨兵也会去监听自己监控的每个 master+slaves 对应的 __sentinel__:hello channel，然后去感知到同样在监听这个 master+slaves 的其他哨兵的存在**。

**每个哨兵还会跟其他哨兵交换对 master 的监控配置，互相进行监控配置的同步**。      

**slave 配置的自动纠正**        

哨兵会负责自动纠正 slave 的一些配置，比如 slave 如果要成为潜在的 master 候选人，哨兵会确保 slave 复制现有 master 的数据；如果 slave 连接到了一个错误的 master 上，比如故障转移之后，那么哨兵会确保它们连接到正确的 master 上。


**slave->master 选举算法**      

**如果一个 master 被认为 odown 了，而且 majority 数量的哨兵都允许主备切换，那么某个哨兵就会执行主备切换操作**，此时首先要选举一个 slave 来，会考虑 slave 的一些信息：

* 跟 master 断开连接的时长
* slave 优先级
* 复制 offset
* run id        
* 
如果一个 slave 跟 master 断开连接的时间已经超过了 down-after-milliseconds 的 10 倍，外加 master 宕机的时长，那么 slave 就被认为不适合选举为 master。

(down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state     

接下来会对 slave 进行排序：

* **按照 slave 优先级进行排序**，slave priority 越低，优先级就越高。
* 如果 slave priority 相同，那么**看 replica offset，哪个 slave 复制了越多的数据，offset 越靠后，优先级就越高**。
* 如果上面两个条件都相同，那么**选择一个 run id 比较小的那个 slave**。


**quorum 和 majority**      

**每次一个哨兵要做主备切换，首先需要 quorum 数量的哨兵认为 odown**，然后选举出一个哨兵来做切换，这个哨兵还需要得到 majority 哨兵的授权，才能正式执行切换。

**如果 quorum < majority**，比如 5 个哨兵，majority 就是 3，quorum 设置为 2，那么就 3 个哨兵授权就可以执行切换。

但是**如果 quorum >= majority**，那么必须 quorum 数量的哨兵都授权，比如 5 个哨兵，quorum 是 5，那么必须 5 个哨兵都同意授权，才能执行切换。

**configuration epoch**     

哨兵会对一套 Redis master+slaves 进行监控，有相应的监控的配置。

**执行切换的那个哨兵，会从要切换到的新 master（salve->master）那里得到一个 configuration epoch，这就是一个 version 号，每次切换的 version 号都必须是唯一的**。

**如果第一个选举出的哨兵切换失败了，那么其他哨兵，会等待 failover-timeout 时间，然后接替继续执行切换，此时会重新获取一个新的 configuration epoch，作为新的 version 号**。

**configuration 传播**      

**哨兵完成切换之后，会在自己本地更新生成最新的 master 配置，然后同步给其他的哨兵，就是通过之前说的 pub/sub 消息机制**。

这里之前的 **version 号就很重要了，因为各种消息都是通过一个 channel 去发布和监听的，所以一个哨兵完成一次新的切换之后，新的 master 配置是跟着新的 version 号的。其他的哨兵都是根据版本号的大小来更新自己的 master 配置的**。     

**Redis 的持久化有哪几种方式？不同的持久化机制都有什么优缺点？持久化机制具体底层是如何实现的？**          

**Redis 持久化的两种方式**      

* RDB：**RDB 持久化机制，是对 Redis 中的数据执行周期性的持久化**。
* AOF：**AOF 机制对每条写入命令作为日志，以 append-only 的模式写入一个日志文件中，在 Redis 重启的时候，可以通过回放 AOF 日志中的写入指令来重新构建整个数据集**。      


通过 RDB 或 AOF，都可以将 Redis 内存中的数据给持久化到磁盘上面来，然后可以将这些数据备份到别的地方去，比如说阿里云等云服务。

如果 Redis 挂了，服务器上的内存和磁盘上的数据都丢了，可以从云服务上拷贝回来之前的数据，放到指定的目录中，然后重新启动 Redis，Redis 就会自动根据持久化数据文件中的数据，去恢复内存中的数据，继续对外提供服务。

如果**同时使用 RDB 和 AOF 两种持久化机制，那么在 Redis 重启的时候，会使用 AOF 来重新构建数据，因为 AOF 中的数据更加完整**。        

**RDB 优缺点**      

* **RDB 会生成多个数据文件，每个数据文件都代表了某一个时刻中 Redis 的数据，这种多个数据文件的方式，非常适合做冷备**，可以将这种完整的数据文件发送到一些远程的安全存储上去，比如说 Amazon 的 S3 云服务上去，在国内可以是阿里云的 ODPS 分布式存储上，以预定好的备份策略来定期备份 Redis 中的数据。

* **RDB 对 Redis 对外提供的读写服务，影响非常小，可以让 Redis 保持高性能**，因为 Redis 主进程只需要 fork 一个子进程，让子进程执行磁盘 IO 操作来进行 RDB 持久化即可。

* 相对于 AOF 持久化机制来说，**直接基于 RDB 数据文件来重启和恢复 Redis 进程，更加快速**。

* 如果**想要在 Redis 故障时，尽可能少的丢失数据，那么 RDB 没有 AOF 好。一般来说，RDB 数据快照文件，都是每隔 5 分钟，或者更长时间生成一次，这个时候就得接受一旦 Redis 进程宕机，那么会丢失最近 5 分钟的数据**。

* **RDB 每次在 fork 子进程来执行 RDB 快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒**。

**AOF 优缺点**      

* AOF 可以更好的保护数据不丢失，**一般 AOF 会每隔 1 秒，通过一个后台线程执行一次 fsync 操作，最多丢失 1 秒钟的数据**。
* **AOF 日志文件以 append-only 模式写入，所以没有任何磁盘寻址的开销，写入性能非常高**，而且文件不容易破损，即使文件尾部破损，也很容易修复。
* AOF 日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在 rewrite log 的时候，会对其中的指令进行压缩，创建出一份需要恢复数据的最小日志出来。在创建新日志文件的时候，老的日志文件还是照常写入。当新的 merge 后的日志文件 ready 的时候，再交换新老日志文件即可。
* **AOF 日志文件的命令通过可读较强的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复**。比如某人不小心用 flushall 命令清空了所有数据，只要这个时候后台 rewrite 还没有发生，那么就可以立即拷贝 AOF 文件，将最后一条 flushall 命令给删了，然后再将该 AOF 文件放回去，就可以通过恢复机制，自动恢复所有数据。
* **对于同一份数据来说，AOF 日志文件通常比 RDB 数据快照文件更大**。
* **AOF 开启后，支持的写 QPS 会比 RDB 支持的写 QPS 低，因为 AOF 一般会配置成每秒 fsync 一次日志文件**，当然，每秒一次 fsync ，性能也还是很高的。（如果实时写入，那么 QPS 会大降，Redis 性能会大大降低）
* 以前 AOF 发生过 bug，就是通过 AOF 记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似 AOF 这种较为复杂的基于命令日志 / merge / 回放的方式，比基于 RDB 每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有 bug。不过 AOF 就是为了避免 rewrite 过程导致的 bug，因此每次 rewrite 并不是基于旧的指令日志进行 merge 的，而是基于当时内存中的数据进行指令的重新构建，这样健壮性会好很多。

**RDB 和 AOF 到底该如何选择**       

* 不要仅仅使用 RDB，因为那样会导致你丢失很多数据；
* 也不要仅仅使用 AOF，因为那样有两个问题：第一，你通过 AOF 做冷备，没有 RDB 做冷备来的恢复速度更快；第二，RDB 每次简单粗暴生成数据快照，更加健壮，可以避免 AOF 这种复杂的备份和恢复机制的 bug；
* **Redis 支持同时开启开启两种持久化方式，我们可以综合使用 AOF 和 RDB 两种持久化机制，用 AOF 来保证数据不丢失，作为数据恢复的第一选择; 用 RDB 来做不同程度的冷备，在 AOF 文件都丢失或损坏不可用的时候，还可以使用 RDB 来进行快速的数据恢复**。        

**Redis 集群模式的工作原理能说一下么？在集群模式下，Redis 的 key 是如何寻址的？分布式寻址都有哪些算法？了解一致性 hash 算法吗？**


**Redis cluster 介绍**      

* 自动将数据进行分片，每个 master 上放一部分数据
* 提供内置的高可用支持，部分 master 不可用时，还是可以继续工作的   


**在 Redis cluster 架构下，每个 Redis 要放开两个端口号**，比如一个是 6379，**另外一个就是 加1w 的端口号**，比如 16379。

**16379 端口号是用来进行节点间通信的，也就是 cluster bus 的东西**，cluster bus 的通信，**用来进行故障检测、配置更新、故障转移授权。cluster bus 用了另外一种二进制的协议， gossip 协议**，用于节点间进行高效的数据交换，占用更少的网络带宽和处理时间。      

**节点间的内部通信机制**        

**基本通信原理**        

集群元数据的维护有两种方式：集中式、Gossip 协议。**Redis cluster 节点间采用 gossip 协议进行通信**。

**集中式是将集群元数据（节点信息、故障等等）几种存储在某个节点上**。集中式元数据集中存储的一个典型代表，就是大数据领域的 storm 。它是分布式的大数据实时计算引擎，是集中式的元数据存储的结构，底层基于 zookeeper（分布式协调的中间件）对所有元数据进行存储维护。

Redis 维护集群元数据采用另一个方式， **gossip 协议，所有节点都持有一份元数据，不同的节点如果出现了元数据的变更，就不断将元数据发送给其它的节点，让其它节点也进行元数据的变更**。

**集中式的好处在于，元数据的读取和更新，时效性非常好**，一旦元数据出现了变更，就立即更新到集中式的存储中，其它节点读取的时候就可以感知到；**不好在于，所有的元数据的更新压力全部集中在一个地方，可能会导致元数据的存储有压力**。      

**gossip 好处在于，元数据的更新比较分散**，不是集中在一个地方，**更新请求会陆陆续续打到所有节点上去更新，降低了压力**；**不好在于，元数据的更新有延时，可能导致集群中的一些操作会有一些滞后**。

* 10000 端口：**每个节点都有一个专门用于节点间通信的端口，就是自己提供服务的端口号+10000**，比如 7001，那么用于节点间通信的就是 17001 端口。**每个节点每隔一段时间都会往另外几个节点发送 ping 消息，同时其它几个节点接收到 ping 之后返回 pong** 。

* 交换的信息：**信息包括故障信息，节点的增加和删除，hash slot 信息等等**。        

**gossip 协议**     

gossip 协议包含多种消息，包含 ping , pong , meet , fail 等等。

* meet：**某个节点发送 meet 给新加入的节点，让新节点加入集群中**，然后新节点就会开始与其它节点进行通信。      

Redis-trib.rb add-node      

其实内部就是发送了一个 gossip meet 消息给新加入的节点，通知那个节点去加入我们的集群。

* ping：**每个节点都会频繁给其它节点发送 ping，其中包含自己的状态还有自己维护的集群元数据，互相通过 ping 交换元数据**。
* pong：**返回 ping 和 meet，包含自己的状态和其它信息，也用于信息广播和更新**。
* fail：**某个节点判断另一个节点 fail 之后，就发送 fail 给其它节点**，通知其它节点说，某个节点宕机啦。

**ping 消息深入**       

ping 时要携带一些元数据，如果很频繁，可能会加重网络负担。

**每个节点每秒会执行 10 次 ping，每次会选择 5 个最久没有通信的其它节点。当然如果发现某个节点通信延时达到了 cluster_node_timeout / 2 ，那么立即发送 ping，避免数据交换延时过长**，落后的时间太长了。比如说，两个节点之间都 10 分钟没有交换数据了，那么整个集群处于严重的元数据不一致的情况，就会有问题。所以 **cluster_node_timeout 可以调节，如果调得比较大，那么会降低 ping 的频率**。

**每次 ping，会带上自己节点的信息，还有就是带上 1/10 其它节点的信息**，发送出去，进行交换。至少包含 3 个其它节点的信息，最多包含 总节点数减 2 个其它节点的信息。      


**Redis cluster 的高可用与主备切换原理**        

Redis cluster 的高可用的原理，几乎跟哨兵是类似的。

**判断节点宕机**    

**如果一个节点认为另外一个节点宕机，那么就是 pfail ，主观宕机。如果多个节点都认为另外一个节点宕机了，那么就是 fail ，客观宕机**，跟哨兵的原理几乎一样，sdown，odown。

在 **cluster-node-timeout 内，某个节点一直没有返回 pong ，那么就被认为 pfail** 。

如果**一个节点认为某个节点 pfail 了，那么会在 gossip ping 消息中， ping 给其他节点，如果超过半数的节点都认为 pfail 了，那么就会变成 fail** 。

**从节点过滤**      

对宕机的 master node，从其所有的 slave node 中，选择一个切换成 master node。

**检查每个 slave node 与 master node 断开连接的时间，如果超过了 cluster-node-timeout * cluster-slave-validity-factor ，那么就没有资格切换成 master** 。

**从节点选举**      

**每个从节点，都根据自己对 master 复制数据的 offset，来设置一个选举时间，offset 越大（复制数据越多）的从节点，选举时间越靠前，优先进行选举**。

**所有的 master node 开始 slave 选举投票，给要进行选举的 slave 进行投票，如果大部分 master node （N/2 + 1） 都投票给了某个从节点，那么选举通过，那个从节点可以切换成 master**。

从节点执行主备切换，从节点切换为主节点。

**与哨兵比较**      

整个流程跟哨兵相比，非常类似，所以说，**Redis cluster 功能强大，直接集成了 replication 和 sentinel 的功能**。


**分布式寻址算法**      

* hash 算法（大量缓存重建）
* 一致性 hash 算法（自动缓存迁移）+ 虚拟节点（自动负载均衡）
* Redis cluster 的 hash slot 算法

**hash 算法**       

**来了一个 key，首先计算 hash 值，然后对节点数取模**。然后打在不同的 master 节点上。**一旦某一个 master 节点宕机，所有请求过来，都会基于最新的剩余 master 节点数去取模**，尝试去取数据。这会**导致大部分的请求过来，全部无法拿到有效的缓存，导致大量的流量涌入数据库**。        

**一致性 hash 算法**        

**一致性 hash 算法将整个 hash 值空间组织成一个虚拟的圆环，整个空间按顺时针方向组织**，下一步**将各个 master 节点（使用服务器的 ip 或主机名）进行 hash。这样就能确定每个节点在其哈希环上的位置**。

**来了一个 key，首先计算 hash 值，并确定此数据在环上的位置，从此位置沿环顺时针“行走”，遇到的第一个 master 节点就是 key 所在位置**。

在一致性哈希算法中，**如果一个节点挂了，受影响的数据仅仅是此节点到环空间前一个节点（沿着逆时针方向行走遇到的第一个节点）之间的数据**，其它不受影响。增加一个节点也同理。

然而，**一致性哈希算法在节点太少时，容易因为节点分布不均匀而造成缓存热点的问题**。为了解决这种热点问题，**一致性 hash 算法引入了虚拟节点机制，即对每一个节点计算多个 hash，每个计算结果位置都放置一个虚拟节点**。这样就实现了数据的均匀分布，负载均衡。

**Redis cluster 的 hash slot 算法**     

**Redis cluster 有固定的 16384 个 hash slot，对每个 key 计算 CRC16 值，然后对 16384 取模，可以获取 key 对应的 hash slot**。

Redis cluster 中每个 master 都会持有部分 slot，比如有 3 个 master，那么可能每个 master 持有 5000 多个 hash slot。**hash slot 让 node 的增加和移除很简单，增加一个 master，就将其他 master 的 hash slot 移动部分过去，减少一个 master，就将它的 hash slot 移动到其他 master 上去**。移动 hash slot 的成本是非常低的。客户端的 api，可以对指定的数据，让他们走同一个 hash slot，通过 hash tag 来实现。

任何一台机器宕机，另外两个节点，不影响的。因为 key 找的是 hash slot，不是机器。

### 分库分表        

**为什么要分库分表（设计高并发系统的时候，数据库层面该如何设计）？用过哪些分库分表中间件？不同的分库分表中间件都有什么优点和缺点？你们具体是如何对数据库如何进行垂直拆分或水平拆分的？**        

**为什么要分库分表？（设计高并发系统的时候，数据库层面该如何设计？）**

**分表**    

比如你单表都几千万数据了，你确定你能扛住么？绝对不行，**单表数据量太大，会极大影响你的 sql 执行的性能**，到了后面你的 sql 可能就跑的很慢了。一般来说，就以我的经验来看，单表到几百万的时候，性能就会相对差一些了，你就得分表了。

分表是啥意思？就是把一个表的数据放到多个表中，然后查询的时候你就查一个表。比如按照用户 id 来分表，将一个用户的数据就放在一个表中。然后操作的时候你对一个用户就操作那个表就好了。这样可以控制每个表的数据量在可控的范围内，比如每个表就固定在 200 万以内。

**分库**        

分库是啥意思？就是**你一个库一般我们经验而言，最多支撑到并发 2000，一定要扩容了**，而且一个健康的单库并发值你最好保持在每秒 1000 左右，不要太大。那么你可以将一个库的数据拆分到多个库中，访问的时候就访问一个库好了。


```
# 	                 分库分表前	                      分库分表后
并发支撑情况	MySQL 单机部署，扛不住高并发	MySQL从单机到多机，能承受的并发增加了多倍
磁盘使用情况	MySQL 单机磁盘容量几乎撑满	拆分为多个库，数据库服务器磁盘使用率大大降低
SQL 执行性能	单表数据量太大，SQL 越跑越慢	单表数据量减少，SQL 执行效率明显提升
```

**用过哪些分库分表中间件？不同的分库分表中间件都有什么优点和缺点？**

**Sharding-jdbc**       

当当开源的，属于 client 层方案，是 ShardingSphere 的 client 层方案， ShardingSphere 还提供 proxy 层的方案 Sharding-Proxy。确实之前用的还比较多一些，因为 SQL 语法支持也比较多，没有太多限制，而且截至 2019.4，已经推出到了 4.0.0-RC1 版本，支持分库分表、读写分离、分布式 id 生成、柔性事务（最大努力送达型事务、TCC 事务）。而且确实之前使用的公司会比较多一些（这个在官网有登记使用的公司，可以看到从 2017 年一直到现在，是有不少公司在用的），目前社区也还一直在开发和维护，还算是比较活跃，个人认为算是一个现在也可以选择的方案。

**Mycat**       

基于 Cobar 改造的，属于 proxy 层方案，支持的功能非常完善，而且目前应该是非常火的而且不断流行的数据库中间件，社区很活跃，也有一些公司开始在用了。但是确实相比于 Sharding jdbc 来说，年轻一些，经历的锤炼少一些。

**总结**        

综上，现在其实建议考量的，就是 Sharding-jdbc 和 Mycat，这两个都可以去考虑使用。

**Sharding-jdbc 这种 client 层方案的优点在于不用部署，运维成本低，不需要代理层的二次转发请求，性能很高**，但是如果遇到升级啥的需要各个系统都重新升级版本再发布，各个系统都需要耦合 Sharding-jdbc 的依赖；

**Mycat 这种 proxy 层方案的缺点在于需要部署**，自己运维一套中间件，运维成本高，**但是好处在于对于各个项目是透明的**，如果遇到升级之类的都是自己中间件那里搞就行了。

通常来说，这两个方案其实都可以选用，但是我个人建议中小型公司选用 Sharding-jdbc，client 层方案轻便，而且维护成本低，不需要额外增派人手，而且中小型公司系统复杂度会低一些，项目也没那么多；但是中大型公司最好还是选用 Mycat 这类 proxy 层方案，因为可能大公司系统和项目非常多，团队很大，人员充足，那么最好是专门弄个人来研究和维护 Mycat，然后大量项目直接透明使用即可。


**你们具体是如何对数据库如何进行垂直拆分或水平拆分的？**        

**水平拆分的意思，就是把一个表的数据给弄到多个库的多个表里去，但是每个库的表结构都一样，只不过每个库表放的数据是不同的**，所有库表的数据加起来就是全部数据。**水平拆分的意义，就是将数据均匀放更多的库里，然后用多个库来扛更高的并发，还有就是用多个库的存储容量来进行扩容。**

**垂直拆分的意思，就是把一个有很多字段的表给拆分成多个表，或者是多个库上去。每个库表的结构都不一样，每个库表都包含部分字段**。**一般来说，会将较少的访问频率很高的字段放到一个表里去，然后将较多的访问频率很低的字段放到另外一个表里去**。因为数据库是有缓存的，**你访问频率高的行字段越少，就可以在缓存里缓存更多的行，性能就越好**。这个一般在表层面做的较多一些。

还有表层面的拆分，就是分表，将一个表变成 N 个表，就是让每个表的数据量控制在一定范围内，保证 SQL 的性能。否则单表数据量越大，SQL 性能就越差。一般是 200 万行左右，不要太多，但是也得看具体你怎么操作，也可能是 500 万，或者是 100 万。你的SQL越复杂，就最好让单表行数越少。

好了，**无论分库还是分表，上面说的那些数据库中间件都是可以支持的**。就是基本上那些中间件可以做到你分库分表之后，**中间件可以根据你指定的某个字段值，比如说 userid，自动路由到对应的库上去，然后再自动路由到对应的表里去**。

你就得考虑一下，你的项目里该如何分库分表？一般来说，垂直拆分，你可以在表层面来做，对一些字段特别多的表做一下拆分；水平拆分，你可以说是并发承载不了，或者是数据量太大，容量承载不了，你给拆了，按什么字段来拆，你自己想好；分表，你考虑一下，你如果哪怕是拆到每个库里去，并发和容量都 ok 了，但是每个库的表还是太大了，那么你就分表，将这个表分开，保证每个表的数据量并不是很大。

**现在有一个未分库分表的系统，未来要分库分表，如何设计才可以让系统从未分库分表动态切换到分库分表上？**        

**停机迁移方案**        

我先给你说一个最 low 的方案，就是很简单，大家伙儿凌晨 12 点开始运维，网站或者 app 挂个公告，说 0 点到早上 6 点进行运维，无法访问。

接着到 0 点停机，系统停掉，没有流量写入了，此时老的单库单表数据库静止了。然后你之前得写好一个导数的一次性工具，此时直接跑起来，然后将单库单表的数据哗哗哗读出来，写到分库分表里面去。

导数完了之后，就 ok 了，修改系统的数据库连接配置啥的，包括可能代码和 SQL 也许有修改，那你就用最新的代码，然后直接启动连到新的分库分表上去。

**双写迁移方案**        

这个是我们常用的一种迁移方案，比较靠谱一些，不用停机，不用看北京凌晨 4 点的风景。

简单来说，就是**在线上系统里面，之前所有写库的地方，增删改操作，除了对老库增删改，都加上对新库的增删改，这就是所谓的双写，同时写俩库，老库和新库**。

然后**系统部署之后，新库数据差太远，用之前说的导数工具，跑起来读老库数据写新库，写的时候要根据 gmt_modified 这类字段判断这条数据最后修改的时间**，除非是读出来的数据在新库里没有，或者是比新库的数据新才会写。简单来说，**就是不允许用老数据覆盖新数据**。

**导完一轮之后**，有可能数据还是存在不一致，那么就**程序自动做一轮校验**，比对新老库每个表的每条数据，接着如果有不一样的，就针对那些不一样的，从老库读数据再次写。**反复循环，直到两个库每个表的数据都完全一致为止**。

接着当数据完全一致了，就 ok 了，基于仅仅使用分库分表的最新代码，重新部署一次，不就仅仅基于分库分表在操作了么，还没有几个小时的停机时间，很稳。所以现在基本玩儿数据迁移之类的，都是这么干的。

**如何设计可以动态扩容缩容的分库分表方案？**        

**停机扩容（不推荐）**      

这个方案就跟停机迁移一样，步骤几乎一致，唯一的一点就是那个导数的工具，是把现有库表的数据抽出来慢慢倒入到新的库和表里去。但是最好别这么玩儿，有点不太靠谱，因为既然分库分表就说明数据量实在是太大了，可能多达几亿条，甚至几十亿，你这么玩儿，可能会出问题。

从单库单表迁移到分库分表的时候，数据量并不是很大，单表最大也就两三千万。那么你写个工具，多弄几台机器并行跑，1小时数据就导完了。这没有问题。

如果 3 个库 + 12 个表，跑了一段时间了，数据量都 1~2 亿了。光是导 2 亿数据，都要导个几个小时，6 点，刚刚导完数据，还要搞后续的修改配置，重启系统，测试验证，10 点才可以搞完。所以不能这么搞。      

**优化后的方案**

1.设定好几台数据库服务器，每台服务器上几个库，每个库多少个表，推荐是 32 库 * 32 表，对于大部分公司来说，可能几年都够了。      
2.路由的规则，orderId 模 32 = 库，orderId / 32 模 32 = 表       
3.扩容的时候，申请增加更多的数据库服务器，装好 MySQL，呈倍数扩容，4 台服务器，扩到 8 台服务器，再到 16 台服务器。       
4.由 DBA 负责将原先数据库服务器的库，迁移到新的数据库服务器上去，库迁移是有一些便捷的工具的。      
5.我们这边就是修改一下配置，调整迁移的库所在数据库服务器的地址。       
6.重新发布系统，上线，原先的路由规则变都不用变，直接可以基于 n 倍的数据库服务器的资源，继续进行线上系统的提供服务。

**分库分表之后，id 主键如何处理？**     

**数据库自增 id**       

这个就是说你的系统里每次得到一个 id，都是往一个库的一个表里插入一条没什么业务含义的数据，然后获取一个数据库自增的一个 id。拿到这个 id 之后再往对应的分库分表里去写入。

这个方案的好处就是方便简单，谁都会用；缺点就是单库生成自增 id，要是高并发的话，就会有瓶颈的；如果你硬是要改进一下，那么就专门开一个服务出来，这个服务每次就拿到当前 id 最大值，然后自己递增几个 id，一次性返回一批 id，然后再把当前最大 id 值修改成递增几个 id 之后的一个值；但是无论如何都是基于单个数据库。

适合的场景：你分库分表就俩原因，要不就是单库并发太高，要不就是单库数据量太大；除非是你并发不高，但是数据量太大导致的分库分表扩容，你可以用这个方案，因为可能每秒最高并发最多就几百，那么就走单独的一个库和表生成自增主键即可。

**设置数据库 sequence 或者表自增字段步长**      

可以通过设置数据库 sequence 或者表的自增字段步长来进行水平伸缩。

比如说，现在有 8 个服务节点，每个服务节点使用一个 sequence 功能来产生 ID，每个 sequence 的起始 ID 不同，并且依次递增，步长都是 8。

适合的场景：在用户防止产生的 ID 重复时，这种方案实现起来比较简单，也能达到性能目标。但是服务节点固定，步长也固定，将来如果还要增加服务节点，就不好搞了。

**snowflake 算法**      

snowflake 算法是 twitter 开源的分布式 id 生成算法，采用 Scala 语言实现，是把一个 64 位的 long 型的 id，1 个 bit 是不用的，用其中的 41 bits 作为毫秒数，用 10 bits 作为工作机器 id，12 bits 作为序列号。

* 1 bit：不用，为啥呢？因为二进制里第一个 bit 为如果是 1，那么都是负数，但是我们生成的 id 都是正数，所以第一个 bit 统一都是 0。        
* 41 bits：表示的是时间戳，单位是毫秒。41 bits 可以表示的数字多达 2^41 - 1 ，也就是可以标识 2^41 - 1 个毫秒值，换算成年就是表示69年的时间。      
* 10 bits：记录工作机器 id，代表的是这个服务最多可以部署在 2^10 台机器上，也就是 1024 台机器。但是 10 bits 里 5 个 bits 代表机房 id，5 个 bits 代表机器 id。意思就是最多代表 2^5 个机房（32 个机房），每个机房里可以代表 2^5 个机器（32台机器）。       
* 12 bits：这个是用来记录同一个毫秒内产生的不同 id，12 bits 可以代表的最大正整数是 2^12 - 1 = 4096 ，也就是说可以用这个 12 bits 代表的数字来区分同一个毫秒内的 4096 个不同的 id。
