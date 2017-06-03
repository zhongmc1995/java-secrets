## JVM判断对象是否死亡的常用算法
JVM在对对象进行垃圾回收之前，首先要先判断对象的状态，对象是否存活还是已经死亡，都是JVM进行垃圾回收的依据
## 引用计数算法
事先给一个对象赋予一个引用计数器，当有一个地方引用给对象是就给引用计数器加一，当失效时就给引用计数器减一，
任何时刻引用计数器为零的对象都是不可再被引用的了，也就是可能要被垃圾回收的对象。这种方法在大多数场景都是
有用且非常高效的，但是java语言没有选用这种算法主要是因为它很难解决对象相互循环引用的问题。
## 根搜索算法
根搜索算法的基本思路是：通过一系列的名为“GC Roots”的对象作为起点，从这些节点向下搜索，搜索走过的路径称为
引用链(Reference Chain),当一个对象到GC Roots没有任何引用链相连的时候，也就是所从GC Roots到这个对象不
可达的时候，则证明该对象是不可用的
在java语言里面，可作为GC Roots的对象有如下：    
    1. 虚拟机栈(栈帧中的本地变量表)中的引用对象   
    2. 方法区中的静态属性引用的对象   
    3. 方法区中常量引用的对象     
    4. 本地方法栈中的JNI(即常说的Native方法)的引用对象    
## Java中引用类型
1.	强引用：强引用是代码中最普遍存在的，比如Object o = new Object()；只要这个引用还存在，对象就永远不会被垃圾回收器回收
2.	软引用：软引用是用来描述一些还是有用，但是非必须的对象，对于软引用关联的对象，如果虚拟机的内存吃紧，垃圾回收器会将这些对象列入回收范围之中并进行第二次回收，如果在第二次回收之后内存还是不足就抛出内存溢出的异常，在jdk1.2之后提供了softReference类来实现软引用
3.	弱引用：弱引用是用来描述非必须的对象，被弱引用关联的对象只能生存到垃圾回收器进行垃圾收集之前，当回收工作开始后，无论内存是否足够，都会被回收。Java中提供了weakReference类来实现弱引用
4.	虚引用：它是最弱的一种引用，一个对象有虚引用的存在，完全不会关系到它的生存周期，也不能通过虚引用获得一个对象的实例，弱引用存在的唯一目的就是，当为一个对象设置弱引用时，希望这个对象被垃圾回收器回收的时候收到一个系统通知，java中提供PhantomReference类来实现虚引用。
## 对象死亡的全过程
Java中是使用根搜索算法来判断对象的状态的，在该算法中，不可达的对象，也并非是“非死不可”，这个时候这类对象只是暂时处于“缓刑”阶段，一个对象要真正宣告死亡
至少需要经历两次被标记的过程：一个对象在进行根搜索说法中发现是不可达的时候，那么这类对象会被第一次标记同时进行一次筛选，筛选的条件是是否有必要执行该对象
的finalize()方法。当对象没有覆盖这个方法，或者这个方法已经被JVM调用过时，JVM会将这两种情况作为没有必要执行。
如果这个对象被判定为有必要执行finalize()方法时，那么这个对象会被放置在一个F-Queue的队列中，并在稍后由虚拟机自动建立一个低优先级的Finalizer线程去执行
这个方法。这个过程都是JVM自动帮我们做的。finalize()方法是对象逃脱死亡的最后一次机会，在这个方法中，GC将对F-Queue中的对象进行第二次小规模的标记，如果
在这个方法中成功拯救了自己---只要重新与引用链上的任何一个对象建立关联即可，比如把自己赋值给某个变量，那么在第二次进行标记的时候就会被移除“即将回收”的集合；
如果这个时候对象还没有逃脱，那么它离死亡就不远啦
## 对象自我拯救

### 代码示例
```java
package com.briup.jvm;


/**
 * Created by zhongmc on 2017/6/3.
 */
public class FinalizeGCTest {
    public static FinalizeGCTest INS = null;
    public void isAlive(){
        System.out.println("i am alive");
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize method executed");
        //自我拯救
        INS = this;
    }

    public static void main(String[] args) throws InterruptedException {
        INS = new FinalizeGCTest();
        //INS第一次成功拯救
        INS = null;
        System.gc();

        Thread.sleep(1000);
        if (INS!=null){
            INS.isAlive();
        }else{
            System.out.println("i am dead");
        }

        //INS第二次成功拯救
        INS = null;
        System.gc();
        //Finalizer线程优先级比较低，先让主线程休眠，等待Finalizer执行完毕
        Thread.sleep(1000);
        if (INS!=null){
            INS.isAlive();
        }else{
            System.out.println("i am dead");
        }

        /**
         * 输出结果：
         *  finalize method executed
            i am alive
            i am dead
         */
    }

}
```
上述代码中第一次INS自我拯救成功，第二次却失败了，这个为什么呢？这是因为在第一次拯救成功是比较好理解的，该对象重写了
finalize()方法且没有被JVM调用过，并且在finalize()方法中该对象经自己(this)赋值给了一个静态变量(INS)，因此能自我
拯救成功；但是第二次因为之前JVM调用了一次finalize方法，所以JVM会视为没有必要执行finalize()方法，所以这个拯救失败

