---
title: Synchronized关键字原理分析
date: 2022-06-16 10:36:00
tags:
  - java
  - jvm
search: true
---



### Sync 使用

1. 修饰实例方法，锁的是当前对象的锁（锁是什么？）

   ```java
   synchronized void method() {
       //业务代码
   }
   ```

2. 修饰静态方法，锁的是当前类的锁（锁是什么？）

   ```java
   synchronized static void method() {
       //业务代码
   }
   ```

3. 修饰代码块，锁的给定对象的锁

   ```java
   synchronized(this) {
       //业务代码
   }
   
   synchronized(Object.Class) {
       //业务代码
   }
   ```

### Sync 特性

#### 原子性

原子性是指一个操作是不可以中断的，要么全部执行成功，要么全部失败

```java
// 实例同步方法
public synchronized void add(){
    count++;
}
// 静态同步方法
public static synchronized void add(){
    count++;
}
// 同步代码块
public void add(){
	synchronized (this) {
		count++;
	}
}
```

以上方法反编译后的结果

```java
// 实例同步方法
public synchronized void add();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED // 带有 ACC_SYNCHRONIZED 标记
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field count:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field count:I
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   Lorg/itstack/interview/Test;
// 静态同步方法
public static synchronized void add();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED // 带有 ACC_SYNCHRONIZED 标记
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #2                  // Field count:I
         3: iconst_1
         4: iadd
         5: putstatic     #2                  // Field count:I
         8: return
      LineNumberTable:
        line 12: 0
        line 13: 8
 // 同步代码块           
public void add();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter					// 获取 Monitor 
         4: aload_0
         5: dup
         6: getfield      #2                  // Field count:I
         9: iconst_1
        10: iadd
        11: putfield      #2                  // Field count:I
        14: aload_1
        15: monitorexit						// 释放 Monitor 
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit						// 出现异常，释放 Monitor 
        22: aload_2
        23: athrow
        24: return
      Exception table:
         from    to  target type
             4    16    19   any
            19    22    19   any
      LineNumberTable:
        line 12: 0
        line 13: 4
        line 14: 14
        line 15: 24
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      25     0  this   Lorg/itstack/interview/Test;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class org/itstack/interview/Test, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
 
```

- 同步方法和静态方法

  当一个线程调用某一个方法时，检查到当前方法带有 ACC_SYNCHRONIZED 标记，会尝试获取锁，成功就执行代码；否则，根据锁的类型进行阻塞或者循环等待

- 同步代码块

  - `monitorenter`，在判断拥有同步标识 `ACC_SYNCHRONIZED` 抢先进入此方法的线程会优先拥有 Monitor 的 owner ，此时计数器 +1。
  - `monitorexit`，当执行完退出后，计数器 -1，归 0 后被其他进入的线程获得。

#### 可见性

一个共享变量被一个线程修改后，其他线程能够感知到修改

- volatile 

  通过 lock 前缀指令，设置内存屏障。当一个线程执行了修改操作，将把共享变量的值刷新到主内存，并使其他线程的缓存失效

- synchronized

  - 线程解锁前，必须把共享变量的最新值刷新到主内存中。
  - 线程加锁前，将清空工作内存中共享变量的值，从而使用共享变量时需要从主内存中重新读取最新的值。

#### 有序性

如果在本线程内观察，所有的操作都是有序的；如果在一个线程观察另一个线程，所有的操作都是无序的。

- volatile 通过禁止指令重排保证了有序性
- synchronized 通过加锁，保证了代码块只能有一个线程进入

#### 重入性

允许一个线程多次请求自己持有对象锁的临界资源

```java
public  static synchronized void add0(){
    count++;
    add1();
}
public static synchronized  void add1(){
    count++;
    add2();
}
public static synchronized  void add2(){
    count++;
}
```

以上代码，不会出现死锁，因为 synchronized 锁对象的时候有个计数器，他会记录下线程获取锁的次数，在执行完对应的代码块之后，计数器就会-1，直到计数器清零，就释放锁了。

### Sync 原理

- 同步方法

  在方法的字节码文件中的，flags字段有 `ACC_SYNCHRONIZED` 标记，表示是一个同步方法，从而执行同步调用

- 同步代码块

  在进入临界区时执行 `monitorenter` 指令，退出临界区时执行 `monitorexit` 指令，如果方法出现异常，会执行 `monitorexit` 指令后抛出异常

### 锁的类型和优化

#### 对象头结构

![图 15-1 64位JVM对象结构描述](https://bugstack.cn/assets/images/2020/interview/interview-15-01.png)

#### 锁类型

- 偏向锁

  锁被偏向于第一个获取它的线程，如果在接下来的执行过程中，锁都没有被其他线程获取，那么持有该偏向锁的线程，将不会进行同步（加锁、解锁和对Mark Word更新操作等）。

  注意：如果对象在加锁前，已经计算过 `HashCode` 等信息，会跳过偏向锁，直接升级为轻量锁。同理，如果当锁的状态是偏向锁时，需要计算 `HashCode` 时，锁会升级到轻量锁。

- 轻量锁

  当锁是偏向锁的时候，被另一个线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，提高性能。（JDK6 新增自适应自旋，优化自旋次数和过程）

  在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），JVM虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。

- 重量锁

  会指向`ObjectMonitor`对象，使用操作系统底层互斥量进行系统的同步


**对象监视器（Monitor）的数据结构**

```c++
  // initialize the monitor, exception the semaphore, all other fields
  // are simple integers or pointers
  ObjectMonitor() {
    _header       = NULL;
    _count        = 0;			// 记录当前线程获取锁的次数（可以重入）
    _waiters      = 0,			
    _recursions   = 0;			// 线程重入次数
    _object       = NULL;	
    _owner        = NULL;		// 指向持有 ObjectMonitor对象的线程
    _WaitSet      = NULL;		// 存放处于 wait 状态的线程队列
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;		
    FreeNext      = NULL ;
    _EntryList    = NULL ;		// 处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }
```

当锁膨胀为重量级锁时，对象头中 Mark Word  字段将会指向 `ObjectMonitor` 对象

![](https://assets.hruit.cn/images/2022/05/27/upload_pmqev3pfezg2tj4h0es0xws1ka4lua4o.png)

**TODO**

- [ ] ObjectMonitor 源码

### 锁升级流程

TODO