---
title:  Springboot 自定义starter

date:  2019-10-09 17:12:59

author: hb0730

authorLink: https://blog.hb0730.com

tags: ['Java','Spring Boot']

categories: ['Java']

---

## Spring Boot Starter 简介

+ `Spring Boot Starter` 是一组方便使用的依赖关系描述符，可以在应用程序中包含这些描述符。借助 `Spring Boot Starter` 开发人员可以获得所需的所有 `Spring` 及相关技术的一站式服务，而无需查看示例代码或复制粘贴依赖的库文件。譬如，如果需要 `Spring JPA` 访问数据库，则可以在工程中直接饮用 `spring-boot-starter-data-jpa`

+ 有关 starter 命名规范，所有官方发布的 starter 都遵循以下命名模式 `spring-boot-starter-*`，其中 `*` 指特定的应用程序代号或名称。任何第三方提供的 `starter` 都不能以 `spring-boot` 作为前缀，应该将应用程序代号或名称作为前缀，譬如 `mybatis-spring-boot-starter`

## Spring Boot Starter 开发步骤

### 一 引用pom文件

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
    </dependency>
</dependencies>
```

其中 `spring-boot-configuration-processor` 的作用是编译时生成 `spring-configuration-metadata.json` ，此文件主要给IDE使用。如当配置此jar相关配置属性在 `application.yml` ，你可以用ctlr+鼠标左键点击属性名，IDE会跳转到你配置此属性的类中。

我们日常使用的Spring官方的Starter一般采取`spring-boot-starter-{name}` 的命名方式，如 `spring-boot-starter-web` 。

而非官方的Starter，官方建议 `artifactId`命名应遵循`{name}-spring-boot-starter` 的格式。 例如： `weather-spring-boot-starter`

### 二 相关属性的配置properties

```java
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * <p>
 * </P>
 *
 * @author bing_huang
 * @since V1.0
 */
@SuppressWarnings("WeakerAccess")
@ConfigurationProperties(prefix = "weather")
public class WeatherProperties {
    /**
     * <p>
     * 请求地址
     * </p>
     */
    private String url;
    /**
     * <p>
     * 项目id
     * </p>
     */
    private String userId;
    /**
     * 密钥
     */
    private String secretKey;

    /**
     * <p>
     * ip库
     * </p>
     */
    private Ip2region ip2region;

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }

    public String getSecretKey() {
        return secretKey;
    }

    public void setSecretKey(String secretKey) {
        this.secretKey = secretKey;

    }

    public Ip2region getIp2region() {
        return ip2region;
    }

    public void setIp2region(Ip2region ip2region) {
        this.ip2region = ip2region;
    }

    public static class Ip2region {
        /**
         * ip.db 地址
         */
        private String address;

        public String getAddress() {
            return address;
        }

        public void setAddress(String address) {
            this.address = address;
        }
    }

}
```

**备注:**

`@ConfigurationProperties(prefix = "weather")`表示前缀为 `weather`的属性才会注入到 `WeatherProperties`

## 三 自动配置类

```java
/**
 * <p>
 *     starter 配置
 * </P>
 *
 * @author bing_huang
 * @since V1.0
 */
@Configuration
@EnableConfigurationProperties({WeatherProperties.class})
@ConditionalOnClass(WeatherService.class)
@ConditionalOnProperty(prefix = "weather",value = "enabled",havingValue = "true")
public class WeatherAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(WeatherService.class)
    public WeatherService weatherService(WeatherProperties properties) {
        return new WeatherService(properties);
    }
}
```

**备注**:

* `@ConditionalOnClass` 当classpath下发现该类的情况下进行自动配置。

* `@ConditionalOnProperty(prefix = "weather",value = "enabled",havingValue = "true")`当配置文件中weather.enabled=true时才会启用 WeatherAutoConfiguration 

**SpringBoot中的所有@Conditional注解及作用**

> `@ConditionalOnBean`:当容器中有指定的Bean的条件下  
>  `@ConditionalOnClass`：当类路径下有指定的类的条件下  
>  `@ConditionalOnExpression`:基于SpEL表达式作为判断条件  
>  `@ConditionalOnJava`:基于JVM版本作为判断条件  
>  `@ConditionalOnJndi`:在JNDI存在的条件下查找指定的位置  
>  `@ConditionalOnMissingBean`:当容器中没有指定Bean的情况下  
>  `@ConditionalOnMissingClass`:当类路径下没有指定的类的条件下  
>  `@ConditionalOnNotWebApplication`:当前项目不是Web项目的条件下  
>  `@ConditionalOnProperty`:指定的属性是否有指定的值  
>  `@ConditionalOnResource`:类路径下是否有指定的资源  
>  `@ConditionalOnSingleCandidate`:当指定的Bean在容器中只有一个，或者在有多个Bean的情况下，用来指定首选的Bean
>  `@ConditionalOnWebApplication`:当前项目是Web项目的条件下  

### 最后一步，在`resources/META-INF/`下创建`spring.factories`文件，并添加如下内容：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration = \
  com.demo.config.starter.WeatherAutoConfiguration 
```
