# Lock

## ReentrantLock

​	ReentrantLock重入锁，是实现Lock接口的一个类，也是在实际编程中使用频率很高的一个锁，**支持重入性，表示能够对共享资源能够重复加锁，即当前线程获取该锁再次获取不会被阻塞**。

ReentrantLock还支持**公平锁和非公平锁**两种方式

### 使用

#### 1. 基本使用

1. 代替 synchronized, 在需要锁定的代码**前后使用** `lock.lock()和 lock.unlock()`

2. 必须要手动释放锁**（重要)**

   > 使用syn锁定的话如果遇到异常，jvm会自动释放锁，但是lock必须手动释放锁，因此**经常在finally中进行锁的释放**

###### 代码

```java
public class ReentrantLock2 {
    public static void main(String[] args) throws InterruptedException {
        Lock lock = new ReentrantLock();
        Apple apple = new Apple(lock);
        new Thread(() -> {
            apple.setName(Thread.currentThread().getName());
            apple.eat();
        }, "小红").start();

        Thread.sleep(300);

        new Thread(() -> {
            apple.setName(Thread.currentThread().getName());
            apple.eat();
        }, "小明").start();
    }
}

// 资源类
class Apple {
    String name;
    Lock lock;

    public Apple(Lock lock) {
        this.lock = lock;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void eat() {
        try {
            lock.lock();
            System.out.println(name + "正在吃苹果...");
            Thread.sleep(1500);
            System.out.println(name + "苹果吃完了...");
        } catch (InterruptedException e) {
        } finally {
            lock.unlock();
        }
    }
}
```

###### 结果

```
小红正在吃苹果...
小明苹果吃完了...
小明正在吃苹果...
小明苹果吃完了...
```

### 2. Condition 控制线程通信

​	在 Condition对象中, 与 `wait, notify 和notifyAll`方法对应的分别是 `await, signal和singalAll`方式

Condition 实例是指上被绑定到一个锁上, 要为特定 Lock实例获得获得 Condition实例, 使用 `newCondition()` 方法

```java
public class TestProductAndConsumerForLock {

    public static void main(String[] args) {
        Clerk clerk = new Clerk();
        Productor productor = new Productor(clerk);
        Consumer consumer = new Consumer(clerk);

        new Thread(productor, "生产者1").start();
        new Thread(consumer, "消费者2").start();
        new Thread(productor, "生产者3").start();
        new Thread(consumer, "消费者4").start();
        new Thread(productor, "生产者5").start();
        new Thread(consumer, "消费者6").start();
    }
}

// 店员
class Clerk1 {
    private int productNum = 0;

    // 重点
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    // 进货
    public  void getPro() {
        lock.lock();
        try {
            while (productNum >= 1) {
                System.out.println("仓库已满, 不能再进货了!");
                try {
                    Thread.sleep(50);
                    condition.await();
                } catch (InterruptedException e) {
                }
            }
            System.out.println(Thread.currentThread().getName() + " 进入第 " + ++productNum + "件货");
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    // 卖货
    public synchronized void selePro() {
        lock.lock();
        try {
            while (productNum <= 0) { // 为了避免虚假唤醒问题, wait()应该总是使用在循环中
                System.out.println("仓库缺货, 不能再卖商品了!");
                try {
                    Thread.sleep(50);
                    condition.await();
                } catch (InterruptedException e) {
                }
            }
            System.out.println(Thread.currentThread().getName() + "卖出第 " + --productNum + "件货");
            condition.signalAll();
        }finally {
            lock.unlock();
        }

    }
}

// 生产者
class Productor1 implements Runnable {
    private Clerk clerk;

    public Productor1(Clerk clerk) {
        this.clerk = clerk;
    }

    @Override
    public void run() {
        // 模拟 20次进货
        for (int i = 0; i < 20; i++) {
            clerk.getPro();
        }
    }
}

// 消费者
class Consumer1 implements Runnable {
    private Clerk clerk;

    public Consumer1(Clerk clerk) {
        this.clerk = clerk;
    }

    @Override
    public void run() {
        // 模拟 20次消费
        for (int i = 0; i < 20; i++) {
            clerk.selePro();
        }
    }
}
```

### 3. 进行“尝试锁定” tryLock

这样无法锁定，或者在指定时间内无法锁定，线程可以决定是否继续等待

- 不管锁定与否，方法都将继续执行. 可以根据tryLock的返回值来判定是否锁定
- 也可以指定tryLock的时间，由于tryLock(time)抛出异常，所以要注意 unclock的处理，必须放到finally中

