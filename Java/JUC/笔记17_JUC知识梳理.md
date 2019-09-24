## J.U.C知识点梳理

**java.util.concurrent :提供了并发编程的解决方案**

* CAS是java.util.concurrent.atomic包的基础
* AQS是java.util.concurrent.locks包以及一 些常用类比如
  Semophore , ReentrantLock等类的基础

### J.U.C包的分类

* 线程执行器executor
* 锁locks
* 原子变量类atomic
* 并发工具类tools
* 并发集合collections

看不清的话看原图

![JUC结构图](C:\Users\zn\Desktop\ALL\秋招\总复习\Java\JUC\翔仔\图片\JUC结构图.png)





#### JUC分类

* **locks部分:** 显式锁(互斥锁和读写锁)相关;
* **atomic部分:** 原子变量类相关，是构建非阻塞算法的基剧;executor部分:线程池相关;
* **collections部分:** 并发容器相关:
* **tools部分:** 同步T具相关，如信号量、闭锁、棚栏等功能;

#### JUC常用同步器框架AQS

JUC当中的大多数同步器实现都是围绕着共同的基础行为，**比如等待队列、条件队列、独占获取、共享获取等，而这个行为的抽象就是基于AbstractQueuedSynchronizer简称AQS**, AQS是-一个抽象同步框架，可以用来实现一个依赖状态的同步器。

AQS具备特性

* 阻塞等待队列
* 共享/独占
* 公平/非公平
* 可重入
* 允许中断

#### JUC基于AQS的内部实现

JUC当中同步器的实现如Lock,Latch,Barrier等，都是基于AQS框架实现

- 一般通过定义内部类 Sync继承 AQS
- 将同步器所有调用都映射到Sync对应的方法



#### AQS框架 — 管理状态

* AQS内部维护属性 **volatile int state (32位)**
  * state表示资源的可用状态
  * State三种访问方式
    * getState()
    * setState()
    * compareAndSetState()

* AQS定义两种资源共享方式
  * Exclusive独占，只有一个线程能执行， 如ReentrantLock
  * Share共享，多个线程可以同时执行，如Semaphore/CountDownLatch
* AQS定义两种队列
  * 同步等待队列
  * 条件等待队列

#### 同步队列(CLH队列)

CLH队列是Craig、Landin、 Hagersten三人发明的一种基于双向链表数据结构的队列, Java中的CLH队列是原CLH队列的一个变种,线程由原自旋机制改为阻塞机制。

FIFO先入先出线程等待队列

![同步队列](C:\Users\zn\Desktop\ALL\秋招\总复习\Java\JUC\翔仔\图片\synchronized底层\同步队列.png)

