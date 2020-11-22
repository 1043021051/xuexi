# synchronized 实现原理

synchronized可以保证变量的原子性，可见性和顺序性，所以可以保证方法或者代码块在运行时只有一个方法可以进入临界区获取资源，同时还可以保证内存变量的内存可见性。并且synchronized是一个可重入锁:
### 代码
```
public class SynchronizedDemo1 {

    // 同步方法
    public synchronized void wc1 () {
        System.out.println("锁住A");
    }


    // 同步方法 (静态)
    public static void wc2 () {
        synchronized (SynchronizedDemo1.class) {
            System.out.println("锁住B");
        }
    }


    // 同步代码块
    public void wc3 () {
        synchronized (this) {
            System.out.println("锁住C");
        }
    }

}


```

我们现在对上面的方法进行反编译操作:

synchronized是基于进入和退出管程(Monitor)对象实现（monitorenter和monitorexit），
monitorenter指令插入到同步代码块的开始位置，monitorexit指令插入到同步代码块结束的位置，任何一个对象都有一个Monitor与之相关联，当一个线程持有Minitor后，它将处于锁定状态。 

```
LINENUMBER 8 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lcom/zero/day3/SynchronizedDemo1; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x21
  public synchronized wc1()V
   L0
    LINENUMBER 15 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "\u6211\u5728\u4e0a\u5395\u62401"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L1
    LINENUMBER 16 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/zero/day3/SynchronizedDemo1; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

  // access flags 0x9
  public static wc2()V
    TRYCATCHBLOCK L0 L1 L2 null
    TRYCATCHBLOCK L2 L3 L2 null
   L4
    LINENUMBER 21 L4
    LDC Lcom/zero/day3/SynchronizedDemo1;.class
    DUP
    ASTORE 0
    MONITORENTER
   L0
    LINENUMBER 22 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "\u6211\u5728\u4e0a\u5395\u62402"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L5
    LINENUMBER 23 L5
    ALOAD 0
    MONITOREXIT
   L1
    GOTO L6
   L2
   FRAME FULL [java/lang/Object] [java/lang/Throwable]
    ASTORE 1
    ALOAD 0
    MONITOREXIT
   L3
    ALOAD 1
    ATHROW
   L6
    LINENUMBER 24 L6
   FRAME CHOP 1
    RETURN
    MAXSTACK = 2
    MAXLOCALS = 2

  // access flags 0x1
  public wc3()V
    TRYCATCHBLOCK L0 L1 L2 null
    TRYCATCHBLOCK L2 L3 L2 null
   L4
    LINENUMBER 29 L4
    ALOAD 0
    DUP
    ASTORE 1
    MONITORENTER
   L0
    LINENUMBER 30 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "\u6211\u5728\u4e0a\u5395\u62403"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L5
    LINENUMBER 31 L5
    ALOAD 1
    MONITOREXIT
   L1
    GOTO L6
   L2
   FRAME FULL [com/zero/day3/SynchronizedDemo1 java/lang/Object] [java/lang/Throwable]
    ASTORE 2
    ALOAD 1
    MONITOREXIT
   L3
    ALOAD 2
    ATHROW
   L6
    LINENUMBER 32 L6
   FRAME CHOP 1
    RETURN
   L7
    LOCALVARIABLE this Lcom/zero/day3/SynchronizedDemo1; L4 L7 0
    MAXSTACK = 2
    MAXLOCALS = 3
}
```