```java
public class ReentrantLock2 {
    Lock lock = new ReentrantLock();

    void m1() {
        try {
            lock.lock();
            for (int i = 0; i < 10; i++) {
                System.out.println(i);
            }
        }finally {
            lock.unlock();
        }
    }

    void m2() {
        boolean locked = false;
        try {
            // 返回的是 boolean 类型的值
            locked = lock.tryLock(5, TimeUnit.SECONDS);
            if (locked) {
                System.out.println("m2拿到锁啦, 继续执行");
            } else {
                System.out.println("m2拿不到锁啦, 做其他事情");
            }
        } finally {
            if(locked) lock.unlock();
        }
    }
}
```

### 4. 调用 lockInterruptibly方法

调用lockInterruptibly方法，可以对线程interrupt方法做出响应，

```java
public class ReentrantLock4 {

    public static void main(String[] args) {
        Lock lock = new ReentrantLock();

        Thread t1 = new Thread(()->{
            try {
                System.out.println("t1 start");
                TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
            } finally {
                lock.unlock();
            }
        });
        t1.start();

        Thread t2 = new Thread(()->{
            try {
                // lock.lock();
                lock.lockInterruptibly(); // 可以对interrupt()方法做出响应
            } finally {
                lock.unlock();
            }
        });
        t2.start();

        t2.interrupt(); // 打断线程2的等待
    }
}
```

### 5. 可以指定为公平锁

谁等的时间长, 下一个获得锁的就是他, 效率比较低

```java
private static Lock lock=new ReentrantLock(true);
```

### synchronized和 ReentrantLock的区别

#### 1. ReentrantLock (再入锁)

* 位于 `java.util.concurrent.locks`包
* 和CountDownLatch、Future Task、Semaphore 一样基于AQS实现
* 能够实现比 synchronized更细粒度的控制, 如控制fairness
* 调用之后, 必须调用 unlock() 释放锁
* 性能未必比 synchronized高, 并且也是可重入的

#### 2. ReentrantLock公平性的设置

* `ReentrantLock fairLock = new ReentrantLock(true);`

* 参数为true时,倾向于将锁赋予等待时间最久的线程

  > 公平性是减少线程饥饿的一种办法. 所谓饥饿, 就是值个别线程长期等待锁, 但却又无法获得锁的情况. 

* **公平锁:** 获取锁的顺序按先后调用lock方法的顺序(慎用)

* **非公平锁:** 抢占的顺序不一定,看运气

* synchronized是非公平锁

  > 通用场景中, 公平性未必有想象中那么重要. Java的默认的调度策略, 很少会导致饥饿情况的发生. 与此同时, 若要保证公平性, 只会引入额外的开销. 自然会导致一定的吞吐量下降. 

#### 3. ReentrantLock将锁对象化

* 判断是否有线程,或者某个特定线程,在排队等待获取锁
* 带超时的获取锁的尝试
* 感知有没有成功获取锁

#### 4. 能否将 wait\notify\notifyAll对象化

* 可以, 使用 `java.util.concurrent.locks.Condition`

正因为是类, 所以提供了更多, 更灵活的特性. 可以有继承, 可以有方法, 可以有各种各样的变量. 

#### 总结

* synchronized是关键字 , ReentrantLock是类

* ReentrantLock可以对获取锁的等待时间践行设置, 避免死锁

* ReentrantLock可以获取各种锁的信息

* RееntrаntLосk可以灵活的实现多路通知

* **机制: sync操作 Mark Word, lock调用 Unsafe类的 park() 方法**

  > Mark Word 是位于对象头中
  >
  > Unsafe类是类似于后门的工具. 可以用来在任意内存地址里面读写数据(对于普通用户, 使用起来比较危险)



## ReentrantReadWriteLock

### 预备知识

#### 独占锁与共享锁

- **独占锁:** 指该锁一次只能被一个线程所持有
- **共享锁:** 指该锁可被多个线程所持有

> 对 `ReentrantLock`和`Synchronized`而言都是独占锁
>
> 对`ReentrantReadWriteLock`其读锁是共享锁，其写锁是独占锁。

读锁的共享锁可保证并发读是非常高效的. 多个线程同时读一个资源类没有问题, 所以为了满足并发量, 读取共享资源应该可以同时进行, 但是如果有一个线程想要去写共享资源类, 就不应该再有其他线程可以对该资源进行读或写

* 读写，写读，写写的过程是互斥的

* 读读可以同时进行

### 测试对比

```java
public class ReadWriteLockDemo {

    public static void main(String[] args) {
        MyCache myCache = new MyCache();

        // 5个线程写
        for (int i = 0; i < 5; i++) {
            int temp = i;
            new Thread(() -> {
                myCache.put(temp + "", temp + "");
            }, String.valueOf(i)).start();
        }

        // 5个线程读
        for (int i = 0; i < 5; i++) {
            int temp = i;
            new Thread(() -> {
                myCache.get(temp + "");
            }, String.valueOf(i)).start();
        }
    }
}
```

