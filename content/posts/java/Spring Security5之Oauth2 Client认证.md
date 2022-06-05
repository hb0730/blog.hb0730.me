---

title: Spring Security5 Oauth2 Client认证

date: 2020-10-16 15:13:53

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Spring Security5' ]

categories: [ 'Spring Security5', ' Spring Oauth2' ]

---

# Spring Security5之Oauth2 Client认证

> 本项目采用spring-boot的方式：spring-boot版本为:2.3.3.RELEASE 
> 
> 源码地址： [oauth2-login-multivariate](https://github.com/hb0730/spring-security-oauth2-example/tree/master/oauth2-login-multivariate)
> 
> 博客地址: https://blog.hb0730.com

# 1. 搭建授权服务器Authorization Server

> Authorization Server采用的还是spring Security oauth提供的
> 
> 提供端口为：8081

## pom.xml

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security.oauth</groupId>
            <artifactId>spring-security-oauth2</artifactId>
            <version>2.3.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
```

## Config配置

* AuthorizationServerConfiguration

> 主要提供了两个客户端 一个采用security5 Oauth2Client方式访问，一个通过传统的spring security Oauth SSO方式访问
> 
> 采用的html方式进行登录,client 登录默认使用**授权码**方式，
> 
> Security5 Oauth2Login登录核心关注**redirectUris**重定向的路径问题
> 
> 项目目录: <img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/authorizationServer-01.png">

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
                .checkTokenAccess("isAuthenticated()")
                .tokenKeyAccess("permitAll()");
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        // 内存模式
        clients.inMemory()
                //客户端id
                .withClient("client1")
                // 客户端secret
                .secret(passwordEncoder.encode("secret"))
                //范围
                .scopes("all")
                //权限
                .authorities("read", "writer")
                //授权类型
                .authorizedGrantTypes("password", "refresh_token", "authorization_code")
                .redirectUris("http://localhost:8085/client1")

                .and()
                .withClient("client2")
                .secret(passwordEncoder.encode("secret"))
                .scopes("all")
                .authorities("read", "writer")
                .authorizedGrantTypes("password", "refresh_token", "authorization_code")
                .redirectUris("http://localhost:8086/login/oauth2/code/client2");
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

* webSecurityConfig

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
        http.formLogin().loginPage("/login")
                .and()
                .authorizeRequests()
                .antMatchers("/login").permitAll()
                .anyRequest().authenticated()
                .and()
                .csrf().disable().cors();
        // 这里并没有禁用session，page 跳转使用的session
        ;

    }

    @Override
    @Bean
    protected UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        userDetailsManager.createUser(
                User.withUsername("user")
                        .password(passwordEncoder().encode("123456"))
                        .authorities("ROLE_USER").build());
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

## Controller

> 主要就是返回视图

```java
/**
 * @author bing_huang
 */
@Controller
@RequestMapping
public class LoginController {

    @GetMapping("/login")
    public String login() {
        return "login";
    }

}
```

# Spring Security Oauth之SSO

> 项目目录: <img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/sso-05.png">
> 
> 端口： 8085

## pom.xml

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security.oauth</groupId>
            <artifactId>spring-security-oauth2</artifactId>
            <version>2.3.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.security.oauth.boot</groupId>
            <artifactId>spring-security-oauth2-autoconfigure</artifactId>
            <version>2.3.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-oauth2-resource-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
```

比**Authorication-server**多出一个`spring-security-oauth2-autoconfigure`

## config

* WebSecurityConfig
  
  ```java
  /**
  * @author bing_huang
  */
  @Configuration
  @EnableOAuth2Sso
  public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.logout().logoutSuccessUrl("http://localhost:8081/logout");
        http.authorizeRequests().anyRequest().authenticated();
        http.csrf().disable();
    }
  }
  ```

## yaml配置

```
server:
  port: 8085
  servlet:
    session:
      cookie:
        # Possible CSRF detected - state parameter was required but no state could be
        name: resource1
security:
  oauth2:
    client:
      client-id: client1
      client-secret: secret
      access-token-uri: http://localhost:8081/oauth/token
      user-authorization-uri: http://localhost:8081/oauth/authorize
      pre-established-redirect-uri: http://localhost:8085/client1
      use-current-uri: false
    resource:
      token-info-uri: http://localhost:8081/oauth/check_token
    sso:
      login-path: /client1

logging:
  level:
    root: WARN
    org.springframework.web: DEBUG
    org.springframework.security: DEBUG
```

**说明:**

1. session，应该使用的同一个路径导致回冲掉前一个缓存信息，所以重命名cookie，异常信息 Possible CSRF detected - state parameter was required but no state could be
2. 认证后响应路径问题，默认路径为`/login`,这里自定义路径

# 访问

当使用浏览器访问http://localhost:8085/ 时会自动跳转到 http://localhost:8081
<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/sso-01.png">

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/sso-02.png">

然后输入用户名密码`user/123456` 登录授权，值得我们注意的是地址栏的变化
<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/sso-03.png">

是`授权码`模式登录的，授权完成后又会跳回 http://localhost:8085

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/sso-04.png">

这里就已经完成了sso登录，sso登录采用的是client`授权码`方式

# 如果我们使用 security5 推荐的方式呢

## security-client

> 目录: <img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/security5-oauth2login-01.png">
> 
> 端口: 8086

### config配置

* WebSecurityConfig
  
  ```java
  /**
   * @author bing_huang
   */
  @EnableWebSecurity
  @Configuration
  public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
      @Override
      protected void configure(HttpSecurity http) throws Exception {
          http
                  .authorizeRequests()
                  .anyRequest().authenticated()
                  .and()
                  .csrf().disable()
                  .cors();
  
          http.oauth2Login();
          http.oauth2Client();
      }
  
      @Bean
      public PasswordEncoder passwordEncoder() {
          return new BCryptPasswordEncoder();
      }
  }
  ```

### yaml配置

```yaml
server:
  port: 8086
  servlet:
    session:
      cookie:
        name: resource2
spring:
  security:
    oauth2:
      client:
        registration:
          client2:
            provider: client2
            client-id: client2
            client-secret: secret
            authorization-grant-type: authorization_code
            redirectUri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          client2:
            authorization-uri: http://localhost:8081/oauth/authorize
            token-uri: http://localhost:8081/oauth/token

logging:
  level:
    root: WARN
    org.springframework.web: DEBUG
    org.springframework.security: DEBUG
```

### 访问

配置完成后启动项目访问 http://localhost:8086
<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/security5-oauth2login-02.png">

地址栏: http://localhost:8081/oauth/authorize?response_type=code&client_id=client2&state=ym9ynGfFaLlMdFyJC1sBkakT9C2ttaUVwcAcdJt1BIY%3D&redirect_uri=http://localhost:8086/login/oauth2/code/client2

授权登录
<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/security5-oauth2login-03.png">

发现有一个必要配置`UserInfoEndpoint` `user_info_uri`,按照**sso**方式我们发现还差一个`http://localhost:8081/oauth/check_token`,修改yaml

```yaml
server:
  port: 8086
  servlet:
    session:
      cookie:
        name: resource2
spring:
  security:
    oauth2:
      client:
        registration:
          client2:
            provider: client2
            client-id: client2
            client-secret: secret
            authorization-grant-type: authorization_code
            redirectUri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          client2:
            authorization-uri: http://localhost:8081/oauth/authorize
            token-uri: http://localhost:8081/oauth/token
            user-info-uri: http://localhost:8081/oauth/check_token

logging:
  level:
    root: WARN
    org.springframework.web: DEBUG
    org.springframework.security: DEBUG
```

再一次访问，发现还有有问题

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/security5-oauth2login-04.png">

还有一个`user-name-attribute`

```yaml
server:
  port: 8086
  servlet:
    session:
      cookie:
        name: resource2
spring:
  security:
    oauth2:
      client:
        registration:
          client2:
            provider: client2
            client-id: client2
            client-secret: secret
            authorization-grant-type: authorization_code
            redirectUri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          client2:
            authorization-uri: http://localhost:8081/oauth/authorize
            token-uri: http://localhost:8081/oauth/token
            user-info-uri: http://localhost:8081/oauth/check_token
            user-name-attribute: name

logging:
  level:
    root: WARN
    org.springframework.web: DEBUG
    org.springframework.security: DEBUG
```

访问发现是未经授权**401**

<img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/security5-oauth2login-05.png">

### security5 client 是如何获取access_token的，是通过那个filter换取access_token

1. 查看错误日志发现
   
   <img src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/security5-oauth2login-07.png">

发现其中的类: `DefaultOAuth2UserService` ,`OAuth2LoginAuthenticationProvider` ,`OAuth2LoginAuthenticationFilter`
`OAuth2AuthorizationRequestRedirectFilter`,

2. 查看 `DefaultOAuth2UserService#loadUser`, `OAuth2UserRequestEntityConverter`使用的是`BearerAuth`,而只有是资源服务时如**SSo EnableResourceServer**启用的是资源服务以及**securiy5 Resources**[springsecurity5之resourceserver资源服务器](https://blog.hb0730.com/archives/springsecurity5%E4%B9%8Bresourceserver%E8%B5%84%E6%BA%90%E6%9C%8D%E5%8A%A1%E5%99%A8)，才是通过**BearerAuthen认证**其余都是**Basic认证**

**补充**

* BearerAuth -----> BearerTokenAuthenticationFilter   resource-server
* BasicAuth ------> BasicAuthenticationFilter        security-web （任何）

## Security5 resources server

[springsecurity5之resourceserver资源服务器](https://blog.hb0730.com/archives/springsecurity5%E4%B9%8Bresourceserver%E8%B5%84%E6%BA%90%E6%9C%8D%E5%8A%A1%E5%99%A8)

新增一个controller 用于 `user-info-uri`

```java
/**
 * 为了支持oauth2Login(), 为何不用oauth/check_token ，
 * /oauth/check_token 是basic认证
 *
 * @author bing_huang
 */
@RestController
public class UserController {
    @RequestMapping("/user/me")
    public Map<String, Object> user(Principal principal) {
        // 为何转为map
        // 当security5 Client请求过来时流程
        // Oauth2LoginAuthenticationFilter#attemptAuthentication 184
        // OAuth2LoginAuthenticationProvider#authenticate 110
        // DefaultOAuth2Service#loadUser
        // DefaultOAuth2User
        BearerTokenAuthentication authentication = (BearerTokenAuthentication) SecurityContextHolder.getContext().getAuthentication();
        Map<String, Object> map = new HashMap<>();
        map.put("authorities", authentication.getAuthorities());
        map.put("details", authentication.getDetails());
        map.put("authenticated", authentication.isAuthenticated());
        map.put("principal", authentication.getPrincipal());
        map.put("credentials", authentication.getCredentials());
        map.put("token", authentication.getToken());
        map.put("tokenAttributes", authentication.getTokenAttributes());
        map.put("name", authentication.getTokenAttributes().get("user_name"));
        return map;
    }
}
```

# 最终

## Security5 client

```yaml
server:
  port: 8086
  servlet:
    session:
      cookie:
        name: resource2
spring:
  security:
    oauth2:
      client:
        registration:
          client2:
            provider: client2
            client-id: client2
            client-secret: secret
            authorization-grant-type: authorization_code
            redirectUri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          client2:
            authorization-uri: http://localhost:8081/oauth/authorize
            token-uri: http://localhost:8081/oauth/token
#            user-info-uri: http://localhost:8081/oauth/check_token
            # 为何不用/oauth/check_token ,原因是/oauth/check_token是basic认证
            # DefaultOAuth2UserService#loadUser
            # Bearer 认证 Security5 resources BearerTokenAuthenticationFilter
            # Basic 认证 Security Basic BasicAuthenticationFilter
            user-info-uri: http://localhost:8082/user/me
            userInfoAuthenticationMethod: post
            user-name-attribute: name

logging:
  level:
    root: WARN
    org.springframework.web: DEBUG
    org.springframework.security: DEBUG
```

**注意**

* **user-name-attribute** 对应在资源服务`user-info-uri`的**name**
  具体可以查看源码`OAuth2LoginAuthenticationFilter#attemptAuthentication`第195行

## Security5 Client5主要filter



<img title="" src="https://raw.githubusercontent.com/hb0730/spring-security-oauth2-example/master/doc/oauth2-login-multivariate/security5-oauth2login-06.png" alt="" width="825">


