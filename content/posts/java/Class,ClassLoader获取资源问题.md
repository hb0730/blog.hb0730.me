---

title: Java Class ClassLoader 资源加载

date: 2021-04-16 09:37:14

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Java' ]

categories: [ 'Java' ]

---

> 原因：在看Java SPI的时候`ServiceLoader.load(Class<S> service) `,发现是通过通过当前线程的`Thread.currentThread().getContextClassLoader()`然后通过`PREFIX = "META-INF/services/"`这种规定去找到`service` ,这不禁想到之前做的[dbvc](https://github.com/hb0730/dbvc)使用springBoot的`PathMatchingResourcePatternResolver` 来加载`classpath`路径下的资源文件

## 案例

```java
 URL url = ResourcesUtils.getResource("/sql/test.sql", this.getClass()); //1
 url = ResourcesUtils.getResource("sql/test.sql", this.getClass()); //2
 url = ResourcesUtils.getResource("sql/test.sql", this.getClass().getClassLoader()); //3
 url = ResourcesUtils.getResource("/sql/test.sql", this.getClass().getClassLoader()); //4
```

得到的结果

```
file:/E:/IdeaWork/customer-work/hb0730-commons/commons-lang/target/test-classes/sql/test.sql
null
file:/E:/IdeaWork/customer-work/hb0730-commons/commons-lang/target/test-classes/sql/test.sql
null
```

## 源码分析

## 通过Class得到的resource

```java
 //Class类
 public java.net.URL getResource(String name) {
        name = resolveName(name);
        ClassLoader cl = getClassLoader0(); // 获取加载该Class的ClassLoader，sun.misc.Launcher$AppClassLoader@18b4aac2
        if (cl==null) {  //如果加载该Class的ClassLoader为null，则表示这是一个系统class
            // A system class.
            return ClassLoader.getSystemResource(name);  //如果是系统class
        }
        return cl.getResource(name); //调用ClassLoader的getResource方法
    }
```

查看`ClassLoader.getResource`方法

```java
 public URL getResource(String name) {
        URL url;
        if (parent != null) { //这里的parent为sun.misc.Launcher$ExtClassLoader@7d4793a8
            url = parent.getResource(name); //这里是一个递归调用，再次进入之后parent为null
        } else {
            url = getBootstrapResource(name); //到达系统启动类加载器
        }
        if (url == null) { //系统启动类加载器没有加载到，递归回退到第一次调用然后是扩展类加载器
            url = findResource(name);
        }
        return url; //最后如果都没有加载到，双亲委派加载失败，则加载应用本身自己的加载器
    }
```

再看`resolveName`

```java
private String resolveName(String name) {
        if (name == null) {
            return name;
        }
        if (!name.startsWith("/")) {  //对于不以/开头的文件，
            Class<?> c = this; //获取当前加载类的完整的类路径，
            while (c.isArray()) {
                c = c.getComponentType();
            }
            String baseName = c.getName();
            int index = baseName.lastIndexOf('.');//找到文件的包名称
            if (index != -1) {
                name = baseName.substring(0, index).replace('.', '/')
                    +"/"+name; //将包名称中的.替换为/ 并在最后加上/ 文件名
            }
        } else {
            name = name.substring(1);  //对于/开头的文件名，会只保留文件名称部分。
        }
        return name;
    }
```

如何理解 `name = baseName.substring(0, index).replace('.', '/')
                    +"/"+name;` 与 `name = name.substring(1);`

1. ` baseName.substring(0, index).replace('.', '/')
   
                    +"/"+name;`
   
   举例： 当前启动路径为 `com.hb0730.commons.lang.io.ResourcesUtilsTest`,那么当前baseName则为当前路径,
    ` baseName.substring(0, index).replace('.', '/')
   
                    +"/"+name;` 则为去掉`class`名后的路径得到的文件名路径 `com/hb0730/commons/lang/io/文件路径`
2. `name.substring(1)`则去掉`/`后的资源路径 `sql/test.sql`

然后在通过`ClassLoader`去找到资源

## ClassLoader#getResource

```java
 public URL getResource(String name) {
        URL url;
        if (parent != null) { //这里的parent为sun.misc.Launcher$ExtClassLoader@7d4793a8
            url = parent.getResource(name); //这里是一个递归调用，再次进入之后parent为null
        } else {
            url = getBootstrapResource(name); //到达系统启动类加载器
        }
        if (url == null) { //系统启动类加载器没有加载到，递归回退到第一次调用然后是扩展类加载器
            url = findResource(name);
        }
        return url; //最后如果都没有加载到，双亲委派加载失败，则加载应用本身自己的加载器
    }
```

对于`ClassLoader.getResource`， 直接调用的就是`ClassLoader` 类的`getResource`方法，那么对于`getResource("")`，`path`不以`'/'`开头时，首先通过双亲委派机制，使用的逐级向上委托的形式加载的，最后发现双亲没有加载到文件，最后通过当前类加载`classpath`根下资源文件。对于`getResource("/")`，`'/'`表示`Boot ClassLoader`中的加载范围，因为这个类加载器是C++实现的，所以加载范围为`null`。

## 类加载器ClassLoader

### 1. 类加载器（ClassLoader）

我们都知道 Java 文件被运行，第一步，需要通过 javac 编译器编译为 class 文件；第二步，JVM 运行 class 文件，实现跨平台。而 JVM 虚拟机第一步肯定是 **加载 class 文件**，所以，**类加载器**实现的就是（来自《深入理解Java虚拟机》）：

> 通过一个类的全限定名来获取描述此类的二进制字节流

类加载器有几个重要的特性：

1. 每个类加载器都有自己的预定义的**搜索范围**，用来加载 class 文件；
2. 每个类和加载它的类加载器共同确定了这个类的唯一性，也就是说如果一个 class 文件被**不同的类加载器加载**到了 JVM 中，那么这两个类就是不同的类，虽然他们都来自同一份 class 文件；
3. 双亲委派模型。
   
   ### 2.1 双亲委派模型
4. 所有的类加载器都是有层级结构的，每个类加载器都有一个父类类加载器（通过组合实现，而不是继承），除了**启动类加载器（Bootstrap ClassLoader）**；
5. 当一个类加载器接收到一个类加载请求时，首先将这个请求委派给它的父加载器去加载，所以每个类加载请求最终都会传递到顶层的**启动类加载器**，如果父加载器无法加载时，子类加载器才会去尝试自己去加载；

通过双亲委派模型就实现了类加载器的三个特性：

1. **委派（delegation）**：子类加载器委派给父类加载器加载；
2. **可见性（visibility）**：子类加载器可访问父类加载器加载的类，父类不能访问子类加载器加载的类；
3. **唯一性（uniqueness）**：可保证每个类只被加载一次，比如 Object 类是被 Bootstrap ClassLoader 加载的，因为有了双亲委派模型，所有的 Object 类加载请求都委派到了 Bootstrap ClassLoader，所以保证了只被加载一次。

以上就是类加载器的一些特性，那么在 Java 中类加载器是如何实现的呢？

### 2.2 Java 中的类加载器

从 JVM 虚拟机的角度来看，只存在两种不同的类加载器：

1. 启动类加载器（Bootstrap ClassLoader），是虚拟机自身的一部分；
2. 所有其他的类加载器，独立于虚拟机外部，都继承自抽象类 java.lang.ClassLoader。

而绝大多数 Java 应用都会用到如下 3 中系统提供的类加载器：

1. **启动类加载器（Bootstrap/Primordial/NULL ClassLoader）**：顶层的类加载器，没有父类加载器。负责加载 /lib 目录下的，或则被 -Xbootclasspath 参数所指定路径中的，并被 JVM 识别的（仅按文件名识别，如 rt.jar，名字不符合的类库即使放在 lib 目录也不会被加载）类库加载到虚拟机内存中。所有被 Bootstrap classloader 加载的类，它的 Class.getClassLoader 方法返回的都是 null，所以也称作 NULL ClassLoader。
2. **扩展类加载器（Extension CLassLoader）**：由 sun.misc.Launcher$ExtClassLoader 实现，负责加载 <JAVA_HOME>/lib/ext 目录下，或被 java.ext.dirs 系统变量所指定的目录下的所有类库；
3. **应用程序类加载器（Application/System ClassLoader）**：由 sun.misc.Launcher$AppClassLoader 实现。它是 ClassLoader.getSystemClassLoader() 方法的默认返回值，所以也称为系统类加载器（System ClassLoader）。它负责加载 classpath 下所指定的类库，如果应用程序没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

如下，就是 Java 程序中的类加载器层级结构图：

![20180914180854640](https://hb0730-blog-hk.oss-cn-hongkong.aliyuncs.com/20180914180854640_1618537886667.png)

## ClassPath

java项目中Classpath路径到底指的是哪里？

1. src不是classpath, WEB-INF/classes,lib才是classpath，WEB-INF/ 是资源目录, 客户端不能直接访问。
2. WEB-INF/classes目录存放src目录java文件编译之后的class文件，xml、properties等资源配置文件，这是一个定位资源的入口。
3. 引用classpath路径下的文件，只需在文件名前加classpath:
   
   ```xml
   <param-value>classpath:applicationContext-*.xml</param-value> 
   <!-- 引用其子目录下的文件,如 -->
   <param-value>classpath:context/conf/controller.xml</param-value>
   ```
4. lib和classes同属classpath，两者的访问优先级为: lib>classes。
5. classpath 和 classpath* 区别：
   
   ```java
   classpath：只会到你的class路径中查找找文件;
   classpath*：不仅包含class路径，还包括jar文件中(class路径)进行查找。
   ```

## 参考

* 作者：zhangshk_
  链接：https://blog.csdn.net/zhangshk_/article/details/82704010
* 作者：逍遥不羁
  链接：https://blog.csdn.net/javaloveiphone/article/details/51994268
