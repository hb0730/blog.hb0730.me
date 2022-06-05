---

title: SpringBoot本地依赖jar

date: 2020-09-24 10:30:39

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Spring Boot' ]

categories: [ 'Spring Boot' ]

---

# 原因

因为要使用钉钉的sdk，但是钉钉的sdk只能下载https://ding-doc.dingtalk.com/doc#/faquestions/vzbp02/RL1dn,项目采用的是springboot方式，所以记录手动依赖jar

# pom

```xml
<dependency>
    <groupId>com.dingtalk.open</groupId>
    <artifactId>taobao-sdk-java</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/src/main/resources/lib/taobao-sdk-java-auto_1479188381469-20191014.jar</systemPath>
</dependency>

<plugin>
    <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <includeSystemScope>true</includeSystemScope>
    </configuration>
 </plugin>
```

# jar所在位置

**`src/main/resources/lib`** 即可
