---
title: SpringCloud自定义Feign解码器
date: 2017-12-02 22:53:26
categories: Spring Cloud
tags:
    - Spring Cloud
    - Spring Boot
---
![spring-cloud](/images/post/2017/11/10/spring-cloud-logo.jpg)
在SpringCloud微服务中，Feign组件帮我们把跨服务HTTP请求模板化，我们的FeignClient看上去可能是下面这样，

返回值都被封装在一个ApiResponse中，调用者获取真正内容时需要再次获取ApiResponse中的data内容，略显恶心。

<!-- more -->
```java
@FeignClient(value = Const.FeignClient.USER_CENTER, path = "user")
public interface UserCloudService {

    @PostMapping("save")
    ApiResponse<User> saveUser(@RequestBody UserVo userVo);

}
```

使用其Decoder让我们更方便地自定义FeignClient中的方法返回值，

```java
    @Bean
    public Decoder decoder() {
        return (response, type) -> {
            String body = Util.toString(response.body().asReader());
            ObjectMapper mapper = new ObjectMapper();
            JavaType javaType = mapper.getTypeFactory()
                    .constructParametricType(ApiResponse.class, mapper.getTypeFactory().constructType(type));
            ApiResponse<?> apiResponse = mapper.readValue(body, javaType);
            return apiResponse.getData();
        };
    }

```



所以以上FeignClient就会变成如下，跟普通Service无异，平滑过渡。
```java
@FeignClient(value = Const.FeignClient.USER_CENTER, path = "user")
public interface UserCloudService {

    @PostMapping("save")
    User saveUser(@RequestBody UserVo userVo);

}
```


