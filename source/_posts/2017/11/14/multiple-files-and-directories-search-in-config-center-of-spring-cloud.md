---
title: SpringCloud配置中心多级目录多文件匹配搜索
date: 2017-11-14 23:03:26
categories: Spring Cloud
tags:
    - Spring Cloud
    - Spring Boot
---
![spring-cloud](/images/post/2017/11/10/spring-cloud-logo.jpg)

SpringCloud将所有配置通过文件或Git模式集中化，微服务在启动时通过注册中心找到配置中心并拉取对应配置文件，
让微服务动态更新配置成为可能。很多情况下，不同功能角色的配置文件分散在不同的配置文件中，比如`redis`和`rabbitmq`，
多文件匹配SpringCloud也是支持的，如下：

<!-- more -->
配置中心 application.yml：
```yaml
spring:
    application:
        name: config-center
    cloud:
        bus:
            trace:
                enabled: true
        config:
            server:
                git:
                    uri: ${GIT_URL:https://git.xyz.com/cloud-config-repo.git}
                    username: ${GIT_USERNAME:someuser}
                    password: ${GIT_PASSWORD:somepass}
                    clone-on-start: ${CLONE_ON_START:true}
                    # 搜索配置仓库多级目录
                    search-paths: services/**,platform/**

```


微服务bootstrap.yml:
```yaml
spring:
    application:
        name: base-service
    profiles:
        active: default
    cloud:
        config:
            # 匹配配置中心多个配置文件
            name: common,datasource,redis,rabbitmq,${spring.application.name},${spring.application.name}-application
            profile: ${spring.profiles.active}
            label: ${COFNIG_BRANCH:master}
            discovery:
                enabled: true
                service-id: config-center
```

通过命令行`spring.profiles.active`参数指定目标运行环境（dev、test、prod），比如运行在test环境，微服务启动时会去配置中心匹配如下文件：
- common-test.properties
- datasource-test.properties
- redis-test.properties
- rabbitmq-test properties
- base-service-test.properties
- base-service-application.yml
