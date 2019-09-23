## 请谈谈你对 OOM 的认识？

一下均为出现 OOM的情况

#### 1. java.lang.StackOverflowError

在一个函数中调用自己就会产生这个错误, 递归调用

##### 模拟

```java
public class MyJava {
    public static void main(String[] args) throws InterruptedException {
        me();
    }
	
    // 递归调用自己
    private static void me() {
        me();
    }
}
```

##### 结果

```
Exception in thread "main" java.lang.StackOverflowError
	at test.MyJava.me(MyJava.java:20)
	at test.MyJava.me(MyJava.java:20)
	at test.MyJava.me(MyJava.java:20)
	...
```



#### 2. java.lang.OutOfMemoryError : Java heap space

堆内存溢出, new 一个很大对象, 或者 new了很多的对象 

##### 模拟

```java
public class MyJava {
    public static void main(String[] args) throws InterruptedException {
        // 方法一: 创建大对象, 撑爆内存
        byte[] bytes = new byte[20 * 1024 * 1024];

        // 方法二: new很多的对象
        String str = "aaa";
        while (true) {
            str += str + new Random().nextInt(1111111111) + new Random().nextInt(222222222);
            str.intern();
        }
    }
}
```

##### 结果

```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at test.MyJava.main(MyJava.java:17)
```



#### 3. java.lang.OutOfMemoryError : GC overhead limit exceeded

执行垃圾收集的时间比例太大， 有效的运算量太小，默认情况下,，如果GC花费的时间超过 **98%**， 并且GC回收的内存少于 **2%**， JVM就会抛出这个错误。

> 假如不抛出此异常, 那么GC清理的这么点内存很快会再次填满, 迫使GC再次执行, 这样就形成了恶性循环, CPU使用率一直是 100%, 而 GC却没有任何成果. 

##### 模拟

```java
public static void main(String[] args) throws InterruptedException {
    int i = 0;
    List<String> list = new ArrayList<>();
    try {
        while (true)
            list.add(String.valueOf(++i).intern());
    } catch (Throwable e) {
        System.out.println("====================== i:" + i);
        e.printStackTrace();
        throw e;
    }
}
```

##### 结果

```
[Full GC (Ergonomics) [PSYoungGen: 2047K->2047K(2560K)] [ParOldGen: 7082K->7082K(7168K)] 9130K->9130K(9728K), [Metaspace: 3579K->3579K(1056768K)], 0.0496168 secs] [Times: user=0.11 sys=0.00, real=0.05 secs] 
[Full GC (Ergonomics) [PSYoungGen: 2047K->2047K(2560K)] [ParOldGen: 7084K->7084K(7168K)] 9132K->9132K(9728K), [Metaspace: 3579K->3579K(1056768K)], 0.0534932 secs] [Times: user=0.06 sys=0.00, real=0.05 secs] 
====================== i:145204
[Full GC (Ergonomics) [PSYoungGen: 2047K->0K(2560K)] [ParOldGen: 7100K->649K(7168K)] 9148K->649K(9728K), [Metaspace: 3604K->3604K(1056768K)], 0.0246396 secs] [Times: user=0.00 sys=0.00, real=0.02 secs] 
java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.lang.Integer.toString(Integer.java:401)
	at java.lang.String.valueOf(String.java:3099)
	at test.MyJava.main(MyJava.java:25)
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.lang.Integer.toString(Integer.java:401)
	at java.lang.String.valueOf(String.java:3099)
	at test.MyJava.main(MyJava.java:25)
...
```



#### 4. java.lang.OutOfMemoryError : Direct buffer memory

##### 导致原因:

与NIC巷序经常使ByteBuffer来读取或者写入数据，这是-种基于通道(Channel)与缓冲区(Buffer)的/0方式,

它可以使用Native函数库直接分配堆外内存，然后通过一个 存储在Java堆里面的irectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场最中显著提高性能， 因为避免了在Java雄和Native堆中来回复制数据。

* ByteBuffer. al locate(capability) 第 1种方式是分配VM堆内存，属于GC管辖范围，由于需要拷贝所以速度相对较慢

