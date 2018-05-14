---
title: SpringCloud设置HttpMessageConverter为fastjson格式化输出
date: 2017-11-16 23:03:26
categories: Spring Cloud
tags:
    - Spring Cloud
    - Spring Boot
---
![spring-cloud](/images/post/2017/11/10/spring-cloud-logo.jpg)

我们的API响应的Media类型一般是`application/json;charset=UTF-8`，在SpringCloud中可以通过如下方式设置，

并且将HttpMessageConverter设置为fastjson，使用fastjson提供的各种功能格式化输出内容。

<!-- more -->
```java
    @Bean
    public HttpMessageConverters fastjsonHttpMessageConverter() {

        StringHttpMessageConverter converter = new StringHttpMessageConverter(StandardCharsets.UTF_8);

        FastJsonHttpMessageConverter fastConverter  = new FastJsonHttpMessageConverter();
        // 配置Media类型
        List<MediaType> supportedMediaTypes = new ArrayList<>();
        supportedMediaTypes.add(MediaType.TEXT_HTML);
        supportedMediaTypes.add(MediaType.APPLICATION_JSON_UTF8);
        fastConverter.setSupportedMediaTypes(supportedMediaTypes);
        
        // fastjson配置
        FastJsonConfig fastJsonConfig = new FastJsonConfig();
        fastJsonConfig.setSerializerFeatures(SerializerFeature.PrettyFormat);
        fastJsonConfig.setSerializerFeatures(SerializerFeature.DisableCircularReferenceDetect);
        fastJsonConfig.setSerializerFeatures(SerializerFeature.WriteMapNullValue);
        fastJsonConfig.setSerializerFeatures(SerializerFeature.WriteNullStringAsEmpty);
        fastConverter.setFastJsonConfig(fastJsonConfig);
        
        return new HttpMessageConverters(converter, fastConverter);
    }

```

