# synchronized异步加锁
### synchronized简介
在java代码中使用synchronized可是使用在代码块和方法中，根据Synchronized用的位置可以有这些使用场景：
```
方法:
//方法一
public synchronized void method()
{
   // todo
}

//方法二 静态锁的是类对象
public static synchronized void method()
{
   // todo
}

//代码块
public void method()
{
   synchronized(this) {
      // todo
   }
}
```

synchronized可以用在方法上也可以使用在代码块中，其中方法是实例方法和静态方法分别锁的是该类的实例对象和该类的对象。
而使用在代码块中也可以分为三种  

这里的需要注意的是：如果锁的是类对象的话，尽管new多个实例对象，但他们仍然是属于同一个类依然会被锁住，即线程之间保证同步关系。
现在我们已经知道了怎样synchronized了，看起来很简单，拥有了这个关键字就真的可以在并发编程中得心应手.

### 加锁和不加锁对比
（1）synchronized：可以在任意对象及方法上加锁，而加锁的这段代码称为“互斥区”或“临界区”。

（2）不使用synchronized实例（代码A）：
```
public class MyThread extends Thread {

    private int count = 5;

    @Override
    public void run() {
        count--;
        System.out.println(this.currentThread().getName() + " count:" + count);
    }

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        Thread thread1 = new Thread(myThread, "thread1");
        Thread thread2 = new Thread(myThread, "thread2");
        Thread thread3 = new Thread(myThread, "thread3");
        Thread thread4 = new Thread(myThread, "thread4");
        Thread thread5 = new Thread(myThread, "thread5");
        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
        thread5.start();
    }
}
```
输出结果:
```
thread3 count:2
thread4 count:1
thread1 count:2
thread2 count:3
thread5 count:0
```
可以看到，上述的结果是不正确的，这是因为，多个线程同时操作run（）方法，对count进行修改，进而造成错误。<br>
3）使用synchronized实例（代码B）:
```
public class MyThread extends Thread {

    private int count = 5;

    @Override
    public synchronized void run() {
        count--;
        System.out.println(this.currentThread().getName() + " count:" + count);
    }

    public static void main(String[] args) {
        MyThread myThread = new MyThread();
        Thread thread1 = new Thread(myThread, "thread1");
        Thread thread2 = new Thread(myThread, "thread2");
        Thread thread3 = new Thread(myThread, "thread3");
        Thread thread4 = new Thread(myThread, "thread4");
        Thread thread5 = new Thread(myThread, "thread5");
        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
        thread5.start();
    }
}
```
输出结果:
```
thread1 count:4
thread2 count:3
thread3 count:2
thread5 count:1
thread4 count:0
```
可以看出: <br>
当多个线程访问MyThread 的run方法的时候，如果使用了synchronized修饰，那个多线程就会以排队的方式进行处理（这里排队是按照CPU分配的先后顺序而定的），一个线程想要执行synchronized修饰的方法里的代码，首先是尝试获得锁，如果拿到锁，执行synchronized代码体的内容，如果拿不到锁的话，这个线程就会不断的尝试获得这把锁，直到拿到为止，而且多个线程同时去竞争这把锁，也就是会出现锁竞争的问题。

### 修饰代码块
1）一个线程访问一个对象中的synchronized(this)同步代码块时，其他试图访问该对象的线程将被阻塞
``Demo一``
```
class SyncThread implements Runnable {
       private static int count;
 
       public SyncThread() {
          count = 0;
       }
 
       public  void run() {
          synchronized(this) {
             for (int i = 0; i < 5; i++) {
                try {
                   System.out.println(Thread.currentThread().getName() + ":" + (count++));
                   Thread.sleep(100);
                } catch (InterruptedException e) {
                   e.printStackTrace();
                }
             }
          }
       }
 
       public int getCount() {
          return count;
       }
}
 
public class Demo00 {
    public static void main(String args[]){
　　　　//test01
　　　　//SyncThread s1 = new SyncThread();
　　　　//SyncThread s2 = new SyncThread();
　　　　//Thread t1 = new Thread(s1);
　　　　//Thread t2 = new Thread(s2);
　　　　//test02        
        SyncThread s = new SyncThread();
        Thread t1 = new Thread(s);
        Thread t2 = new Thread(s);
        
        t1.start();
        t2.start();
    }
}
```
执行:
```
Thread-0:0
Thread-0:1
Thread-0:2
Thread-0:3
Thread-0:4
Thread-1:5
Thread-1:6
Thread-1:7
Thread-1:8
Thread-1:9

```
当两个并发线程(thread1和thread2)访问同一个对象(syncThread)中的synchronized代码块时，在同一时刻只能有一个线程得到执行，另一个线程受阻塞，必须等待当前线程执行完这个代码块以后才能执行该代码块。Thread1和thread2是互斥的，因为在执行synchronized代码块时会锁定当前的对象，只有执行完该代码块才能释放该对象锁，下一个线程才能执行并锁定该对象.


### 对象锁的同步和异步
（1）同步：synchronized

同步的概念就是共享，我们要知道“共享”这两个字，如果不是共享的资源，就没有必要进行同步，也就是没有必要进行加锁；

同步的目的就是为了线程的安全，其实对于线程的安全，需要满足两个最基本的特性：原子性和可见性;

（2）异步：asynchronized

异步的概念就是独立，相互之间不受到任何制约，两者之间没有任何关系。

```
public class MyObject {

    public void method() {
        System.out.println(Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        final MyObject myObject = new MyObject();

        Thread t1 = new Thread(new Runnable() {
            public void run() {
                myObject.method();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {
            public void run() {
                myObject.method();
            }
        }, "t2");

        t1.start();
        t2.start();
    }
}
```

### 使用场景
1.批量导入异步导入插入数据时的并发问题:  </br>
方案:
```
    增加一个静态变量或者使用redis缓存将要保存的list

    加锁:key可以是list元素对象的元素(如:UNIQUE_COMM)
    synchronized ("key"){
        //保存操作:比较list里是不是有这条数据,有的话数据库插入数据,没有操作
        //然后去除list的这条元素
    }

```
