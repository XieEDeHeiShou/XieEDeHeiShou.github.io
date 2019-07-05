---
title: "How to solve 'DiscoveryJerseyProvider   : Cannot parse request body'"
published: true
---

微服务向新配置的 Eureka 注册中心发起注册请求时出现以下异常:

>DiscoveryJerseyProvider   : Cannot parse request body
>
>com.fasterxml.jackson.databind.JsonMappingException: Root name 'timestamp' does not match expected ('instance') for type [simple type, class com.netflix.appinfo.InstanceInfo]

### Solution: Disable eureka's csrf protection.

由于 Spring security 的 csrf 保护默认是开启的, 这导致微服务在发起注册请求时由于权限不足导致请求被拒绝,
进而触发 Json 解析异常. 因此关闭 Eureka 注册中心的 csrf 保护即可.

```java
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
        http.csrf().disable();
    }
}
```

### Reference:
+ [Eureka Java client heartbeat is failing intermittently](https://github.com/Netflix/eureka/issues/1006)