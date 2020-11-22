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

synchronized可以用在方法上也可以使用在代码块中，其中方法是实例方法和静态方法分别锁的是该类的实例对象和该类的对象。而使用在代码块中也可以分为三种，具体的可以看上面的表格。这里的需要注意的是：如果锁的是类对象的话，尽管new多个实例对象，但他们仍然是属于同一个类依然会被锁住，即线程之间保证同步关系。
现在我们已经知道了怎样synchronized了，看起来很简单，拥有了这个关键字就真的可以在并发编程中得心应手了吗？爱学的你，就真的不想知道synchronized底层是怎样实现了吗？
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
