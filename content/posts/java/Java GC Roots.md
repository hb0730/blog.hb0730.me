---
title: Java GC Roots

date: 2021-03-29 15:45:56

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Java' ]

categories: [ 'Java' ]

---

> 可达性算法中以`GC Root`对象为起点开始搜索。

> GC Roots 是**类的静态变量**，或者**方法的局部变量**

![GC Roots](https://hb0730-blog-hk.oss-cn-hongkong.aliyuncs.com/GC%20Roots_1621921497696.jpg)

# 什么是GC Root对象

## 虚拟机栈中引用的对象

```java
public class Rumenz{
    public static  void main(String[] args) {
         Rumenz a = new Rumenz();
         a = null;
    }

}
```

> a是栈帧中的本地变量,a就是GC Root,由于 `a=null`,a与`new Rumenz()`对象断开了链接,所以对象会被回收。

## 方法区类的静态成员引用的对象

```java
public class Rumenz{
    public static Rumenz r;
    public static void main(String[] args){
       Rumenz a=new Rumenz();
       a.r=new Rumenz();
       a=null;
    }
}
```

> 栈帧中的本地变量`a=null`,由于`a`断开了与`GC Root`对象(a对象)的联系,所以`a`对象会被回收。由于给`Rumenz`的成员变量`r`赋值了变量的引用,并且`r`成员变量是静态的,所以`r`就是一个`GC Root`对象,所以`r`指向的对象不会被回收。

## 方法区常量引用的对象

```java
public class Rumenz{
    public static final Rumenz r=new Rumenz();
    public static void main(String[] args){
       Rumenz a=new Rumenz();
       a=null;
    }

}
```

> 常量`r`引用的对象不会因为`a`引用的对象的回收而被回收。

## 本地方法栈中JNI引用的对象

```java
JNIEXPORT void JNICALL Java_com_pecuyu_jnirefdemo_MainActivity_newStringNative(JNIEnv *env, jobject instance，jstring jmsg) {
...
   // 缓存String的class
   jclass jc = (*env)->FindClass(env, STRING_PATH);
}
```

![34292096757e25e28245a124a3_fix732.jpg](https://hb0730-blog-hk.oss-cn-hongkong.aliyuncs.com/3429209675-7e25e28245a124a3_fix732_1617003649136.jpg)

> 本地方法就是一个 java 调用非 java 代码的接口，该方法并非 Java 实现的，可能由 C 或 Python等其他语言实现的， Java 通过 JNI 来调用本地方法， 而本地方法是以库文件的形式存放的（在 WINDOWS 平台上是 DLL 文件形式，在 UNIX 机器上是 SO 文件形式）。通过调用本地的库文件的内部方法，使 JAVA 可以实现和本地机器的紧密联系，调用系统级的各接口方法，

> 当调用 Java 方法时，虚拟机会创建一个栈桢并压入 Java 栈，而当它调用的是本地方法时，虚拟机会保持 Java 栈不变，不会在 Java 栈祯中压入新的祯，虚拟机只是简单地动态连接并直接调用指定的本地方法。

作者：入门小站
链接：https://segmentfault.com/a/1190000038405609
