## synchronized 介绍

### 预备知识

#### 线程安全问题的主要诱因

多线程编程中,有可能会出现多个线程同时访问同一个**共享,可变**资源的情况

* 资源可能是: 对象, 变量,文件等

* 共享: 资源可以有多个线程同时访问

* 可变: 资源可以在其生命周期内被修改

**存在的问题:** 由于线程执行的过程是不可控的,所以需要采用同步机制来协同对对象可变状态的访问

#### 解决问题的根本方法

**同一时刻有且只有一个线程在操作共享数据,其他线程必须等到该线程处理完数据后再对共享数据进行操作**

#### 互斥锁的特性

​	能达到互斥访问的目的. 当一个共享数据被当前正在访问的线程, 加上互斥锁之后. 在同一个时刻, 其他线程只能处于等待的状态. 知道当前线程处理完, 释放该锁后. 才有可能获得操作该共享数据的权利. 

* **互斥性:** 即在同一时间只允许一个线程持有某个对象锁,通过这种特性来实现多线程的协调机制,这样在同一时间只有一个线程对需要同步的代码块(复合操作)进行访问。互斥性也称为操作的原子性。

* **可见性:**必须确保在锁被释放之前,对共享变量所做的修改,对于随后获得该锁的另一个线程是可见的(即在获得锁时应获得最新共享变量的值) ,否则另一个线程可能是在本地缓存的某个副本上继续操作,从而引起不一致。

### synchronized使用注意

​	对于Java来讲, 关键字synchronized 满足了上述的要求, 在线程同步中. 就扮演了非常重要的作用. 他可以保证, 在同一个时刻, 只有一个线程可以执行某个方法或者代码块, 主要是对方法或者代码块中存在对共享数据的操作. 同时, synchronized也可以保证一个线程的变化, 主要是共享数据的变化. 保证这个变化被其他线程可看到. 也就是说, 保证共享数据的可见性. 

**synchronized 锁的不是代码, 锁的都是对象.** 

前面学习 JVM的时候了解到, 堆是线程间共享的. 和码农打交道最多的区域. 因此恰当的合理的给一个对象上锁，是解决安全问题的关键. 

#### 根据获取的锁的分类: 获取对象锁和类锁

* 获取对象锁的两种用法

  1. 同步代码块 

     > ` synchronized(this), synchronized (类实例对象)`锁是小括号()中的实例对象。

  2. 同步非静态方法

     > `synchronized method` , 锁是当前对象的实例对象。

* 获取类锁的两种用法

  1. 同步代码块 

     > ` synchronized (类.class)` ,锁是小括号 ()中的类对象(Class对象)

  2. 同步静态方法 

     > `synchronized static method` , 锁是当前对象的类对象(Class对象)

#### 对象锁和类锁的总结

1. 有线程访问对象(共享资源的对象)的同步代码块时 ,另外的线程可以访问该对象的非同步代码块;
2. 若锁住的是同一个对象,一个线程在访问对象的同步代码块时,另一个访问对象的同步代码块的线程会被阻塞;
3. 若锁住的是同一个对象,一个线程在访问对象的同步方法时,另一个访问对象同步方法的线程会被阻塞;
4. 若锁住的是同一个对象,一个线程在访问对象的同步代码块时,另一个访问对象同步方法的线程会被阻塞,反之亦然;
5. 同一个类的不同对象的对象锁互不干扰 ;
6. 类锁由于也是一 种特殊的对象锁,因此表现和上述1 , 2,3,4-致,而由于一个类只有一把对象锁,所以同一个类的不同对象使用类锁将会是同步的;
7. 类锁和对象锁互不干扰

后面的线程八锁有验证

### 使用synchronized

#### 1. 使用 synchronized

卖票问题, 票是共享资源

###### 代码

