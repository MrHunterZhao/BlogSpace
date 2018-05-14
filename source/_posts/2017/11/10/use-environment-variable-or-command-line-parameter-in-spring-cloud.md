---
title: SpringCloud中读取命令行参数或系统环境变量
date: 2017-11-10 20:03:26
categories: Spring Cloud
tags:
    - Spring Cloud
    - Spring Boot
---

![spring-cloud](/images/post/2017/11/10/spring-cloud-logo.jpg)

SpringCloud中帮我们很方便地提供了环境适配方案，通过命令行参数或export系统环境变量指定`REGISTRY_CENTER_URI`值，

如果读取到，SpringCloud优先使用该值，如果该变量未指定则使用默认值`http://localhost:9000/eureka/`
<!-- more -->

```yaml
eureka:
    instance:
        prefer-ip-address: true
    client:
        serviceUrl:
            defaultZone: ${REGISTRY_CENTER_URI:http://localhost:9000/eureka/}

```



