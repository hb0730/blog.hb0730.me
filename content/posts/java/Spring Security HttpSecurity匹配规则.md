---

title: Spring Security 匹配规则

date: 2020-01-23 14:46:16

author: hb0730

authorLink: https://blog.hb0730.com

tags: ['Spring Security5']

categories: [ 'Spring Security5' ]

---

# SpringSecurity匹配规则

## 一 URL匹配

1. `requestMatchers()` 配置一个`request Mather数组`，参数为`RequestMatcher` 对象，其`match` 规则自定义,需要的时候放在最前面，对需要匹配的的规则进行自定义与过滤

2. `authorizeRequests()` URL权限配置

3. `antMatchers()` 配置一个`request Mather` 的 **string数组**，参数为 `ant` 路径格式， 直接**匹配url**

4. `anyRequest` 匹配任意`url`，无参 ,最好放在最后面

## 二 保护URL

1. `authenticated()` 保护UrL，需要用户登录

2. `permitAll()` 指定URL无需保护，一般应用与静态资源文件

3. `hasRole(String role)` 限制单个角色访问，角色将被增加 **ROLE_** .所以 **ADMIN** 将和 **ROLE_ADMIN** 进行比较. 另一个方法是 `hasAuthority(String authority)`

4. `hasAnyRole(String… roles)` 允许多个角色访问. 另一个方法是`hasAnyAuthority(String… authorities)`

5. `access(String attribute)` 该方法使用 **SPEL**, 所以可以创建复杂的限制 例如如`access("permitAll")`, `access("hasRole('ADMIN') and hasIpAddress('123.123.123.123')")`

6. `hasIpAddress(String ipaddressExpression)` 限制IP地址或子网

## 三 登录login

1. `formLogin()` 基于表单登录

2. `loginPage()` 登录页

3. `defaultSuccessUrl` 登录成功后的默认处理页

4. `failuerHandler`登录失败之后的处理器

5. `successHandler`登录成功之后的处理器

6. `failuerUrl`登录失败之后系统转向的url，默认是 **this.loginPage + "?error"**

## 四 登出logout

1. `logoutUrl` 登出url ， 默认是 **/logout**， 它可以是一个`ant` `path` `url`

2. `logoutSuccessUrl` 登出成功后跳转的 `url` 默认是 **"/login?logout"**

3. `logoutSuccessHandler` 登出成功处理器，设置后会把`logoutSuccessUrl` 置为 `null`



作者：FlyXhc 链接：https://www.jianshu.com/p/e3a9a8c4876c 来源：简书
