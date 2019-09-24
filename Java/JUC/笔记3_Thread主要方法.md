## sleep和 wait的区别

#### 基本的差别

* sleep是Thread类的方法, wait是Object类中定义的方法
* sleep()方法可以在任何地方使用. 
* wait()方法只能在 synchronized方法或 synchronized块中使用

#### 最主要的本质区别

* Thread.sleep只会让出CPU ,不会导致锁行为的改变

  >即当前线程是拥有锁的, 那么 sleep() 不会让线程释放锁. 而是主动会让出 CPU, 让出CPU之后, CPU就可以去执行其他任务了. 因此 sleep() 不会影响锁的相关行为

* Object.wait不仅让出CPU ,还会释放已经占有的同步资源锁

  > 以便让其他等待该资源的线程得到该资源.wait()方法只能在synchronized方法或synchronized块中使用, 是因为已经获取锁了, 才能够释放锁. 

###### 代码一

验证wait() 会让出锁

```java
public class WaitSleepDemo {
    public static void main(String[] args) {
        final Object lock = new Object();
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread A is waiting to get lock");
                synchronized (lock){
                    try {
                        System.out.println("thread A get lock");
                        Thread.sleep(20);
                        System.out.println("thread A do wait method");
                        lock.wait(2000);
                        System.out.println("thread A is done");
                    } catch (InterruptedException e){
                        e.printStackTrace();
                    }
                }
            }
        }).start();
        try{
            Thread.sleep(10);
        } catch (InterruptedException e){
            e.printStackTrace();
        }
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread B is waiting to get lock");
                synchronized (lock){
                    try {
                        System.out.println("thread B get lock");
                        System.out.println("thread B is sleeping 10 ms");
                        Thread.sleep(10);
                        System.out.println("thread B is done");
                    } catch (InterruptedException e){
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```

###### 结果

```java
thread A is waiting to get lock
thread A get lock
thread B is waiting to get lock
thread A do wait method
thread B get lock
thread B is sleeping 10 ms
thread B is done
// 2s后
thread A is done
```

###### 代码二

验证sleep() 不会让出锁

```java
public class WaitSleepDemo {
    public static void main(String[] args) {
        final Object lock = new Object();
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread A is waiting to get lock");
                synchronized (lock) {
                    try {
                        System.out.println("thread A get lock");
                        Thread.sleep(20);
                        System.out.println("thread A do sleep method");
//                        lock.wait(2000);
                        Thread.sleep(2000);
                        System.out.println("thread A is done");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("thread B is waiting to get lock");
                synchronized (lock) {
                    try {
                        System.out.println("thread B get lock");
                        System.out.println("thread B is wait 10 ms");
                        lock.wait(10);
                        System.out.println("thread B is done");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```

###### 结果

```
thread A is waiting to get lock
thread A get lock
thread B is waiting to get lock
thread A do sleep method
// 2s后
thread A is done
thread B get lock
thread B is wait 10 ms
thread B is done
```



## notify和 notifyAll的区别

作用是唤醒 wait()的线程

### 两个概念

* 锁池EntryList

* 等待池 WaitSet

对于运行程序的每个对象来说. 都有两个池: 锁池EntryList, 等待池 WaitSet. 而这两个池, 有与 Object基类的wait, notify, notifyAll 三个方法以及synchronnized相关. 

#### 锁池

​	假设线程 A已经拥有了某个对象(不是类)的锁，而其它线程B、C想要调用这个对象的某个 synchronized方法( 或者块)，由于B、C线程在进入对象的synchronized方法(或者块)之前必须先获得该对象锁的拥有权，而恰巧该对象的锁目前正被线程 A所占用，此时B、C线程就会被阻塞，进入一个地方去等待锁的释放，这个地方便是该对象的锁池

> 就是被阻塞, 无法获取 synchronized的线程去的地方

#### 等待池

​	假设线程 A调用了某个对象的 wait()方法，线程 A就会释放该对象的锁，同时线程 A就进入到了该对象的等待池中，进入到等待池中的线程不会去竞争该对象的锁。

> 就是调用wait()方法的线程去的地方

​	在真实的开发中, 会有很多的线程去竞争这个锁. 此时优先级高的线程竞争到锁的概率会更大. 假若某线程没有竞争到该锁. 他就会留到锁池中, 并不会重新进入到等待池中.而竞争到锁的线程, 就继续往下执行, 直到执行完 synchronized或者遇到异常才会释放该锁. 此时锁池中的线程会继续竞争该锁. 锁池和等待池都是针对对象而言的.

一个锁可以放在多个代码块上. 

### 区别

* notifyAll会让所有处于等待池的线程**全部进入锁池**去竞争获取锁的机会

  > 没有获取到锁而已经呆在锁池中的线程只能等待其他机会去获取锁. 而不能再主动回到等待池当中. 

* notify只会**随机选取一个**处于等待池中的线程进入锁池去竞争获取锁的机会。

###### 代码

