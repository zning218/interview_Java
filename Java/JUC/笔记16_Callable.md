## Callable 接口

#### 继承关系

![FutureTask继承关系](E:\Git\秋招_基础知识\Java\JUC\图片\Callable\FutureTask继承关系.png)

###### FutureTask 构造器

```java
FutureTask(Callable<V> callable) 
Creates a FutureTask that will, upon running, execute the given Callable.
```

###### 代码实现

```java
public class CallableDemo {

    public static void main(String[] args) throws Exception {
        // 创建任务
        FutureTask<Integer> futureTask = new FutureTask(new MyThread());

        new Thread(futureTask, "AA").start();
        // 如果 FutureTask 一样, 只执行一次
        // new Thread(futureTask, "BB").start();

        int result01 = 100;
        System.out.println("------------------------------------");

        // AA线程还没有算完, 可以执行其他指令
        while (!futureTask.isDone()) {
            // System.out.println("主线程先做点别的事!");
        }

        // 要求获得 Callable线程的计算结果, 如果没有计算完成就要求去强求, 会导致阻塞,值得计算完成
        // 建议放在最后
        int result02 = futureTask.get();
        System.out.println("result: " + (result01 + result02));
    }
}

class MyThread implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        System.out.println("-------- come in Callable");
        TimeUnit.SECONDS.sleep(2);
        return 1024;
    }
}
```

## 创建线程的第三种方式

使用 `Callable` 

### 与 `Runnable`的区别

1. 方法可以有返回值
2. 可以抛出异常

### 使用

1. 执行 `Callable方式`, 需要 `FutureTask` 实现类的支持, 用于接口运算结果.

   ```java
   TheadDemo02 td = new TheadDemo02();
   FutureTask<Integer> result = new FutureTask<>(td);
   new Thread(result).start();
   ```

2. 接收线程运算后的结果

   ```java
   int sum = result.get(); // FutureTask 也可以用于闭锁的操作
   System.out.println("sum: " + sum);
   ```

###### 代码

```java
public class TestCallable {

    public static void main(String[] args) throws Exception {
        TheadDemo02 td = new TheadDemo02();

        // 1.执行 Callable方式, 需要 FutureTask 实现类的支持, 用于接口运算结果.
        FutureTask<Integer> result = new FutureTask<>(td);
        new Thread(result).start();

        // 2.接收线程运算后的结果
        int sum = result.get(); // FutureTask 也可以用于闭锁的操作
        System.out.println("sum: " + sum);

    }
}

class TheadDemo02 implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        int sum = 0;

        for (int i = 0; i <= 100; i++) {
            sum++;
        }
        return sum;
    }
}
```

### 应用

多线程计算, 最后汇总到一起,提高执行效率, 会比单线程高很多

