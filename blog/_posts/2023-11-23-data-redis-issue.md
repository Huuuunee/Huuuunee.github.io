---
layout: post
title: 레디스 TTL이 정상적으로 작동하지 않던 이슈가 생겼습니다.
image: 
  path: /assets/img/blog/jeremy-bishop@0,5x.jpg
description: >
  프로젝트를 배포하고 유지보수중 인덱싱 걸린 데이터의 삭제가 이루어지지 않는것을 보게되었습니다.
sitemap: false
---

## Redis의 TTL이 동작하지 않던 이슈.
프로젝트의 Redis를 적용 후 배포 한 후 redis의 데이터가 TTL이 지나도 안 사라지던 이슈를 발견하게 되었습니다.

다른 사람들도 이런 사례가 존재하였는지 찾아보려 구글링을 해보니 @Indexed가 지정된 컬럼이 TTL이 적용이 안된다라는 저와 같은 사례를 찾게 되었습니다.  

왜 그런가 봤더니 github issue 중 **Index cleanup happens on the application side and isn't automatially performed by Redis servers** 즉  
기본적으로 @RedisHash에서 @Id가 달려있는 Key에 대해서만 TTL기능이 동작한다는거 였습니다.

저희 코드를 보니
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
위와 같이 @Indexed가 붙어져 있었죠.
### 해결방법
저러한 이슈를 찾은 후 해결하기 위한 방법을 보니 RedisRepository에서 index가 걸려있는 키들을 삭제하기위해선 keyspace events가 탐지되어야 같이 삭제할 수 있다라는 방법을 찾게 되어

RedisConfig에 추가해보았습니다 

```kotlin
@EnableRedisRepositories(enableKeyspaceEvents = RedisKeyValueAdapter.EnableKeyspaceEvents.ON_STARTUP)
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }
}
```
해당 옵션을 설정하게 되면 데이터가 삭제될때 Spring Data Redis에서 Key Events를 수신하게되며, Key expired events를 수신하여 Index된 키들도 삭제가 가능하게 됩니다.

## 느낀점 및 배운점
Redis에서 Index이 걸린 키들이 @Id가 달려있는 키의 TTL이 만료되며 삭제될때 keyspace events가 탐지되어야 삭제를 진행할 수 있다는것과 배포하기전 테스트를 더욱 꼼꼼하게 진행해야 할거 같다라는 것을 배웠습니다.