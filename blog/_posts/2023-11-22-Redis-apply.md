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
이제 yml에 넣어논 주소에 연결하게 됩니다.  
데이버베이스에 접속까지 하였으니 한번 회원가입을 해보도록 하겠습니다.

## 구현
Redis에 저장할 테이블을 생성해보겠습니다.

``` kotlin
@RedisHash(value = "refresh_token", timeToLive = 60L * 60 * 24 * 7)
data class RefreshTokenEntity(
    @Id
    @Indexed
    val token: String,
    @Indexed
    val userId: UUID
)
```
살펴 보았을때 @RedisHash 어노테이션에 value를 통하여 나중에 생성되는 key의 prefix를 지정해줄 수 있고 @Id 어노테이션을 통하여 prefix:구분자 형태(keyspace:@id)로 데이터에 대한 키를 저장하여 각 데이터를 구분할 수 있습니다. @Id가 붙여져있는 컬럼이 null로 저장될경우 랜덤값이 들어가게 됩니다.

추가적으로 저희는 리프레시 토큰을 저장하기에 리프레시 토큰의 만료시간이 다되면 데이터가 폐기되도록
TTL을 리프레시 토큰의 만료 시간과 일치시켜줬습니다.

``` kotlin
@UseCase
class SignInUseCase(
    private val gAuthPort: GAuthPort,
    private val jwtPort: JwtPort,
    private val refreshTokenPort: RefreshTokenPort,
    private val userService: UserService,
    private val studentService: StudentService
) {

    fun execute(request: SignInRequestData): SignInResponseData =
    ...
            val (accessToken, accessTokenExp, refreshToken, refreshTokenExp) = jwtPort.receiveToken(user.id, role)

            refreshTokenPort.saveRefreshToken(RefreshToken(refreshToken, user.id))
    ...
}

```

여기서 살펴볼 곳은 saveRefreshToken() 메소드인데요 이 메소드를 호출하게 될시

``` kotlin
@Repository
interface RefreshTokenRepository : CrudRepository<RefreshTokenEntity, Long> {
    fun findByToken(token: String): RefreshTokenEntity?
}
```
이 CrudRepository interface의 존재하는 save함수를 호출하게 되는 로직입니다.  
위 코드를 바라보니 data jpa 와 굉장히 비슷한 모습을 띄고 있는걸 확인할 수 있습니다.

그리고 저흰 토큰 재발급도 구현이 되어 있는데요!

```kotlin
@UseCase
class ReIssueTokenUseCase(
    private val refreshTokenPort: RefreshTokenPort,
    private val userService: UserService,
    private val jwtPort: JwtPort
) {
    fun execute(token: String): ReIssueTokenResponseData {
       ...
        refreshTokenPort.deleteRefreshToken(queryToken)

        val (accessToken, accessTokenExp, refreshToken, refreshTokenExp) = jwtPort.receiveToken(user.id, role)

        refreshTokenPort.saveRefreshToken(
            refreshToken = RefreshToken(
                token = refreshToken,
                userId = user.id
            )
        )

       ...
    }
}
```
위 코드를 보게 되면 @Id로 지정되어 있는 token 값으로 Redis에서 삭제 후 발급 한 Token을 저장하도록 구현하였습니다.

한번 실행하여 로그인 시도를 해보도록 하겠습니다.

### 로그인 시도 전 Redis
![](/assets/img/blog/redis/%EB%B9%88%EB%A0%88%EB%94%94%EC%8A%A4.png)

아무 값이 들어있지 않은 상태입니다.

### 첫 로그인 요청을 보낸 상태
![](/assets/img/blog/redis/%EB%A1%9C%EA%B7%B8%EC%9D%B8%EC%9A%94%EC%B2%AD.png)

로그인 성공으로 토큰이 정상적으로 발급되어있는 모습을 볼 수 있고 Redis를 보았을때
![](/assets/img/blog/redis/%EC%B2%AB%EB%A1%9C%EA%B7%B8%EC%9D%B8.png)

값이 제대로 저장된 것을 볼 수 있습니다 !

### 토큰 재발급
토큰을 이용하여 재발급 요청을 해봤습니다.
![](/assets/img/blog/redis/%EC%9E%AC%EB%B0%9C%EA%B8%89.png)

재발급에 성공하여 새로운 토큰들이 발급된 모습을 볼 수 있고 마찬가지로 Redis를 확인했을때
![](/assets/img/blog/redis/%EC%9E%AC%EB%B0%9C%EA%B8%89%ED%86%A0%ED%81%B0.png)

기존에 있던 토큰은 삭제가 되었고 새로운 토큰이 저장되어있는 모습을 볼 수 있습니다.

## 마무리
보다보면 궁금점이 생길 수 있는데요 왜 token을 @Id로 사용하는거지? userId 라는 고유값이 있는데? 저희 프로젝트는 안드로이드, 아이폰, 웹사이트 세곳을 모두 지원하는 서비스이다 보니 userId를 Id로 사용시 userId:토큰값까지 모두 검사를 해야하여 속도적으로 차이가 나서 token을 Id로 지정하게 되었습니다.  

이번에 Redis를 도입해본김에 캐싱도 한번 적용해보고 싶어졌습니다 읽어주셔서 감사합니다!