```java
public class Pro01 {
    public static void main(String[] args) {
        Ticket ticket = new Ticket();

        // 模拟十个卖票窗口
        for (int i = 0; i < 10; i++) {
            String name = i + "号窗口";
            new Thread(ticket, name).start();
        }
    }
}

class Ticket implements Runnable {
    // 共享资源, 一共 100张票
    private int tick = 100;

    @Override
    public void run() {
        while (true) {
            if (tick > 0) {
                try {
                    Thread.sleep(100);
                    System.out.println(Thread.currentThread().getName() + ": 卖出第" + tick-- + "张票");
                } catch (Exception e) {
                }
            }
        }
    }
}
```

###### 结果

会出现卖重票或者负票的情况

```java
...
9号窗口: 卖出第0张票
8号窗口: 卖出第-1张票
1号窗口: 卖出第-2张票
5号窗口: 卖出第-3张票
...
```

###### 原因

卖票这个操作不是同步的, 会出现并发问题

**a. 卖重票** 

1. `tick--`这个操作不是原子性的, 分三步, 1. 先获取, 2. 在修改, 3. 再写回. 
2. 假设当线程1获取`tick = 5`之后, 被挂起, 
3. 线程 2执行到 `tick--`这个操作, 获取到相同的值 `tick = 5`, 成功进行操作后修改为 4, 进行输出
4. 之后线程 1在继续执行, 把 4再次写回,在输出到控制台
5.  **相当于两次操作但只是 减了 1**, 发生卖的票是重票

**b. 卖0票或者负票** 

1. 线程 1执行, 此时 tick = 1, 执行完 `if(tick > 0)` 之后被挂起, 
2. 之后线程 2执行, `tick = 1`, 执行完之后, `tick = 0`. 
3. 切换后线程 1继续执行, 此时 `tick = 0` , 再次执行输出语句, 出现卖第 0张票. 
4. 卖负票是多个线程执行到 `if(tick > 0)`之后被挂起, 然后又执行. 出现多张负票情况, 

###### 使用synchronized

在需要同步(只能单个线程操作)的地方加上代码块 `synchronized(this)`

也就是synchronized中的操作变成类原子操作. 

```java
class Ticket implements Runnable {
    // 一共 100张票
    private int tick = 100;

    @Override
    public void run() {
        while (true) {
            synchronized (this) {
                if (tick > 0) {
                    try {
                        Thread.sleep(100);
                        System.out.println(Thread.currentThread().getName() + ": 卖出第" + tick-- + "张票");
                    } catch (Exception e) {
                    }
                }
            }
        }
    }
}
```

###### 结果

不会出现上述问题

```
...
6号窗口: 卖出第3张票
7号窗口: 卖出第2张票
7号窗口: 卖出第1张票
```

### 线程八锁

工作中非常常见的八种情况, 对象锁和类锁的使用情况, 判断打印的是 “one” or “two”

#### 情况一

两个普通的同步方法, 两个线程, 标准打印

```java
public class TestThreadMonitor {
    public static void main(String[] args) {
        Number number = new Number();

        new Thread(() -> {
            number.getOne();
        }).start();

        new Thread(() -> {
            number.getTwo();
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
```

###### 结果

```java
// 立即打印
one 
two
```

#### 情况二

新增 `Thread.sleep()` 给 `getOne()`

```java
public class TestThreadMonitor {

    public static void main(String[] args) {
        Number number = new Number();

        new Thread(() -> {
            number.getOne();
        }).start();

        new Thread(() -> {
            number.getTwo();
        }).start();
    }
}

class Number {
    public synchronized void getOne() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
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
```

###### 结果

```java
// 两秒后打印
one 
two
```

解释: 使用的都是同一把锁, 锁是当前对象的实例对象

#### 情况三

新增一个普通方法 `getThree()`

```java
public class TestThreadMonitor {

    public static void main(String[] args) {
        Number number = new Number();

        new Thread(() -> {
            number.getOne();
        }).start();

        new Thread(() -> {
            number.getTwo();
        }).start();

        new Thread(() -> {
            number.getThree();
        }).start();
    }
}

class Number {
    public synchronized void getOne() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {

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
```

