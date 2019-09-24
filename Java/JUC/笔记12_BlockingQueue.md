# 阻塞队列(BlockingQueue )

## 介绍

​	主要用于生产者消费者模式,在多线程场景时生产者线程在队列尾部添加元素,而消费者线程则在队列头部消费元素,通过这种方式能够达到将任务的生产和消费进行隔离的目的

#### 继承关系

![Queue继承关系](E:\Git\秋招_基础知识\Java\JUC\图片\阻塞队列\Queue继承关系.jpg)

**总结:** BlockingQueue 是一个队列, 级别相当于 和List

#### 使用

- 实现类 ArrayBlockingQueue 等价于 ArrayList
- LinkedBlockingQueue 等价于 LinkedList

#### 作用

![阻塞队列](E:\Git\秋招_基础知识\Java\JUC\图片\阻塞队列\阻塞队列.jpg)

- 当阻塞队列是空时, 从队列中获取元素的操作将会被阻塞
- 当阻塞队列是满时, 从队列中添加元素的操作将会被阻塞

### 种类

都是线程安全的

- ==**ArrayBlockingQueue:**== 由数组结构组成的有界阻塞队列

  > 容量是有限的, 必须在初始化的时候指定容量大小, 先进先出的方式存储数据

- ==**LinkedBlockingQueue:**== 由链表结构组成的有界/无界阻塞队列

  > 可以使用参数指定容量大小
  >
  > 如果默认(大小默认值为 `Integer.MAX_ _VALUE`), 相当于无界了, 先进先出的方式存储数据

- ==**PriorityBlockingQueue:**== 支持优先级排序的无界阻塞队列。

  > 按照优先级顺序被移出, 该队列没有上限. 但是如果元素为空, 操作 take() 就会被阻塞. 该对象的元素是要具备可比性的

- **DelayQueue:**使用优先级队列实现的延迟无界阻塞队列

  > 支持延时获取元素
  >
  > 队列中的元素必须实现 Delay接口, 在创建元素的时候, 可以指定多久(延迟期满)才可以从队列中获取元素. 

- **SynchronousQueue:** 不存储元素的阻塞队列，也即単个元素的队列。(产生一个, 消费一个, 专属定制版)

  > 仅允许容纳一个元素, 当一个线程插入一个元素后, 会被阻塞, 直到另外一个线程将他消费掉. 

- **LinkedTransferQueue:** 由链表结构组成的无界阻塞队列

  >SynchronousQueue 和 LinkedBlockingQueue的合体, 性能比 SynchronousQueue 更高, 因为是无锁操作. 比 SynchronousQueue能存储更多的元素

- **LinkedBlockingDeque:** 由链表结构组成的**双向阻塞队列**

  > 与窃取工作相关联的. 一个消费者, 生产者设计中, 所有的消费者只共享一个工作队列. 在窃取工作中, 每个消费者都有自己的一个双端队列. 如果一个消费者完成了自己队列的全部工作. 他可以窃取其他消费者双端队列里的末尾的任务. 比传统的模式有更好的可伸缩性. 

#### 基本用法

主要是当队列为空或者过满后再次操作的不同反应

| 方法类型       | 抛出异常  | 特殊值(boolean) | 阻塞   | 超时                 |
| :------------- | :-------- | :-------------- | :----- | :------------------- |
| 插入(队列尾部) | add(e)    | offer(e)        | put(e) | offer(e, time, unit) |
| 取出(队列头部) | remove()  | poll()          | take() | poll(time, unit)     |
| 检查(队列头部) | element() | peek()          |        |                      |

#### 说明

| 类型     | 说明                                                         |
| :------- | :----------------------------------------------------------- |
| 抛出异常 | 当阻塞队列满时，再往队列里add插入元素会抛 `IlegalStateException: Queue full`                          当阻塞队列空时，再往队列里remove移除元素会拋`NoSuchElementException` |
| 特殊值   | 插入方法, 成功返回true, 失败返回false                                                                                                                                            移除方法, 成功是返回出队列的元素, 队列没有元素就返回 null |
| 一直阻塞 | 当阻塞队列满时，生产者线程继续往队列里put元素，队列会一 直阻塞生产线程直到put数据or响应中断退出。当阻塞队列空时，消费者线程试图从队列里take元素，队列会:-.直阻塞消费者线程直到队列可用。 |

## 使用

**题目: **一个初始值为零的变量, 两个线程对齐交替操作, 一个 +1, 一个 -1, 来 5轮

### 传统方式

要点: 

1. 线程    操作  资源类(自身高内聚)

2. 判断    干活  通知

