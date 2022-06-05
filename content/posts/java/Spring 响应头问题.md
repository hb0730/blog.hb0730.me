---
title:  Spring 响应头问题

date:  2021-12-02 15:51:55

author: hb0730

authorLink: https://blog.hb0730.com

tags: ['Java','Spring Boot','Spring']

categories: ['Java']
---

## 背景

在使用飞书审批**关联外部选项**对接时发现始终保存说结构错误，于是使用postman进行测试发现了果然是结构问题，返回了一个`xml`结构，所以在`@PostMapping`添加了一下`produces`然后问题就解决了，为什么需要去手动添加一下，才会输出`json`格式呢

通过对SpringBoot框架源码调试，最终发现SpringBoot框架是在`AbstractMessageConverterMethodProcessor`类中的`writeWithMessageConverters()`方法中实现判断返回格式的。

```java
@SuppressWarnings({"rawtypes", "unchecked"})
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
  ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
  throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

 Object body;
 Class<?> valueType;
 Type targetType;

 // ... 部分省略代码

 MediaType selectedMediaType = null;
 MediaType contentType = outputMessage.getHeaders().getContentType();
 boolean isContentTypePreset = contentType != null && contentType.isConcrete();
 if (isContentTypePreset) {
  if (logger.isDebugEnabled()) {
   logger.debug("Found 'Content-Type:" + contentType + "' in response");
  }
  selectedMediaType = contentType;
 }
 else {
  HttpServletRequest request = inputMessage.getServletRequest();
  // 获取调用方能接受什么类型的MediaType
  List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
  // 获取服务提供方能产生哪些类型的MediaType
  List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

  if (body != null && producibleTypes.isEmpty()) {
   throw new HttpMessageNotWritableException(
     "No converter found for return value of type: " + valueType);
  }
  // 综合请求方和服务提供方的MediaType情况，计算最终能够返回哪些MediaType
  List<MediaType> mediaTypesToUse = new ArrayList<>();
  for (MediaType requestedType : acceptableTypes) {
   for (MediaType producibleType : producibleTypes) {
    if (requestedType.isCompatibleWith(producibleType)) {
     mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
    }
   }
  }
  if (mediaTypesToUse.isEmpty()) {
   if (body != null) {
    throw new HttpMediaTypeNotAcceptableException(producibleTypes);
   }
   if (logger.isDebugEnabled()) {
    logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
   }
   return;
  }

  // 对所有最终可返回的MediaType进行排序
  MediaType.sortBySpecificityAndQuality(mediaTypesToUse);

  // 计算最终选择返回哪个MediaType，按照先后顺序，只要有一个符合条件，则直接返回，忽略剩余其他的可满足条件的MediaType
  for (MediaType mediaType : mediaTypesToUse) {
   if (mediaType.isConcrete()) {
    selectedMediaType = mediaType;
    break;
   }
   else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
    selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
    break;
   }
  }

  if (logger.isDebugEnabled()) {
   logger.debug("Using '" + selectedMediaType + "', given " +
     acceptableTypes + " and supported " + producibleTypes);
  }
 }

 // ...其他省略代码，包括最终的结果的返回

}
```

`writeWithMessageConverters()`方法主要作用就是把接口返回的结果经过合适的`Converter`处理之后再返回。

这就要求首先判断应该返回什么类型的`MediaType`。

`writeWithMessageConverters()`方法判断使用什么类型的`MediaType`逻辑如下：

首先调用`getAcceptableMediaTypes(request)`判断接收方能接受哪些类型的`MediaType`。

如果没有设置的话，则按照默认的来。默认的`MediaType为MEDIA_TYPE_ALL_LIST`

```java
List<MediaType> MEDIA_TYPE_ALL_LIST = Collections.singletonList(MediaType.ALL);

/**
 * Public constant media type that includes all media ranges (i.e. "&#42;/&#42;").
 */
public static final MediaType ALL;

/**
 * A String equivalent of {@link MediaType#ALL}.
 */
public static final String ALL_VALUE = "*/*";
```

接着调用`getProducibleMediaTypes()`方法来计算当前接口能产生哪些类型的`MediaType`。