###### 结果

```java
// 立即打印
three
// 两秒后打印
one
two
```

因为没有加锁, 互不影响

#### 情况四

两个普通同步方法, 两个 Number对象

```java
public class TestThreadMonitor {
    public static void main(String[] args) {
        Number number1 = new Number();
        Number number2 = new Number();

        new Thread(() -> {
            number1.getOne();
        }).start();

        new Thread(() -> {
            number2.getTwo();
        }).start();
    }
}

class Number {
    public synchronized void getOne() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {

        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}
```

###### 结果

```java
// 立即打印
two
// 两秒后
one
```

使用的并不是同一个资源, 所以两个 this锁是两个对象的. 是两把锁

#### 情况五

修改 `getOne()` 为静态同步方法

```java
public class TestThreadMonitor {
    public static void main(String[] args) {
        Number number = new Number();

        new Thread(() -> {
            number.getOne();
        }).start();

        new Thread(() -> {
            number.getTwo();
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {

        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}
```

###### 结果

```java
two
// 两秒后
one
```

一个使用对象锁, 一个使用类锁, 不是同一把锁

#### 情况六

修改两个方法均为静态同步方法, 一个 Number对象

```java 
public class TestThreadMonitor {

    public static void main(String[] args) {
        Number number = new Number();

        new Thread(() -> {
            number.getOne();
        }).start();

        new Thread(() -> {
            number.getTwo();
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
        }
        System.out.println("one");
    }

    public static synchronized void getTwo() {
        System.out.println("two");
    }
}
```

###### 结果

```java
// 两秒后
one
two
```

都使用的是类锁, 同一把锁

#### 情况七

1. `getOne()`是静态同步方法, `Number1` 调用
2. `getTwo()`是非静态同步方法, `Number2` 调用

```java
public class TestThreadMonitor {

    public static void main(String[] args) {
        Number number1 = new Number();
        Number number2 = new Number();

        new Thread(() -> {
            number1.getOne();
        }).start();

        new Thread(() -> {
            number2.getTwo();
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
        }
        System.out.println("one");
    }

    public synchronized void getTwo() {
        System.out.println("two");
    }
}
```

###### 结果

```java
// 立即
two
// 两秒后
one
```

一个使用对象锁, 一个使用类锁, 互不影响

#### 情况八

两个静态同步方法, 两个 Number对象

```java
public class TestThreadMonitor {

    public static void main(String[] args) {
        Number number1 = new Number();
        Number number2 = new Number();

        new Thread(() -> {
            number1.getOne();
        }).start();

        new Thread(() -> {
            number2.getTwo();
        }).start();
    }
}

class Number {
    public static synchronized void getOne() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
        }
        System.out.println("one");
    }

    public static synchronized void getTwo() {
        System.out.println("two");
    }
}
```

###### 结果

```java
// 两秒后
one
two
```

都是用的是类锁. 同一把锁

#### 线程八锁的关键

- **非静态方法的锁:** this
- **静态方法的锁:** 对应的 Class对象
- 某一时刻内, 只能有一个线程持有锁(同一把锁), 无论几个方法
- 如果锁不同,则没有竞争关系



##  线程之间的通信

**概念:** 多个线程在处理**同一个资源**, 但是处理的动作(线程的任务)却不相同

> 通信也称为线程之间的合作

#### 为什么要处理线程间的通信?

当我们需要多个线程来共同完成一件任务, 并且我们我们他们有规律的执行,那么多线程之间需要一些协调通信, 以此来帮我们达到多线程共同操作一份数据

#### 如何保证线程间的通信有效利用资源

多个线程在处理同一个资源，并且任务不同时, **需要线程通信**来帮助解决线程之间对同一个变量的使用和操作. 

==就是多个线程在操作同一份数据时，避免对同一共享变量的争夺，==也就是我们需要一定的手段时，各个线程能有效的利用资源，而这种手段即 — 等待唤醒机制.

