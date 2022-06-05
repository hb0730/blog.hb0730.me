---
title: Spring Security5 资源服务器

date: 2020-09-29 14:12:33

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Spring Security5' ]

categories: [ 'Spring Security5', 'Spring Ouath2' ]

---

> 本项目采用spring-boot的方式：spring-boot版本为:2.3.3.RELEASE 
> 
> 本文主要是想提供一个401解决思路
> 
> 源码地址： [security5-resource](https://github.com/hb0730/spring-security-oauth2-example/tree/master/security5-resource)
> 
> 博客地址: https://blog.hb0730.com/archives/springsecurity5%E4%B9%8Bresourceserver%E8%B5%84%E6%BA%90%E6%9C%8D%E5%8A%A1%E5%99%A8

# authorizationServer 认证服务器

## pom 依赖

```xml
<dependencies>
    <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security.oauth</groupId>
            <artifactId>spring-security-oauth2</artifactId>
            <version>2.3.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
</dependencies>
```

## configuration 配置

**authorization Server授权服务器**

```java
/**
 * 授权服务器配置
 *
 * @author bing_huang
 */
@Configuration
@EnableAuthorizationServer
@RequiredArgsConstructor
public class AuthorizationServerConfiguration extends AuthorizationServerConfigurerAdapter {
    private final PasswordEncoder passwordEncoder;
    private final UserDetailsService userDetailsService;
    /**
     * 来支持 password grant type
     */
    private final AuthenticationManager authenticationManager;

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        // 允许客户端认证ClientCredentialsTokenEndpointFilter
        security.allowFormAuthenticationForClients()
                .checkTokenAccess("permitAll()")
                .tokenKeyAccess("permitAll()");
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        // 内存模式
        clients.inMemory()
                //客户端id
                .withClient("client")
                // 客户端secret
                .secret(passwordEncoder.encode("secret"))
                //范围
                .scopes("all")
                //权限
                .authorities("read", "writer")
                //授权类型
                .authorizedGrantTypes("password", "refresh_token");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        // 允许get,post请求token端点
        endpoints.allowedTokenEndpointRequestMethods(HttpMethod.POST, HttpMethod.GET)
                // token 存储服务
                .tokenServices(defaultTokenServices())
                //user 服务
                .userDetailsService(userDetailsService)
                //password grant type
                .authenticationManager(authenticationManager);
    }

    /**
     * <p>注意，自定义TokenServices的时候，需要设置@Primary，否则报错，</p>
     * 自定义的token
     * 认证的token是存到redis里的
     *
     * @return DefaultTokenServices
     */
    @Primary
    @Bean
    public DefaultTokenServices defaultTokenServices() {
        DefaultTokenServices tokenServices = new DefaultTokenServices();
        tokenServices.setTokenStore(tokenStore());
        tokenServices.setSupportRefreshToken(true);
        return tokenServices;
    }

    /**
     * redis存储令牌
     *
     * @return token存储
     */
    @Bean
    public TokenStore tokenStore() {
        return new InMemoryTokenStore();
    }

}
```

**web Security配置**

```java
/**
 * web Security
 *
 * @author bing_huang
 */
@EnableWebSecurity
@Configuration
@RequiredArgsConstructor
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(this.userDetailsService())
                .passwordEncoder(passwordEncoder());
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                //禁用csrf
                .csrf().disable()
                //禁用session
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                //放开端口
                .antMatchers("/oauth/**", "/actuator/**").permitAll()
                //其余拦截
                .anyRequest().authenticated()
                .and()
                // 允许basic认证
                .httpBasic()
        ;

    }

    @Override
    @Bean
    protected UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        userDetailsManager.createUser(
                User.withUsername("user")
                        .password(passwordEncoder().encode("123456"))
                        .authorities("ROLE_USEr").build());
        return userDetailsManager;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

# resource Server资源服务器

> 这里的资源服务器采用security5方式使用内省方式具体可以看官方demo:[oauth2resourceserver-opaque](https://github.com/spring-projects/spring-security/tree/master/samples/boot/oauth2resourceserver-opaque)

## pom

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-oauth2-resource-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>com.nimbusds</groupId>
            <artifactId>oauth2-oidc-sdk</artifactId>
        </dependency>
    </dependencies>
```

## configuration 配置

```java
/**
 * 资源服务器
 *
 * @author bing_huang
 */
@EnableWebSecurity
@Configuration
@RequiredArgsConstructor
public class WebSecurityServerConfiguration extends WebSecurityConfigurerAdapter {
    private final OAuth2ResourceServerProperties properties;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .csrf().disable()
                .cors();
        http.oauth2ResourceServer()
                .opaqueToken()
                .introspectionClientCredentials(
                        properties.getOpaquetoken().getClientId(), 
                        properties.getOpaquetoken().getClientSecret())
                .introspectionUri(properties.getOpaquetoken().getIntrospectionUri());
    }


    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

## yaml配置

```yaml
server:
  port: 8082
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          client-id: client
          client-secret: secret
          introspection-uri: http://localhost:8081/introspect
```

所以我们在认证服务器添加`introspect`端点

## introspect 内省端点

```java
/**
 * @author bing_huang
 */
@FrameworkEndpoint
@RequiredArgsConstructor
public class IntrospectEndpoint {
    private final TokenStore tokenStore;

    @PostMapping("/introspect")
    @ResponseBody
    public Map<String, Object> introspect(@RequestParam("token") String token) {
        OAuth2AccessToken accessToken = this.tokenStore.readAccessToken(token);
        Map<String, Object> attributes = new HashMap<>();
        if (accessToken == null || accessToken.isExpired()) {
            attributes.put("active", false);
            return attributes;
        }

        OAuth2Authentication authentication = this.tokenStore.readAuthentication(token);

        attributes.put("active", true);
        attributes.put("exp", accessToken.getExpiration().getTime());
        attributes.put("scope", accessToken.getScope().stream().collect(Collectors.joining(" ")));
        attributes.put("sub", authentication.getName());

        return attributes;
    }
}
```

# 访问

## 1. 获取access_token

**注意这里采用的是自定义端点来获取access_token**

1. 认证服务器自定义端点
   
   ```java
   /**
    * 自定义Oauth2自定义返回格式(json)
    *
    * @author bing_huang
    */
   @RestController
   @RequestMapping("/oauth")
   @RequiredArgsConstructor
   public class OauthController {
       private final TokenEndpoint tokenEndpoint;
       private final TokenStore tokenStore;
   
       /**
        * get登录
        */
       @GetMapping("/token")
       public LinkedHashMap<String, Object> getAccessToken(Principal principal, @RequestParam Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
           return custom(tokenEndpoint.getAccessToken(principal, parameters).getBody());
       }
   
       /**
        * post登录
        */
       @PostMapping("/token")
       public LinkedHashMap<String, Object> postAccessToken(Principal principal, @RequestParam Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
           return custom(tokenEndpoint.postAccessToken(principal, parameters).getBody());
       }
   
       private LinkedHashMap<String, Object> custom(OAuth2AccessToken accessToken) {
           DefaultOAuth2AccessToken token = (DefaultOAuth2AccessToken) accessToken;
           LinkedHashMap<String, Object> data = new LinkedHashMap<>(token.getAdditionalInformation());
           data.put("accessToken", token.getValue());
           if (token.getRefreshToken() != null) {
               data.put("refreshToken", token.getRefreshToken());
           }
           return data;
       }
   }
   ```
   
   <img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-01.png">

## 2. 通过access_token访问资源服务器

**请求头必须带有Authorization:Bearer 974f4f99-c7ef-4924-af90-4563cc388a72**

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-02.png">

发现没有任何的错误信息: 修改日志级别

```yaml
logging:
  level:
    org:
      springframework:
        security: DEBUG
```

在此请求方法日志会有报错信息为401
<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-03.png">

为何我们用`access_token`去访问资源服务会报401无权限呢

# 401 无权限访问解决

## 1. 内省`opaqueToken`是如何访问的？？

查看spring官网[oauth2resourceserver-opaque-architecture](https://docs.spring.io/spring-security/site/docs/5.4.0/reference/html5/#oauth2resourceserver-opaque-architecture) 内省模式的工作方式

当一个请求发生:`BearerTokenAuthenticationFilter`将认证信息`BearerTokenAuthenticationToken`传递给`AuthenticationManager`,由`ProviderManager`找到具体的`OpaqueTokenAuthenticationProvider`,而`OpaqueTokenAuthenticationProvider`通过`OpaqueTokenIntrospector`换取认证信息

查看`OpaqueTokenIntrospector`只有一个`NimbusOpaqueTokenIntrospector`的实现,其内部通过`RestTemplate`请求认证服务器
<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-04.png">

而`NimbusOpaqueTokenIntrospector`的请求信息则是: 请求头是`basic`认证的客户端信息,参数则是`token`

这样一来我们就能确认认证服务filter存在问题

## 2. 认证服务器的filter 顺序

 在认证服务器配置`AuthorizationServerConfiguration`中我们已经放开的客户端认证`security.allowFormAuthenticationForClients()`，client认证所对应的filter又是什么

在阅读源码源码得知为`ClientCredentialsTokenEndpointFilter`

源码
`org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerSecurityConfigurer`

```java
@Override
    public void configure(HttpSecurity http) throws Exception {

        // ensure this is initialized
        frameworkEndpointHandlerMapping();
        if (allowFormAuthenticationForClients) {
            clientCredentialsTokenEndpointFilter(http);
        }

        for (Filter filter : tokenEndpointAuthenticationFilters) {
            http.addFilterBefore(filter, BasicAuthenticationFilter.class);
        }

        http.exceptionHandling().accessDeniedHandler(accessDeniedHandler);
    }

private ClientCredentialsTokenEndpointFilter clientCredentialsTokenEndpointFilter(HttpSecurity http) {
        ClientCredentialsTokenEndpointFilter clientCredentialsTokenEndpointFilter = new ClientCredentialsTokenEndpointFilter(
                frameworkEndpointHandlerMapping().getServletPath("/oauth/token"));
        clientCredentialsTokenEndpointFilter
                .setAuthenticationManager(http.getSharedObject(AuthenticationManager.class));
        OAuth2AuthenticationEntryPoint authenticationEntryPoint = new OAuth2AuthenticationEntryPoint();
        authenticationEntryPoint.setTypeName("Form");
        authenticationEntryPoint.setRealmName(realm);
        clientCredentialsTokenEndpointFilter.setAuthenticationEntryPoint(authenticationEntryPoint);
        clientCredentialsTokenEndpointFilter = postProcess(clientCredentialsTokenEndpointFilter);
        http.addFilterBefore(clientCredentialsTokenEndpointFilter, BasicAuthenticationFilter.class);
        return clientCredentialsTokenEndpointFilter;
    }
```

当我们再次请求并且debug在`FilterChainProxy.doFilterInternal`查看`filters`是顺序以及个数和是否存在`ClientCredentialsTokenEndpointFilter`

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-05.png">

发现并没有`clientCredentialsTokenEndpointFilter`

## 消失的`clientCredentialsTokenEndpointFilter`

在`FilterChainProxy.getFilters`中我们发现`filterChains`有两个

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-06.png">

而满足我们的其中的一个,如何确定url呢，在spring oauth2当中提供了获取token的一个远程调用类`RemoteTokenServices`

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-07.png">

和我们的请求类似:其认证端点也是一个可配置化的一般为`/oauth/check_token`

所以我们将**资源服务器的yaml请求改为/oauth/check_token***后再一次请求

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-08.png">

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-09.png">

# filter的生成

`FilterChainProxy`filterChain代理每一个请求都会经过，这里也存储了每一个filter

`filterChainProxy`的生成:

FilterChainProxy的生成是通过`WebSecurity#performBuild() new`

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-10.png">

在`#performBuild`方法中还存在一个默认`DefaultSecurityFilterChain`这个就是`WebSecurity`配置了`ignoredRequests`
而另一个是是一个集合`FilterChain>> securityFilterChainBuilders = new ArrayList<SecurityBuilder<? extends SecurityFilterChain>>()`

`securityFilterChainBuilders`的生成是通过`add`添加进入的,也就是通过`WebSecurity#addSecurityFilterChainBuilder`添加进入.所以当一个应用有几个`webSecurity`应该就有几个`SecurityFilterChain`

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-11.png">

# SecurityBuilder

结构:

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-12.png">

如何找到具体的实现的类呢:
我们通过`securityFilterChainBuilders`往上查找

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-13.png">

可以得知他的实现类则是`HttpSecurity`,所以我们一步步的build进入`HttpSecurity`

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-14.png">

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-15.png">

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-16.png">

进入`httpSecurity`类

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/security5/resourceserver/security5-17.png">

```java
@Override
    protected DefaultSecurityFilterChain performBuild() {
        filters.sort(comparator);
        return new DefaultSecurityFilterChain(requestMatcher, filters);
    }
```

就可以查看有哪些filter添加
