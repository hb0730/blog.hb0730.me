---

title: Java四种引用

date: 2021-03-09 14:03:48

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Java' ]

categories: [ 'Java' ]

---

# 四种引用的区别

其实四种引用的区别在于 **GC的时候，对它们的处理不同**。用一句话来概括，就是：如果一个**对象GC Root可达**，**强引用不会被回收**，**软引用在内存不足时会被回收**，**弱引用在这个对象第一次GC会被回收**。

> 如果GC Root不可达，那不论什么引用，都会被回收

**虚引用比较特殊，等于没有引用，不会影响对象的生命周期**，但可以在对象被收集器回收时收到一个系统通知。

下面结合案例分别来讲一下四种引用在面对GC时的表现以及它们的常见用途。先设置一下JVM的参数：

`-Xms20M -Xmx20M -Xmn10M -verbose:gc -XX:+PrintGCDetails`

## 强引用（StrongReference）

这就是我们平时**最常使用的引用**。**只要GC的时候这个对象GC Root可达，它就不会被回收**。如果JVM内存不够了，直接抛出OOM。比如下面这段代码就会抛出`OutOfMemoryError`：

```java
public static void main(String[] args) {
    List<Object> list = new LinkedList<>();
    for (int i = 0; i < 21; i++) {
        list.add(new byte[1024 * 1024]);
    }
}
```

## 软引用（SoftReference）

软引用: 当GC的时候，如果GC Root可达，如果 **内存足够，就不会被回收**；如果 **内存不够用，会被回收**。将上面的例子改成软引用，就不会被OOM：

```java
public static void main(String[] args) {
    List<Object> list = new LinkedList<>();
    for (int i = 0; i < 21; i++) {
        SoftReference<byte[]> softReference = new SoftReference<>(new byte[1024 * 1024]);
        list.add(softReference);
    }
}
```

我们把程序改造一下，打印出GC后的前后的差别：

```java
public static void main(String[] args) {
    List<SoftReference<byte[]>> list = new LinkedList<>();
    for (int i = 0; i < 21; i++) {
        SoftReference<byte[]> softReference = new SoftReference<>(new byte[1024 * 1024]);
        list.add(softReference);
        System.out.println("gc前:" + softReference.get());
    }
    System.gc();
    for (SoftReference<byte[]> softReference : list) {
        System.out.println("gc后:" + softReference.get());
    }
}
```

会发现，打印出的日志，GC前都是有值的，而GC后，会有一些是`null`，代表它们已经被回收。

而我们设置的堆最大为20M，如果把循环次数改成15，就会发现打印出的日志，GC后没有为`null`的。但通过`-verbose:gc -XX:+PrintGCDetails`参数能发现，JVM还是进行了几次GC的，只是由于内存还够用，所以没有回收。

```java
public static void main(String[] args) {
    List<SoftReference<byte[]>> list = new LinkedList<>();
    for (int i = 0; i < 15; i++) {
        SoftReference<byte[]> softReference = new SoftReference<>(new byte[1024 * 1024]);
        list.add(softReference);
        System.out.println("gc前:" + softReference.get());
    }
    System.gc();
    for (SoftReference<byte[]> softReference : list) {
        System.out.println("gc后:" + softReference.get());
    }
}
```

所以软引用的常见用途就呼之欲出了：**缓存**。尤其是那种希望这个缓存能够持续时间长一点的。

如果一个**对象只具有软引用**，那就类似于**可有可无的生活用品**。如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用**可用来实现内存敏感的高速缓存**。

软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，JAVA 虚拟机就会把这个软引用加入到与之关联的引用队列中。

## 弱引用（WeakReference）

**弱引用，只要这个对象发生GC，就会被回收。**

把上面的代码改成弱引用，会发现打印出的日志，GC后全部为`null`。

```java
public static void main(String[] args) {
    List<WeakReference<byte[]>> list = new LinkedList<>();
    for (int i = 0; i < 15; i++) {
        WeakReference<byte[]> weakReference = new WeakReference<>(new byte[1024 * 1024]);
        list.add(weakReference);
        System.out.println("gc前:" + weakReference.get());
    }
    System.gc();
    for (WeakReference<byte[]> weakReference : list) {
        System.out.println("gc后:" + weakReference.get());
    }
}
```

所以**弱引用也适合用来做缓存**，不过由于它是只要发生GC就会被回收，所以存活的时间比软引用短得多，通常用于做一些**非常临时的缓存**。

我们知道，`WeakHashMap`内部是通过弱引用来管理entry的。它的键是 **“弱键”**，所以在GC时，它对应的键值对也会从Map中删除。

> Tomcat中有一个`ConcurrentCache`，用到了`WeakHashMap`，结合`ConcurrentHashMap`，实现了一个线程安全的缓存，感兴趣的同学可以研究一下源码，代码非常精简，加上所有注释，只有短短59行。

`ThreadLocal`中的静态内部类`ThreadLocalMap`里面的entry是一个`WeakReference`的继承类。

>  使用弱引用，使得`ThreadLocalMap`知道`ThreadLocal`对象是否已经失效，一旦该对象失效，也就是成为垃圾，那么它所操控的Map里的数据也就没有用处了，因为外界再也无法访问，进而决定擦除Map中相关的值对象，Entry对象的引用，来保证Map总是保持尽可能的小。

如果一个**对象只具有弱引用**，那就类似于**可有可无的生活用品**。

弱引用与软引用的区别在于：**只具有弱引用的对象拥有更短暂的生命周期**。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。

弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。

## 虚引用（PhantomReference）

虚引用的设计和上面三种引用有些不同，它并不影响GC，而是为了在对象被GC时，能够收到一个系统通知。

那它是怎么被通知的呢？虚引用必须要配合`ReferenceQueue`，当GC准备回收一个对象，如果发现它还有虚引用，就会在回收之前，把这个虚引用加入到与之关联的`ReferenceQueue`中。

> 那NIO是如何利用虚引用来管理内存的呢？

`DirectBuffer`直接从Java堆之外申请一块内存, 这块内存是不直接受JVM GC管理的, 也就是说在GC算法中并不会直接操作这块内存. 这块内存的GC是由于DirectBuffer在Java堆中的对象被GC后, 通过一个通知机制, 而将其清理掉的.

DirectBuffer内部有一个`Cleaner`。这个Cleaner是PhantomReference的子类。当DirectBuffer对象被回收之后, 就会通知到PhantomReference。然后由ReferenceHandler调用`tryHandlePending()`方法进行`pending`处理. 如果pending不为空, 说明DirectBuffer被回收了, 就可以调用Cleaner的`clean()`进行回收了。

上面这个方法的代码在`Reference类`里面，有兴趣的同学可以去看一下那个方法的源码。

"虚引用"顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。

**虚引用主要用来跟踪对象被垃圾回收的活动**。

虚引用与软引用和弱引用的一个区别在于： **虚引用必须和引用队列（ReferenceQueue）联合使用**。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

特别注意，在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为**软引用可以加速 JVM 对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生**。

# 总结

以上就是Java中四种引用的区别。一般来说，强引用我们都知道，虚引用很少用到。而软引用和弱引用的区别在于回收的时机：软引用GC时，发现内存不够才回收，弱引用只要一GC就会回收。

## 参考

* [什么是GC Roots](https://blog.hb0730.com/archives/%E4%BB%80%E4%B9%88%E6%98%AFgcroots)

作者: yasinshaw
链接: https://yasinshaw.com/articles/90
来源: 个人博客