###  等待唤醒机制

#### 1. 什么是等待唤醒机制？

多个线程间的一种协作机制

> 谈到线程，我们经常想到的是线程间的竞争，比如去争夺所，但这并不是故事的全部线程间也会有协作机制，就好比在公司里你和你的同事们，你们可能存在着竞争式的竞争，但更多时候你们更是一起合作，以完成某些任务

####  2. 专业性理解

在一个线程执行的规定操作后就进入**等待状态(wait())**，等待其他线程执行完他们指定的代码后**再将其唤醒(notify())**, 在还有多个线程进行等待时，如果需要可以使用 `notifyAll()` 来唤醒所有的等待线程

> wait/notify 就是线程见的一种协作机制

### 生产者消费者问题

##### 问题代码

两个线程, 店员A线程, 顾客A线程

```java
public class Pro02 {

    public static void main(String[] args) {
        AppleStore appleStore = new AppleStore();

        Productor productor = new Productor(appleStore);
        Consumer consumer = new Consumer(appleStore);

        new Thread(productor, "店员A").start();
        new Thread(consumer, "顾客A").start();
    }
}

// 资源类(苹果店)
class AppleStore {
    private int num = 0;    // 苹果数量

    // 进苹果, 苹果数量 = 5时, 就不能继续进了
    public synchronized void getApple() {
        if (num >= 5)
            System.out.println("又要进苹果, 仓库已满, 不能再进苹果了!");
        else
            System.out.println(Thread.currentThread().getName() + "\t进入第 " + ++num + "个苹果");
    }

    // 卖苹果, 苹果数量 = 0时, 就不能再卖了
    public synchronized void seleApple() {
        if (num <= 0)
            System.out.println("又要买苹果, 仓库缺货, 不能再卖苹果了!");
        else
            System.out.println(Thread.currentThread().getName() + "\t买到出第 " + num-- + "个苹果");
    }
}

// 生产者(店员)
class Productor implements Runnable {
    private AppleStore appleStore;

    public Productor(AppleStore appleStore) {
        this.appleStore = appleStore;
    }

    @Override
    public void run() {
        // 进货 20次苹果
        for (int i = 0; i < 20; i++)
            appleStore.getApple();
    }
}

// 消费者(顾客)
class Consumer implements Runnable {
    private AppleStore appleStore;

    public Consumer(AppleStore appleStore) {
        this.appleStore = appleStore;
    }

    @Override
    public void run() {
        // 购买 20次苹果
        for (int i = 0; i < 20; i++)
            appleStore.seleApple();
    }
}
```

###### 结果

```
...
店员A	进入第 4个苹果
店员A	进入第 5个苹果
又要进苹果, 仓库已满, 不能再进苹果了!
又要进苹果, 仓库已满, 不能再进苹果了!
又要进苹果, 仓库已满, 不能再进苹果了!
...
```

###### 问题

苹果仓库已经满了, 但是又继续进了很多次苹果

###### 原因

店员线程一直执行

###### 解决

加入等待唤醒机制

##### 修改代码一

修改资源类, 加入 wait() 和 notifyAll()

为了方便演示, 修改为最多进 1个苹果, 购买和进入 2次

```java
class AppleStore {

    private int num = 0;    // 苹果数量

    // 进苹果, 苹果数量 = 1时, 就不能继续进了
    public synchronized void getApple() {
        if (num >= 1) {
            try {
                System.out.println("又要进苹果, 仓库已满, 不能再进苹果了!");
                this.wait();
            } catch (InterruptedException e) {
            }
        } else {
            System.out.println(Thread.currentThread().getName() + "\t进入第 " + ++num + "个苹果");
            this.notifyAll();
        }
    }

    // 卖苹果, 苹果数量 = 0时, 就不能再卖了
    public synchronized void seleApple() {
        if (num <= 0) {
            try {
                System.out.println("又要买苹果, 仓库缺货, 不能再卖苹果了!");
                this.wait();
            } catch (Exception e) {
            }
        } else {
            System.out.println(Thread.currentThread().getName() + "\t买到出第 " + num-- + "个苹果");
            this.notifyAll();
        }
    }
}
```

