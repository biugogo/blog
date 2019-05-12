---
title: Spring RestTemplate 常用构造方式
date: 2018-3-25 18:14:10
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Spring RestTemplate 常用构造方式
---
## Overview
最初的Spring REST客户端具有类似于Spring中其他模板类的API，如JdbcTemplate，JmsTemplate等。
RestTemplate具有同步API并依赖于阻止I / O，这对于低并发性的客户端方案来说是可以的。
但是高并发情况下，请使用WebClient，非阻塞方案。

-----

### 普通Get请求带url参数
```java
String id = "123";
ResponseEntity<String> result = restTemplate.getForEntity("url/{id}",String.class,id);
```
### Get请求带Header信息+url键值对+cookie
```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);
headers.add("Cookie", COOKIE);
UriComponentsBuilder urlVariables = UriComponentsBuilder.fromHttpUrl("url").queryParam("key"."value");
String url = urlVariables.build().encode().toUri().toString();
HttpEntity<String> httpEntity = new HttpEntity<>(headers);
ResponseEntity<String> result = restTemplate.exchange(url, HttpMethod.GET, httpEntity, String.class);
```
或者
```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);
headers.add("Cookie", COOKIE);
headers.add("key","value");
HttpEntity<String> httpEntity = new HttpEntity<>(headers);
ResponseEntity<String> result = restTemplate.exchange("url", HttpMethod.GET, httpEntity, String.class);
```

### 普通POST请求带Json Body
```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);
headers.add("Cookie", COOKIE);

String jsonBody = ObjectToJsonString(proMeta);

HttpEntity<String> httpEntity = new HttpEntity<String>(jsonBody, headers);

ResponseEntity<String> result = restTemplate.exchange("url", HttpMethod.POST, httpEntity, String.class);
```
ObjectToJsonString:
```java
private static String ObjectToJsonString(Object obj) {
    //**这个mapper一般为成员函数，不放在方法中**
    ObjectMapper mapper = new ObjectMapper();
    try {
        return mapper.writeValueAsString(obj);
    } catch (JsonProcessingException e) {
        e.printStackTrace();
    }
    throw new RuntimeException("ObjectToJsonString failed.");
}

```

### 普通POST请求带x-www-form-urlencoded
```java
MultiValueMap<String, String> multiValueMap = new LinkedMultiValueMap<>();

multiValueMap.add("key", "value");

HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

HttpEntity<MultiValueMap<String, String>> httpEntity = new HttpEntity<>(multiValueMap, headers);

ResponseEntity<String> result = restTemplate.exchange("url", HttpMethod.POST, httpEntity, String.class);
```

### 重写Converter，比如重写一个Converter，即使返回text的数据，也尝试格式化为json。因为有些接口不规范，返回类型为text/html类型的json字符串，可以这样解决。
```java
public class MyMappingJackson2HttpMessageConverter extends MappingJackson2HttpMessageConverter
{
    public MyMappingJackson2HttpMessageConverter(){
        List<MediaType> mediaTypes = new ArrayList<>();
        mediaTypes.add(MediaType.TEXT_HTML);
        mediaTypes.add(MediaType.APPLICATION_JSON);
        setSupportedMediaTypes(mediaTypes);// tag6
    }
}
```
然后这样配置RestTemplate
```java
@Bean
public RestTemplate getRestTemplate()
{
    RestTemplate restTemplate = new RestTemplate();
    restTemplate.getMessageConverters().add(new MyMappingJackson2HttpMessageConverter());
    return restTemplate;
}
```