3. 防止虚假唤醒机制

```java
public class ProdConsumer_TraditionDemo {

    public static void main(String[] args) {
        MyData data = new MyData();
        // 生产者
        new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                try {
                    data.increment();
                } catch (Exception e) {
                }
            }
        }, "t1").start();

        // 消费者
        new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                try {
                    data.decrement();
                } catch (Exception e) {
                }
            }
        }, "t2").start();
    }
}

// 资源类
class MyData {
    private int number = 0;
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    // 添加
    public void increment() throws Exception {
        lock.lock();
        try {
            // 1.判断
            while (number != 0) {
                // 等待, 不能生产
                condition.await();
            }
            // 2.干活
            number++;
            System.out.println(Thread.currentThread().getName() + "\t" + number);
            // 3.通知唤醒
            condition.signalAll();
        } catch (Exception e) {
        } finally {
            lock.unlock();
        }
    }

    // 减少
    public void decrement() throws Exception {
        lock.lock();
        try {
            // 1.判断
            while (number == 0) {
                // 等待, 不能减少
                condition.await();
            }
            // 2.干活
            number--;
            System.out.println(Thread.currentThread().getName() + "\t" + number);
            // 3.通知唤醒
            condition.signalAll();
        } catch (Exception e) {
        } finally {
            lock.unlock();
        }
    }
}
```

###### 结果

```java
t1	1
t2	0
t1	1
t2	0
t1	1
t2	0
t1	1
t2	0
t1	1
t2	0
```



### 使用 BlockingQueue 

使用 BlockingQueue 解决生产者消费者问题

```java
public class Prodconsumer_BlockQueueDemo {

    public static void main(String[] args) throws Exception {
        MyResource myResource = new MyResource(new ArrayBlockingQueue<>(10));

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t生产者线程启动");
            try {
                myResource.myProd();
            } catch (Exception e) {
            }
        }, "Prod").start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t消费者者线程启动");
            try {
                myResource.myConsumer();
            } catch (Exception e) {
            }
        }, "Consumer").start();


        TimeUnit.SECONDS.sleep(5);

        System.out.println("---------------------");
        System.out.println("5s时间到, 大老板main线程叫停, 活动结束");
        myResource.stop();
    }
}

// 资源类
class MyResource {
    private volatile boolean FLAG = true; // 默认开启, 进行生产 +消费
    private AtomicInteger atomicInteger = new AtomicInteger();

    BlockingQueue<String> blockingQueue = null;

    public MyResource(BlockingQueue<String> blockingQueue) {

        this.blockingQueue = blockingQueue;
        System.out.println(blockingQueue.getClass().getName());
    }

    // 生产
    public void myProd() throws Exception {
        String data = null;
        boolean retValue;
        // 1.判断
        while (FLAG) {
            // 原子版 ++i
            data = atomicInteger.incrementAndGet() + "";
            retValue = blockingQueue.offer(data, 2L, TimeUnit.SECONDS);
            if (retValue) {
                System.out.println(Thread.currentThread().getName() + "\t 插入队列" + data + "成功");
            } else {
                System.out.println(Thread.currentThread().getName() + "\t 插入队列" + data + "失败");
            }
            TimeUnit.SECONDS.sleep(1);
        }
        System.out.println(Thread.currentThread().getName() + "\t 大老板叫停, 表示 FLAG=false， 生产动作结束！");


    }

    public void myConsumer() throws Exception {
        String data = null;
        String result;
        // 1.判断
        while (FLAG) {

            result = blockingQueue.poll(2L, TimeUnit.SECONDS);
            if (null == result || result.equalsIgnoreCase("")) {
                FLAG = false;
                System.out.println(Thread.currentThread().getName() + "\t 超过两秒钟没有取到值, 消费退出");
                return;
            }
            System.out.println(Thread.currentThread().getName() + "\t 消费队列" + result + "成功");
        }
    }

    public void stop() throws Exception {
        this.FLAG = false;
    }
}
```

###### 结果

```
Prod	生产者线程启动
Prod	 插入队列1成功
Consumer	消费者者线程启动
Consumer	 消费队列1成功
Prod	 插入队列2成功
Consumer	 消费队列2成功
Prod	 插入队列3成功
Consumer	 消费队列3成功
Prod	 插入队列4成功
Consumer	 消费队列4成功
Prod	 插入队列5成功
Consumer	 消费队列5成功
---------------------
5s时间到, 大老板main线程叫停, 活动结束
Prod	 大老板叫停, 表示 FLAG=false， 生产动作结束！
Consumer	 超过两秒钟没有取到值, 消费退出
```