###### 结果

```java
店员A	进入第 1个苹果
又要进苹果, 仓库已满, 不能再进苹果了!	// 店家A线程进入 wait()
顾客A	买到出第 1个苹果				 // 顾客A线程唤醒店家A
又要买苹果, 仓库缺货, 不能再卖苹果了!	// 顾客A线程进入wait()
```

###### 问题

结果没有问题, 但是程序有时候会结束, 有时候不会结束

###### 原因

1. 店员 A线程第一轮进入 1个苹果 此时 num = 1, 第二轮想要继续进, 不可以, 进入 wait(), 此时 num = 1

2. 顾客 A线程执行, 第一轮买到 1个苹果. 唤醒店员 A线程, 此时 num = 0, 线程 A继续执行结束, 线程A运行结束; 第二轮 num = 0, 进入到wait(), 无人唤醒. 程序不会终止

###### 解决

问题出现在剩下最后一个没有唤醒, 所以**必须执行 notifyAll()** , 

##### 修改代码二

将 else去掉 (将 notifyAll() 从 else中拿出来 sout还在 else里面, 也可以解决问题, 但是没有办法说明问题了, 这里面没有弄太明白, 应该是睡醒了继续干活吧, 不能够醒了之后就不干活了)

将 notifyAll() 从 else中拿出来

```java
// 资源类(苹果店)
class AppleStore {

    private int num = 0;    // 苹果数量

    // 进苹果, 苹果数量 = 1时, 就不能继续进了
    public synchronized void getApple() {
        if (num >= 1) {
            try {
                System.out.println("又要进苹果, 仓库已满, 不能再进苹果了!");
                this.wait();
            } catch (InterruptedException e) {
            }
        }
        System.out.println(Thread.currentThread().getName() + "\t进入第 " + ++num + "个苹果");

        this.notifyAll();
    }

    // 卖苹果, 苹果数量 < 0时, 就不能再卖了
    public synchronized void seleApple() {
        if (num <= 0) {
            try {
                System.out.println("又要买苹果, 仓库缺货, 不能再卖苹果了!");
                this.wait();
            } catch (Exception e) {
            }
        }
        System.out.println(Thread.currentThread().getName() + "\t买到出第 " + num-- + "个苹果");

        this.notifyAll();
    }
}
```

###### 结果

```java
店家A	进入第 1个苹果
又要进苹果, 仓库已满, 不能再进苹果了!	// 店员A进入 wait()
店家A	买到出第 1个苹果				 // 顾客A执行 notifyAll(), 将店家A唤醒
又要买苹果, 仓库缺货, 不能再卖苹果了!	// 顾客A线程进入 wait()
店家A	进入第 1个苹果			     // 店员A继续执行, 执行 notifyAll(), 执行结束
店家A	买到出第 1个苹果				 // 顾客A继续执行, 执行结束
```

###### 问题

当加入多个线程后

修改为每个线程操作 20次资源类

```java
public static void main(String[] args) {
    AppleStore appleStore = new AppleStore();

    Productor productor = new Productor(appleStore);
    Consumer consumer = new Consumer(appleStore);

    new Thread(productor, "店员A").start();
    new Thread(consumer, "顾客A").start();
    new Thread(productor, "店员B").start();
    new Thread(consumer, "顾客B").start();
    new Thread(productor, "店员C").start();
    new Thread(consumer, "顾客C").start();
    new Thread(productor, "店员D").start();
    new Thread(consumer, "顾客D").start();
}
```

###### 结果

```
店员A	进入第 1个苹果
又要进苹果, 仓库已满, 不能再进苹果了!
店员B	进入第 2个苹果
又要进苹果, 仓库已满, 不能再进苹果了!
店员A	进入第 3个苹果
又要进苹果, 仓库已满, 不能再进苹果了!
...
```

