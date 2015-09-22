---
layout: post
title:  Java引用简介
date:   2015-09-18 17:47:21
tags:
- java 
---

java中的引用总共分为4种，在java.lang.ref包下可以找到5种：

1. 强引用。默认就是它。只要引用还存在，任何时候发生GC都不会被回收，即便发生OOM

```
Obj obj1 = new Obj();
```

2. 软引用。在内存不足时被回收。

```
SoftReference<Obj> obj2 = new SoftReference(new Obj("soft"));
```

3. 弱引用。GC发生时就被回收。

```
WeakReference<Obj> obj3 = new WeakReference(new Obj("weak"));
```

4. 虚引用。创建之后就无法访问（不一定被回收了）。遇到GC就被回收。

```
ReferenceQueue<Obj> referenceQueue = new ReferenceQueue();
PhantomReference<Obj> obj4 = new PhantomReference(new Obj("phantom"), referenceQueue);
```

5. FinalReference。这个类用户程序用不到，由虚拟机使用。java没有析构函数，但有个finalize方法。所有对象构造/克隆或是初始化过程中（有参数可以设置不在构造时注册）都会注册到Finalizer里，这些Finalizer会组成一个链。由一个特殊的FinalizerThread进行调用finalize方法。一旦finalize被调用意味着他的‘死期’将不远了

```
import java.lang.ref.Reference;
import java.lang.ref.ReferenceQueue;
import java.lang.ref.SoftReference;
import java.lang.ref.WeakReference;
import java.lang.ref.PhantomReference;
import java.util.ArrayList;
import java.util.List;

/**
 * -Xmx1m -Xms1m -XX:+PrintGCDetails
 */
public class ReferenceTest {

    public static void main(String... args) throws InterruptedException {

        ReferenceQueue<Obj> referenceQueue = new ReferenceQueue();
        Obj obj1 = new Obj("strong");//就算OOM也不挂
        SoftReference<Obj> obj2 = new SoftReference(new Obj("soft"));//内存不足，挂掉
        WeakReference<Obj> obj3 = new WeakReference(new Obj("weak"));//遇见GC就挂掉
        PhantomReference<Obj> obj4 = new PhantomReference(new Obj("phantom"), referenceQueue);//创建后就挂了

        Thread.sleep(100);
        print(obj1, obj2, obj3, obj4);//测试PhantomReference

        Thread.sleep(100);
        System.gc();
        print(obj1, obj2, obj3, obj4);//测试WeakReference

        try {
            overMemory();
        } catch (Throwable e) {
            Thread.sleep(100);//测试SoftReference，StrongReference
            print(obj1, obj2, obj3, obj4);
        }
    }

    static void print(Obj obj1, Reference obj2, Reference obj3, Reference obj4) {
        System.out.println("\n===========");
        System.out.println("strong  :" + obj1);
        System.out.println("soft    :" + obj2.get());
        System.out.println("weak    :" + obj3.get());
        System.out.println("phantom :" + obj4.get());
    }

    static void overMemory() {
        List<Byte> list = new ArrayList<Byte>();
        while (true) {
            list.add(new Byte((byte) 1));
        }
    }
}

class Obj {
    String name;

    Obj(String name) {
        this.name = name;
    }

    public String toString() {
        return name;
    }

}
```

以上代码打印结果：

```
===========
strong  :strong
soft    :soft
weak    :weak
phantom :null

===========
strong  :strong
soft    :soft
weak    :null
phantom :null

===========
strong  :strong
soft    :null
weak    :null
phantom :null
```

- 第一次print时,虛引用已经不可用。通过GC日志发现这个时候未发生GC。

- 第二次print时已经发生了一次GC，这时虚引用与弱引用都会null。说明虚引用已经被回收掉

- 第三次print时已经发生oom，这个时候发现软引用也已经被回收。软引用是在发生OOM前被回收的。这个时候strong仍然还存活。

