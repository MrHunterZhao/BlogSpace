---
title: SpringCloud设置zuul网关跨域访问
date: 2017-11-22 21:53:26
categories: Spring Cloud
tags:
    - Spring Cloud
    - Spring Boot
---
![spring-cloud](/images/post/2017/11/10/spring-cloud-logo.jpg)
SpringCloud设置跨域访问只需在zuul网关服务中加入如下configuration即可。

<!-- more -->

```java
    /**
     * CORS配置
     * @return
     */
    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");
        config.addAllowedMethod("OPTIONS");
        config.addAllowedMethod("HEAD");
        config.addAllowedMethod("GET");
        config.addAllowedMethod("PUT");
        config.addAllowedMethod("POST");
        config.addAllowedMethod("DELETE");
        config.addAllowedMethod("PATCH");
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }

```