最多进入1个苹果, 结果却进了多个

###### 原因

1. 多个店员线程轮流执行, 即店员 A执行后, 进入 wait() , 此时 num = 1, 
2. 然后店员 B线程继续执行, 进入 wait(), 
3. 然后店员C线程继续执行, 进入 wait(), 
4. 随后店员 A醒了, 进入一个苹果, 店员 B醒了, 进入一个苹果, 店员 C醒了, 进入一个苹果…

>  这种现象叫出现虚假唤醒机制, 多个消费者或生产者连续执行.

###### 解决

使用 while, 即使不该醒的线程醒了, 再做一次判断, 再让他进入wait()

##### 最终代码

```java
...
顾客D	买到出第 1个苹果
又要买苹果, 仓库缺货, 不能再卖苹果了!
又要买苹果, 仓库缺货, 不能再卖苹果了!
店员C	进入第 1个苹果
顾客C	买到出第 1个苹果
...
```

终于没有问题了

###### 疑问

这样看上去不是和没有唤醒机制的代码一样吗? 

其实是不一样的, 没有唤醒机制的代码, 生产者和消费者之间没有通信, 两种线程不存在关联, 而现在是两者的线程有通信, 一方不予执行, 另一方在继续执行

### 使用等待唤醒机制需要注意

* notifyAll() 和 wait() 必须配对使用, 不能够有一方不执行的现象, 否则会出现有一方的线程无法终止

  > notifyAll() 必须执行, 而且要使用 notifyAll() 不使用notify()

* 为了避免虚假唤醒机制, 要使用while不使用if

  > 官方文档中有说明, wait() 配合 while使用, 永远不要使用 if
  >

## synchronized底层实现原理

实在有点难, 总结不了, 下面文章写的很好

​	[☆啃碎并发（七）：深入分析Synchronized原理](https://www.jianshu.com/p/e62fa839aa41)



#### synchronized底层对应的 JMM模型八大原子操作

![synchronized底层](E:/Git/%E7%A7%8B%E6%8B%9B_%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/Java/JUC/%E5%9B%BE%E7%89%87/synchronized%E5%BA%95%E5%B1%82.png)

一旦有线程想要进入到同步块, 需要先对共享变量上锁. 

汇编指令: cmpchg

1. 线程一拿到同步资源的锁, 将共享变量的对象锁住(lock操作)
2. 此时, 线程二, 线程三对于 object对象不可读, 不可写, 保证了任意时间内只有一个线程能够拿到线程锁. 线程二和线程三只能被阻塞. 放在一个区域中
3. 线程一对于代码块中的操作进行操作
4. 操作完之后, 将对象锁释放掉, unlock操作
5. 线程二, 线程三进行竞争.

#### 各种锁

* 偏向锁适用于一个线程反复的进入同一个同步块场景

* 轻量级锁适用于线程交替执行场景

* 重量级锁性能非常低, 适用于高并发的场景

原子操作描述程序在运行过程中数据交互的情况



#### 代码补充

##### 锁消除

###### 代码一

```java
public class StringBufferWithoutSync {
    public void add(String str1, String str2) {
        // StringBuffer是线程安全,由于sb只会在append方法中使用,不可能被其他线程引用
        // 因此sb属于不可能共享的资源,JVM会自动消除内部的锁
        StringBuffer sb = new StringBuffer();
        sb.append(str1).append(str2);
    }

    public static void main(String[] args) {
        StringBufferWithoutSync withoutSync = new StringBufferWithoutSync();
        for (int i = 0; i < 1000; i++) {
            withoutSync.add("aaa", "bbb");
        }
    }
}
```

##### 锁粗话

###### 代码二

```java
public class CoarseSync {
    public static String copyString100Times(String target){
        int i = 0;
        StringBuffer sb = new StringBuffer();
        while (i < 100){
            sb.append(target);
        }
        return sb.toString();
    }
}
```





