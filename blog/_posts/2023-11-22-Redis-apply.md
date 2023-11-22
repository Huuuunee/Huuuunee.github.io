---
layout: post
title: 프로젝트에 레디스 적용하기.
image: 
  path: /assets/img/blog/jeremy-bishop@0,5x.jpg
description: >
  프로젝트 레디스 세팅 부터 적용까지.
sitemap: false
---
## 프로젝트에 Redis 적용하기.
사용자 인증 키를 Redis에 저장하도록 해보겠습니다.

시작하기전 Redis를 사용하기 위해서 Docker로 먼저 띄우고 진행합니다!  

![도커](/assets/img/blog/redis/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202023-11-22%20%EC%98%A4%ED%9B%84%209.02.14.png)

<br>

## 1단계

스프링에서 사용하기 위하여 data-redis 의존성을 추가해줘야 합니다.
```java
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

또한 yml의 redis 정보를 적어주셔야 합니다.
``` yml
spring:
  redis:
    host: 127.0.0.1 //localhost
    port: 6379
```
작성을 완료해주셨다라면 다음 단계로 넘어가야 합니다.

## 2단계

redisConfig을 작성하여 Configuration을 통해 빈 설정 정보를 등록해보도록 하겠습니다.

``` kt
@Configuration
class RedisConfig(

    @Value("\${spring.redis.host}")
    private val redisHost: String,

    @Value("\${spring.redis.port}")
    private val redisPort: Int
) {

    @Bean
    fun redisConnectionFactory(): RedisConnectionFactory {
        val redisConfig = RedisStandaloneConfiguration(redisHost, redisPort)
        val clientConfig = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofSeconds(1))
            .shutdownTimeout(Duration.ZERO)
            .build()

        return LettuceConnectionFactory(redisConfig, clientConfig)
    }
}
```

