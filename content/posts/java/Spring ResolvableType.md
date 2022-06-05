---

title: Spring工具类-ResolvableType

date: 2020-12-23 08:43:35

author: hb0730

authorLink: https://blog.hb0730.com

tags: [ 'Java','Spring Utils' ,'Spring','Spring Boot' ]

categories: [ 'Java' ]

---

# 1.  前言

今天在看 **Spring Security** 源码的时候无意间注意到两行代码;

```java
ResolvableType type = ResolvableType.forClassWithGenerics(OAuth2UserService.class, OAuth2UserRequest.class, OAuth2User.class);

ApplicationContext context = getContext();

String[] names= context.getBeanNamesForType(type);

if (names.length == 1) {
 OAuth2UserService<OAuth2UserRequest, OAuth2User> bean =  (OAuth2UserService<OAuth2UserRequest, OAuth2User>) context.getBean(names[0]);
}
```

`ResolvableType` 居然可以这样获取一个带泛型的类型，于是就研究了一番。

# 2. ResolvableType

通常我们想获取一个类型的信息都要通过 Java 反射从对应的 `Class` 类型中来获取信息，API 非常地底层，操作也十分繁琐。`ResolvableType` 的出现简化了这一过程，文章的开头就是 `ResolvableType` 通过其静态方法来描述了一个带泛型的类型 `OAuth2UserService<OAuth2UserRequest, OAuth2User>`，然后就可以从 **Spring IoC** 中获取对应的 **Spring Bean** 。那么它还有其它那些功能呢？

`ResolvableType` 不能被直接实例化，不过它提供了很多的静态方法。

## forClass

从 `Class` 对象中获取类型的信息，它有一个重载方法可以从基类和实现类中获取组合的类型信息，例如：

```java
//  java.lang.String
ResolvableType resolvableType1 = ResolvableType.forClass(String.class);
//  java.util.Map<?, ?>
ResolvableType resolvableType2 = ResolvableType.forClass(Map.class, HashMap.class);
```

## forClassWithGenerics

从 `Class` 对象中获取类型的信息,并且携带泛型，文章开头的例子就是。

## forConstructorParameter

获取构造函数中特定参数的元信息，例如

```java
User user = new User(“张三”);
 Constructor<?> constructor = user.getClass().getConstructors()[0]
// java.lang.String
ResolvableType.forConstructorParameter(constructor,0)
```

## forField

从成员属性中获取成员属性的类型信息。例如：

```java
private HashMap<String, List<Object>> map;

public void test() throws NoSuchFieldException {
     Field field = getClass().getDeclaredField("map");
     // java.util.HashMap<java.lang.String, java.util.List<java.lang.Object>>
     ResolvableType resolvableType = ResolvableType.forField(field);
}
```

## forMethodParameter

获取方法的参数类型信息。

```java
User user = new User(“张三”);
// setName(String)
Method method = user.getClass().getMethods()[0];
// java.lang.String
ResolvableType.forMethodParameter(method,0)
```

## forMethodReturnType

获取方法返回值的类型信息。

## forArrayComponent

获取特定类型组成的数组形式。例如

```java
ResolvableType resolvableType = ResolvableType.forClass(String.class);
// java.lang.String []
ResolvableType arrayComponent = ResolvableType.forArrayComponent(resolvableType);
```

## forInstance

甚至还可以从对象实例中获取该对象的类型信息。

# 实例方法

> 那么问题来了，这到底有什么用呢？

当你需要利用反射获取 类实例、成员变量、方法的信息时就可以使用该操作类。它提供了获取基类、接口、`Class` 对象、泛型类型等解析功能。例如我们可以从一个成员变量中可以获取：

```java
 private HashMap<Integer, List<String>> myMap;

   public void example() {
       ResolvableType t = ResolvableType.forField(getClass().getDeclaredField("myMap"));
       t.getSuperType(); // AbstractMap<Integer, List<String>>
       t.asMap(); // Map<Integer, List<String>>
       t.getGeneric(0).resolve(); // Integer
       t.getGeneric(1).resolve(); // List
       t.getGeneric(1); // List<String>
       t.resolveGeneric(1, 0); // String
   }
```

感觉可以替代一些常见的反射操作，而且易用性更强。

# Thanks

作者: 码农小胖哥
链接: https://mp.weixin.qq.com/s/yORaBscxoQaii1d_TeUiJQ
来源: 微信公众号
