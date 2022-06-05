---

title: Spring security 访问控制表达式

date: 2020-03-21 19:24:07

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Spring Security5' ]

categories: [ 'Spring Security5' ]

---

## 常见的表达式

Spring Security可用表达式对象的基类是SecurityExpressionRoot。

| 表达                                                                     | 描述                                                                                            |
| ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `hasRole(String role)`                                                 | 用户拥有制定的角色时返回`true` （`Spring security`默认会带有ROLE_前缀）,去除参考Remove the ROLE_                       |
| `hasAnyRole(String…​ roles)`                                           | 用户拥有任意一个制定的角色时返回`true`                                                                        |
| `hasAuthority(String authority)`                                       | 等同于    `hasRole`,但不会带有ROLE_前缀                                                                 |
| `hasAnyAuthority(String…​ authorities)`                                | 等同于`hasAnyRole`                                                                               |
| `principal`                                                            | 代表当前用户的`principle`对象                                                                          |
| `authentication`                                                       | 直接从`SecurityContext`获取的当前`Authentication`对象                                                   |
| `permitAll`                                                            | 总是返回`true`，表示允许所有的                                                                            |
| `denyAll`                                                              | 总是返回`false`，表示拒绝所有的                                                                           |
| `isAnonymous()`                                                        | 当前用户是否是一个匿名用户                                                                                 |
| `isRememberMe()`                                                       | 表示当前用户是否是通过Remember-Me自动登录的                                                                   |
| `isAuthenticated()`                                                    | 表示当前用户是否已经登录认证成功了。                                                                            |
| `isFullyAuthenticated()`                                               | 如果当前用户既不是一个匿名用户，同时又不是通过Remember-Me自动登录的，则返回true。                                              |
| `hasPermission(Object target, Object permission)`                      | 返回true用户是否可以访问给定权限的给定目标。例如，<br><br>`hasPermission(domainObject, 'read')`                      |
| `hasPermission(Object targetId, String targetType, Object permission)` | 返回`true`用户是否可以访问给定权限的给定目标。例如，<br><br>`hasPermission(1, 'com.example.domain.Message', 'read')` |

## Method安全表达式

针对方法级别的访问控制比较复杂，`Spring Security`提供了四种注解，分别是`@PreAuthorize` , `@PreFilter` , `@PostAuthorize` 和 `@PostFilter`

### 使用method注解

1. 开启方法级别注解的配置
   
   ```java
   @Configuration
   @EnableGlobalMethodSecurity(prePostEnabled = true)
   public class webSecurityConfig extends WebSecurityConfigurerAdapter
   ```

2. 配置相应的bean
   
   ```java
   @Bean
   @Override
   public AuthenticationManager authenticationManagerBean() throws Exception {
     return super.authenticationManagerBean();
   }
   
   @Override
   protected void configure(AuthenticationManagerBuilder auth) throws Exception {
     auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
   }
   
   @Bean
   @ConditionalOnMissingBean(PasswordEncoder.class)
   public PasswordEncoder passwordEncoder(){
     return new BCryptPasswordEncoder();
   }
   ```

3. 在方法上面使用注解
   ```java
   /**
* 查询所有人员 
  */ 
  @PreAuthorize(“hasRole(‘ADMIN’)”) 
  @ApiOperation(value = “获得person列表”, notes = “”) 
  @GetMapping(value = “/persons”) 
  public List getPersons() { 
    return personService.findAll(); 
  } 
  
  ```
  
  ```

## PreAuthorize

`@PreAuthorize` 注解适合进入方法前的权限验证

```java
@PreAuthorize("hasRole('ADMIN')")
    List<Person> findAll();
```

## PostAuthorize

`@PostAuthorize` 在方法执行后再进行权限验证,适合验证带有返回值的权限。`Spring EL` 提供 返回对象能够在表达式语言中获取返回的对象`return Object`

```java
@PostAuthorize("returnObject.name == authentication.name")
    Person findOne(Integer id);
```

## PreFilter

 `PreFilter` 针对参数进行过滤

```java
//当有多个对象是使用filterTarget进行标注
@PreFilter(filterTarget="ids", value="filterObject%2==0")
public void delete(List<Integer> ids, List<String> usernames) {
   ...

}
```

## PostFilter

`PostFilter` 针对返回结果进行过滤

```java
@PreAuthorize("hasRole('ADMIN')")
 @PostFilter("filterObject.name == authentication.name")
 List<Person> findAll();
```

参考

作者: 小致Daddy  链接：[Spring Security 基于表达式的权限控制](https://my.oschina.net/liuyuantao/blog/1924776) 来源：oschina

作者: spring官网 链接：[基于表达式的访问控制](https://docs.spring.io/spring-security/site/docs/5.3.0.RELEASE/reference/html5/#el-access) 来源：spring.io