```java
public class NotificationDemo {
    private volatile boolean go = false;

    public static void main(String args[]) throws InterruptedException {
        // 锁
        final NotificationDemo test = new NotificationDemo();

        // 睡眠的任务
        Runnable waitTask = new Runnable() {
            @Override
            public void run() {
                try {
                    test.shouldGo();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " finished Execution");
            }
        };

        // 唤醒的任务
        Runnable notifyTask = new Runnable() {
            @Override
            public void run() {
                test.go();
                System.out.println(Thread.currentThread().getName() + " finished Execution");
            }
        };

        Thread t1 = new Thread(waitTask, "WT1"); //will wait
        Thread t2 = new Thread(waitTask, "WT2"); //will wait
        Thread t3 = new Thread(waitTask, "WT3"); //will wait
        Thread t4 = new Thread(notifyTask, "NT1"); //will notify

        // starting all waiting thread
        t1.start();
        t2.start();
        t3.start();
        // pause to ensure all waiting thread started successfully
        Thread.sleep(200);

        // starting notifying thread
        t4.start();
    }

    /*
     * wait and notify can only be called from synchronized method or bock
     */
    private synchronized void shouldGo() throws InterruptedException {
        while (go != true) {
            System.out.println(Thread.currentThread() + " is going to wait on this object");
            wait(); // release lock and reacquires on wakeup
            System.out.println(Thread.currentThread() + " is woken up");
        }
        go = false; // resetting condition
    }

    /*
     * both shouldGo() and go() are locked on current object referenced by "this" keyword
     */
    private synchronized void go() {
        while (go == false) {
            System.out.println(Thread.currentThread() + " is going to notify all or one thread waiting on this object");

            go = true; // making condition true for waiting thread
            // notify(); // only one out of three waiting thread WT1, WT2,WT3 will woke up
            notifyAll(); // all waiting thread  WT1, WT2,WT3 will woke up
        }
    }
}
```

###### 结果一

使用 notify()

```
Thread[WT1,5,main] is going to wait on this object
Thread[WT3,5,main] is going to wait on this object
Thread[WT2,5,main] is going to wait on this object
Thread[NT1,5,main] is going to notify all or one thread waiting on this object
Thread[WT1,5,main] is woken up
WT1 finished Execution
NT1 finished Execution
```

###### 结果二

使用notifyAll()

```
Thread[WT1,5,main] is going to wait on this object
Thread[WT2,5,main] is going to wait on this object
Thread[WT3,5,main] is going to wait on this object
Thread[NT1,5,main] is going to notify all or one thread waiting on this object
Thread[WT3,5,main] is woken up
WT3 finished Execution
Thread[WT2,5,main] is woken up
Thread[WT2,5,main] is going to wait on this object
Thread[WT1,5,main] is woken up
Thread[WT1,5,main] is going to wait on this object
NT1 finished Execution
```

## yield

#### 概念

​	当调用Thread.yield()函数时,会给线程调度器一个当前线程愿意让出CPU使用的暗示,但是线程调度器可能会忽略这个暗示。

> **通俗:** 当前线程愿意让出CPU的使用权. 可以让其他线程获取到这个CPU的使用权去执行, 但是这取决于调度器, 调度器可能会停止当前线程的执行. 也可能会让该线程继续去执行.

对锁的行为不会有影响. 



## 如何中断线程

#### 已经被抛弃的方法

* 通过调用stop(方法停止线程
* 通过调用suspend()和resume()方法

​	让一个线程调用stop() 去结束另一个线程, 这种方式太过暴力且不安全的. 比如线程 A调用线程 B的stop(). 调用的时候线程 A并不知道线程 B的执行情况. 这种突然的停止会导致线程 B的一些清理工作无法完成，还有就是执行 stop方法后线程 B会马上释放锁. 这可能会引发数据不同步的问题. 

#### 目前使用的方法

* 调用 interrupt() ,通知线程应该中断了

  1. 如果线程处于被阻塞状态,那么线程将立即退出被阻塞状态,并抛出一个InterruptedException异常。
  2. 如果线程处于正常活动状态,那么会将该线程的中断标志设置为true。被设置中断标志的线程将继续正常运行,不受影响。

  > 因此并不能真正的中断线程, 需要被调用的线程自己配合才行. 

* 需要被调用的线程配合中断

  1. 在正常运行任务时,经常检查本线程的中断标志位,如果被设置了中断标志就自行停止线程。
  2. 如果线程处于正常活动状态,那么会将该线程的中断标志设置为true。被设置中断标志的线程将继续正常运行,不受影响。

###### 代码

```java
public class InterruptDemo {
    public static void main(String[] args) throws InterruptedException {
        Runnable interruptTask = new Runnable() {
            @Override
            public void run() {
                int i = 0;
                try {
                    //在正常运行任务时，经常检查本线程的中断标志位，如果被设置了中断标志就自行停止线程
                    while (!Thread.currentThread().isInterrupted()) {
                        Thread.sleep(100); // 休眠100ms
                        i++;
                        System.out.println(Thread.currentThread().getName() + " (" + Thread.currentThread().getState() + ") loop " + i);
                    }
                } catch (InterruptedException e) {
                    //在调用阻塞方法时正确处理InterruptedException异常。（例如，catch异常后就结束线程。）
                    System.out.println(Thread.currentThread().getName() + " (" + Thread.currentThread().getState() + ") catch InterruptedException.");
                }
            }
        };
        Thread t1 = new Thread(interruptTask, "t1");
        System.out.println(t1.getName() +" ("+t1.getState()+") is new.");

        t1.start();                      // 启动“线程t1”
        System.out.println(t1.getName() +" ("+t1.getState()+") is started.");

        // 主线程休眠 300ms，然后主线程给t1发“中断”指令。
        Thread.sleep(300);
        t1.interrupt();
        System.out.println(t1.getName() +" ("+t1.getState()+") is interrupted.");

        // 主线程休眠 300ms，然后查看 t1的状态。
        Thread.sleep(300);
        System.out.println(t1.getName() +" ("+t1.getState()+") is interrupted now.");
    }
}
```

###### 结果

```
t1 (NEW) is new.
t1 (RUNNABLE) is started.
t1 (RUNNABLE) loop 1
t1 (RUNNABLE) loop 2
t1 (TIMED_WAITING) is interrupted.
t1 (RUNNABLE) catch InterruptedException.
t1 (TERMINATED) is interrupted now.
```