#### 1. 不加锁

###### 代码

```java
class MyCache {

    // 缓存的东西要保证可见性， 很重要
    private volatile Map<String, Object> map = new HashMap<>();

    public void put(String key, Object value) {
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在写入: " + key);
            Thread.sleep(300);
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "\t 写入完成");
        } catch (InterruptedException e) {
        }
    }

    public void get(String key) {
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在读取");
            Thread.sleep(300);
            Object result = map.get(key);
            System.out.println(Thread.currentThread().getName() + "\t 读取完成: " + result);
        } catch (InterruptedException e) {
        }

    }
}
```

###### 结果

```
0	 正在写入: 0
1	 正在写入: 1
2	 正在写入: 2
3	 正在写入: 3
4	 正在写入: 4
0	 正在读取
1	 正在读取
2	 正在读取
3	 正在读取
4	 正在读取
0	 写入完成
1	 写入完成
2	 写入完成
4	 写入完成
3	 写入完成
0	 读取完成: 0
2	 读取完成: 2
1	 读取完成: 1
4	 读取完成: 4
3	 读取完成: 3
```

> 读和写都不是一个完整的统一体, 中间都被打断了, 写入操作是不可以被打断的

#### 2. 加 ReentrantLock

###### 代码

```java
class MyCache {

    // 缓存的东西要保证可见性， 很重要
    private volatile Map<String, Object> map = new HashMap<>();
    // 锁的粒度太粗了, 读写都一致, 并发性落后
    private Lock lock = new ReentrantLock();

    public void put(String key, Object value) {

        lock.lock();

        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在写入: " + key);
            Thread.sleep(300);
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "\t 写入完成");
        } catch (InterruptedException e) {
        } finally {
            lock.unlock();
        }
    }

    public void get(String key) {

        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在读取");
            Thread.sleep(300);
            Object result = map.get(key);
            System.out.println(Thread.currentThread().getName() + "\t 读取完成: " + result);
        } catch (InterruptedException e) {
        } finally {
            lock.unlock();

        }

    }
}
```

###### 结果

```
0	 正在写入: 0
0	 写入完成
1	 正在写入: 1
1	 写入完成
2	 正在写入: 2
2	 写入完成
3	 正在写入: 3
3	 写入完成
4	 正在写入: 4
4	 写入完成
0	 正在读取
0	 读取完成: 0
1	 正在读取
1	 读取完成: 1
2	 正在读取
2	 读取完成: 2
4	 正在读取
4	 读取完成: 4
3	 正在读取
3	 读取完成: 3
```

> 读和写都是一个完整的统一体, 中间都没有打断了, 锁的粒度粗,读的并发型下降, 其实不需要统一

#### 3. 加读写锁

###### 代码

```java
class MyCache {

    // 缓存的东西要保证可见性， 很重要
    private volatile Map<String, Object> map = new HashMap<>();
    // 锁的粒度太粗了, 读写都一致, 并发性落后
    //    private Lock lock = new ReentrantLock();
    private ReentrantReadWriteLock rwlock = new ReentrantReadWriteLock();

    public void put(String key, Object value) {

        rwlock.writeLock().lock();

        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在写入: " + key);
            Thread.sleep(300);
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "\t 写入完成");
        } catch (InterruptedException e) {
        } finally {
            rwlock.writeLock().unlock();
        }
    }

    public void get(String key) {

        rwlock.readLock().lock();

        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在读取");
            Thread.sleep(300);
            Object result = map.get(key);
            System.out.println(Thread.currentThread().getName() + "\t 读取完成: " + result);
        } catch (InterruptedException e) {
        } finally {
            rwlock.readLock().unlock();
        }

    }
}
```

###### 结果

```
1	 正在写入: 1
1	 写入完成
0	 正在写入: 0
0	 写入完成
2	 正在写入: 2
2	 写入完成
3	 正在写入: 3
3	 写入完成
4	 正在写入: 4
4	 写入完成
0	 正在读取
1	 正在读取
3	 正在读取
2	 正在读取
4	 正在读取
0	 读取完成: 0
4	 读取完成: 4
2	 读取完成: 2
3	 读取完成: 3
1	 读取完成: 1
```

> 写是统一体, 读不是统一体, 减小了锁的粒度, 性能被优化

#### 结论

- **不加锁: ** 读和写都不是一个完整的统一体, 中间都被打断了
- **加 ReentrantLock: **读和写都是一个完整的统一体, 锁的粒度粗,读的并发型下降, 
- **加 ReentantReadWriteLock:** 写是统一体, 读不是统一体, 减小了锁的粒度, **性能被优化**

