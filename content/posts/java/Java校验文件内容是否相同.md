---
title: "Java如何校验两个文件内容是相同的？"
date: 2021-12-01 08:43:26
draft: false
author: "hb0730"
authorLink: "https://blog.hb0730.com"
description: "Java校验文件内容是否相同"
images: []
resources:

tags: ["Java"]
categories: ["Java"]

twemoji: false
lightgallery: true

---

今天做文件上传功能，需求要求文件内容相同的不能重复上传。感觉这个需求挺简单的就交给了一位刚入行的新同学。等合并代码的时候发现这位同学居然用文件名称相同和文件大小相同作为两个文件相同的依据。这种条件判断靠谱吗？

从概率上来说遇到两个文件名称和大小都一样的概率确实太小了。这种判断放在生产环境中也可以稳定的跑上一阵子，不过即使再低的可能性也是有可能的，如果能做到100%就好了。

## 文件摘要校验

我相信同学们都下载过一些好心人开发的小工具，有些小工具会附带一个校验器让你校验附带提供的checksum值，防止有人恶意篡改小工具，保证小工具可以放心使用。

![640](https://hb0730-blog-hk.oss-cn-hongkong.aliyuncs.com/640_1638319050125.png)

如果两个文件的内容相同，那么它们的摘要应该是相同的。这个原理能不能帮助我们鉴定两个文件是否相同呢？

## Java实现文件摘要

带着这个疑问，我写了一个文件摘要提取工具类：

```java
  /**
     * 提取文件 checksum 
     *
     * @param path      文件全路径
     * @param algorithm  算法名 例如 MD5、SHA-1、SHA-256等
     * @return  checksum
     * @throws NoSuchAlgorithmException the no such algorithm exception
     * @throws IOException              the io exception
     */
    public static String extractChecksum(String path, String algorithm) throws NoSuchAlgorithmException, IOException {
        // 根据算法名称初始化摘要算法
        MessageDigest digest = MessageDigest.getInstance(algorithm);
        // 读取文件的所有比特
        byte[] fileBytes = Files.readAllBytes(Paths.get(path));
        // 摘要更新
        digest.update(fileBytes);
        //完成哈希摘要计算并返回特征值
        byte[] digested = digest.digest();
        // 进行十六进制的输出
        return HexUtils.toHexString(digested);
    }
```

接下来做几组对照试验来证明猜想。

### 内容不变

首先要证明一个文件在内容不变的情况下摘要是否有变化，多次执行下面的代码，断言始终都是`true`。

```java
String path = "C:\\Users\\s1\\IdeaProjects\\demo\\src\\main\\resources\\application.yml";
String checksum = extractChecksum(path, "SHA-1");
String hash = "6bf4d6c101b4a7821226d3ec1f8d778a531bf265";
Assertions.assertEquals(hash,checksum);
```

而且我把文件名改成`application-dev.yml`，甚至`application-dev.txt`摘要都是相同的。我又把`yml`文件的内容作了改动，断言就`false`了。这证明了单个文件的情况下，内容不变，`hash`是不变的。

### 文件复制

我把`yml`文件复制了一份，改了文件名称和类型，不改变内容并存到了另一个目录中，来测试一下它们的摘要是否有变化。

```java
String path1 = "C:\\Users\\s1\\IdeaProjects\\demo\\src\\main\\resources\\application.yml";
String path2 = "C:\\Users\\s1\\IdeaProjects\\demo\\src\\main\\resources\\templates\\application-dev.txt";
String checksum1 = extractChecksum(path1, "SHA-1");
String checksum2 = extractChecksum(path2, "SHA-1");
String hash = "6bf4d6c101b4a7821226d3ec1f8d778a531bf265";
Assertions.assertEquals(hash,checksum1);
Assertions.assertEquals(hash,checksum2);
```

结果断言通过，不过改变了其中一个文件的内容后断言就不通过了。

### 新建空文件

> 这里的新建空文件指的是没有进行任何操作的新建的空文件。

新建的空文件会根据特定的算法返回一个固定值，比如`SHA-1`算法下的空文件值是:

```
da39a3ee5e6b4b0d3255bfef95601890afd80709
```

### 结论

通过实验证明了：

+ 在相同算法下，任何新建空文件的摘要值都是固定的。
+ 任何两个内容相同的文件的摘要值都是相同的，和路径、文件名、文件类型无关。
+ 文件的摘要值会随着文件内容的改变而改变。

## 文件摘要运用

根据上面的结论，文件摘要是可以防止同样内容的文件重复提交的， 存储的时候不但要存储文件的路径，还要存储文件的摘要值，可能需要注意新建空文件的的固定摘要问题。另外在[Java12中提供了新的API来处理文件内容重复问题](http://mp.weixin.qq.com/s?__biz=MzUzMzQ2MDIyMA==&mid=2247491953&idx=1&sn=007eab95860415e40c77cb9986224510&chksm=faa104e2cdd68df48fa505fec4c615e98f8561fbe6ffbbf185d0f2a7c2755b56f41abc51ffb7&scene=21#wechat_redirect)，有兴趣的可以研究一下。文件摘要除了防篡改和去重之外，你知道还有其它什么用途吗？欢迎同学们留言讨论。

作者：码农小胖哥
链接：https://mp.weixin.qq.com/s/hOQ-AuyYrhMjHgdaDx3JGg