```java
/**
 * Returns the media types that can be produced. The resulting media types are:
 * <ul>
 * <li>The producible media types specified in the request mappings, or
 * <li>Media types of configured converters that can write the specific return value, or
 * <li>{@link MediaType#ALL}
 * </ul>
 * @since 4.2
 */
@SuppressWarnings("unchecked")
protected List<MediaType> getProducibleMediaTypes(
  HttpServletRequest request, Class<?> valueClass, @Nullable Type targetType) {

 // 如果request中已经指定了MediaType，则直接使用指定的
 Set<MediaType> mediaTypes =
   (Set<MediaType>) request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
 if (!CollectionUtils.isEmpty(mediaTypes)) {
  return new ArrayList<>(mediaTypes);
 }
 else if (!this.allSupportedMediaTypes.isEmpty()) {
  List<MediaType> result = new ArrayList<>();
  // 依次遍历当前系统中所有的HttpMessageConverte列表，只要能够支持写入指定的targetType，即认为可生成converter支持的MediaType
  for (HttpMessageConverter<?> converter : this.messageConverters) {
   if (converter instanceof GenericHttpMessageConverter && targetType != null) {
    if (((GenericHttpMessageConverter<?>) converter).canWrite(targetType, valueClass, null)) {
     result.addAll(converter.getSupportedMediaTypes());
    }
   }
   else if (converter.canWrite(valueClass, null)) {
    result.addAll(converter.getSupportedMediaTypes());
   }
  }
  return result;
 }
 else {
  return Collections.singletonList(MediaType.ALL);
 }
}
```

`getProducibleMediaTypes()`方法首先判断`request`中有没有指定特定的`MediaType`，如果有的话则直接使用指定的，如果没有的话，则依次遍历当前系统中所有的`HttpMessageConverter`，只要对应`Converter`的`canWrite()`方法返回`true`，则把对应`Converter`所支持的所有的`MediaType`加入返回列表中。

当前系统中支的所有`HttpMessageConverter`列表如下:

![image](https://hb0730-blog-hk.oss-cn-hongkong.aliyuncs.com/image_1638433584574.png)

执行完成发现第一个`canWrite()`返回`true`的`Converter`是`BczRequestConfig$HtmlJsonMessageConverter`。

执行完成之后的 result的结果如下图所示：

![image](https://hb0730-blog-hk.oss-cn-hongkong.aliyuncs.com/image_1638433723559.png)

在计算得到所有可以生成的`MediaType`之后，又会依次判断这些可以生成的`MediaType`是否兼容`acceptableTypes`，由于本次请求中`acceptableTypes`为默认值，则默认兼容。

之后会把上一步中得到的所有的`MediaType`按照各自的`qualityValue(每个MediaType都会有一个值)`进行从小到大排序。

本系统中没有做任何特殊的设置，默认值都是1，所有`MediaType`顺序保持不变。

做完上述操作之后，从上一步中处理之后的所有`MediaType`中选择第一个确定的M`ediaType`(所谓的确定的MediaType是指该MediaType对应的type和subtype都是具体的，不存在通配符的)作为该次请求应该返回的`MediaType`

```java
for (MediaType mediaType : mediaTypesToUse) {
 if (mediaType.isConcrete()) {
  selectedMediaType = mediaType;
  break;
 }
 else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
  selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
  break;
 }
}
```

**位于列表中第一个的`MediaType`为`application/xml;charset=UTF-8`，符合条件，所以`application/xml;charset=UTF-8`即被认为只接口请求最终返回的`MediaType`。**

那么如何去解决呢？

## 解决

既然我们知道 `application/xml;charset=UTF-8`排在第一位，只有将`application/json`排在`application/xml;charset=UTF-8`之前就可以解决其问题，所以我们添加一个`HttpMessageConverter`，令他排在第一位就可以解决

1. ```java
    @Override
    public void extendMessageConverters(List<org.springframework.http.converter.HttpMessageConverter<?>> converters) {
        converters.add(0,new MappingJackson2HttpMessageConverter(mapper));
    }
   ```

2. 可以设置请求头的接受类型`accpet`
   
   ```java
          HttpServletRequest request = inputMessage.getServletRequest();
            List<MediaType> acceptableTypes;
            try {
                //请求的 accept type
                acceptableTypes = getAcceptableMediaTypes(request);
            }
            catch (HttpMediaTypeNotAcceptableException ex) {
                int series = outputMessage.getServletResponse().getStatus() / 100;
                if (body == null || series == 4 || series == 5) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Ignoring error response content (if any). " + ex);
                    }
                    return;
                }
                throw ex;
            }
   ```

## 最后

![Snipaste_2021-12-02_15-38-40](https://hb0730-blog-hk.oss-cn-hongkong.aliyuncs.com/Snipaste_2021-12-02_15-38-40_1638430790325.png)