* ByteBuffer. al locteDirect(capability) 第2种方式是分能OS本地内存，不属于GC管辖范围，由于不需要内存拷贝所以速度相对较块。

但如果不断分配本地内存， 堆内存很少使用，那么JVM就不需 要执行GC，DirectByteBuffer对象 们就不会被回收，这时候堆内存充足，但本地内存可能已经使用光了，再次尝试分配本地内存就会出现OutOfMemoryError,那程序就直接崩溃了。

##### 验证

配置参数：-Xms10m -Xmx10m -XX:+PrintGCDetails -XX:MaxDirectMemorySize=5m

```java
public class DirectBufferDemo {
    public static void main(String[] args) {
        System.out.println("maxDirectMemory : " + sun.misc.VM.maxDirectMemory() / (1024 * 1024) + "MB");
        // 为我们配置为 5MB, 但实际使用 6MB
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(6 * 1024 * 1024);
    }
}
```

##### 输出

```
maxDirectMemory : 5MB
[GC (System.gc()) [PSYoungGen: 1315K->464K(2560K)] 1315K->472K(9728K), 0.0008907 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [PSYoungGen: 464K->0K(2560K)] [ParOldGen: 8K->359K(7168K)] 472K->359K(9728K), [Metaspace: 3037K->3037K(1056768K)], 0.0060466 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
	at java.nio.Bits.reserveMemory(Bits.java:694)
	at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:123)
	at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)
	at com.cuzz.jvm.DirectBufferDemo.main(DirectBufferDemo.java:17)
Heap
 PSYoungGen      total 2560K, used 56K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
  eden space 2048K, 2% used [0x00000000ffd00000,0x00000000ffd0e170,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 7168K, used 359K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
  object space 7168K, 5% used [0x00000000ff600000,0x00000000ff659e28,0x00000000ffd00000)
 Metaspace       used 3068K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 336K, capacity 388K, committed 512K, reserved 1048576K
```





#### 5. java.lang.OutOfMemoryError : unable to create new native thread

高并发请求服务器时，经常出现如下异常: java. Lang. OutOfMemoryError: unable to create new native thread准确的讲该native thread异常与对应的平台有关

导致原因:

1  你的应用创建了太多线程了,一个应 用进程创建多个线程，超过系统承载极限

2  你的服务器并不允许你的应用程序创建这么多线程，linux系统默认允许单个进程可以创建的线程数是1024个，

    你的应用创建超过这个数量,就会报java.lang.OutOfMemoryError: unable to create new native thread

解决办法:

1. 想办法降低你应用程序创建线程的数量,分析应用是否真的需要创建这么多线程，如果不是,改代码将线程数降到最低

2. 对于有的应用，确实需要创建很多线程，远超过inux系统的默认1024个线程的限制,可以通过修改Linux服务器配置,扩大l inux默认限制



##### 验证

模拟, 在 win上运行

```java
public class MyJava {
    public static void main(String[] args) throws InterruptedException {

        // 没有结束条件
        for (int i = 1; ; i++) {
            System.out.println("-------------- i: " + i);

            new Thread(() -> {
                try {
                    Thread.sleep(Integer.MAX_VALUE);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "" + i).start();
        }
    }
}
```

结果

```java
-------------- i: 11294
-------------- i: 11295
-------------- i: 11296
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
	at java.lang.Thread.start0(Native Method)
	at java.lang.Thread.start(Thread.java:717)
	at test.MyJava.main(MyJava.java:28)
```



#### 6. java.lang.OutOfMemoryError : Metaspace

- Java 8 之后的版本使用元空间（Metaspace）代替了永久代，元空间是方法区在 HotSpot 中的实现，它与持久代最大的区别是：元空间并不在虚拟机中的内存中而是使用本地内存。
- 元空间存放的信息：
  - 虚拟机加载的类信息
  - 常量池
  - 静态变量
  - 即时编译后的代码

具体的实现可以看看这个帖子：[几种手动OOM的方式](http://www.dataguru.cn/thread-351920-1-1.html)