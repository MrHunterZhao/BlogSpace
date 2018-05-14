---
title: SpringCloud集成Redis哨兵
date: 2017-11-20 21:03:26
categories: Spring Cloud
tags:
    - Spring Cloud
    - Spring Boot
    - Redis
---
![spring-cloud](/images/post/2017/11/10/spring-cloud-logo.jpg)
在SpringCloud中我们可以很容易地集成并使用Redis缓存，在pom.xml中加入依赖，
<!-- more -->
pom.xml
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

在application.yml中加入如下内容：
```yaml
spring:
    profiles:
        active: default
    redis:
        sentinel:
            master: ${spring.redis.sentinel.master}
            nodes: ${spring.redis.sentinel.nodes}
        pool:
            max-idle: ${spring.redis.pool.max-idle}
            min-idle: ${spring.redis.pool.min-idle}
            max-active: ${spring.redis.pool.max-active}
            max-wait: ${spring.redis.pool.max-wait}
```


在Configuration中加入`RedisTemplate`，然后在代码中注入RedisTemplate即可使用。
```java
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
    
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericToStringSerializer<>(String.class));
        redisTemplate.setConnectionFactory(factory);
        return redisTemplate;
    }

```

