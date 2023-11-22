---
layout: post
title: 프로젝트를 진행하다 쿼리가 의도치 않게 많이 나간다.
image: 
  path: /assets/img/blog/jeremy-bishop@0,5x.jpg
description: >
  프로젝트를 진행하다 쿼리가 의도치 않게 많이 나가던 이슈를 해결 및 해결 과정을 작성해보았습니다.
sitemap: false
---
# 프로젝트를 진행하다 쿼리가 의도치 않게 많이 나간다.
SMS 프로젝트를 진행하면서 겪었던 문제를 파악 및 해결한 경험을 작성해보았습니다.

# 문제
SMS를 개발하던 당시 회원 탈퇴를 평상시와 같이 
JPA의 deleteAll()로 작성하여 개발하였을때 쿼리가 생각했던 것보다 더욱 발생하고 있었습니다.

위 사진처럼 학생의 language_certificate 를 삭제하는데 쿼리가 학생이 보유한 language_certificate 개수만큼 나가고 있었죠.

간략히 말하면 기능자체는 돌아가지만 유저가 가지고 있는 개수가 어마무시하다면
쿼리는 그 어마무시한 개수만큼 나가는것은 데이터베이스나 서버에 무리가 많이 갈거 같았습니다.

그래서 전 왜 이런 사태가 발생하나 JPA에 deleteAll()의 코드를 살펴보았습니다
.
``` java
@Override
@Transactional
public void deleteAll(Iterable<? extends T> entities) {

  Assert.notNull(entities, "Entities must not be null!");

  for (T entity : entities) {
  delete(entity);
  }

}
```

코드를 보니 deleteAll() 이란 함수는 받은 N개의 엔티티들을 null이 아닐시 반복문을
통해 delete()하고 있었습니다.

여기서 전 아 이래서 쿼리가 엔티티 하나당 하나로 나간거구나! 라고 깨닫게 되었습니다.
그럼 이 문제를 해결하려면 어떤 방식으로 해결할 수 있을까요?

제가 생각해낸 방식은 크게 3가지 정도 였습니다.

1. JPA의 deleteAllInBatch()를 사용한다.

2. @Query를 통하여 쿼리문을 작성하여 날려본다.

3. QueryDSL을 사용한다.

한번 하나씩 살펴보도록 하겠습니다.

1. deleteAllInBatch()
이걸 아는 분들은 별로 없을텐데요. 구현된 코드를 보게 되면

```java
@Override
@Transactional
public void deleteAllInBatch(Iterable<T> entities) {

 Assert.notNull(entities, "Entities must not be null!");

 if (!entities.iterator().hasNext()) {
  return;
 }

 applyAndBind(getQueryString(DELETE_ALL_QUERY_STRING, entityInformation.getEntityName()), entities, em)
   .executeUpdate();
}

public abstract class QueryUtils {

 public static final String COUNT_QUERY_STRING = "select count(%s) from %s x";
 public static final String DELETE_ALL_QUERY_STRING = "delete from %s x";
 public static final String DELETE_ALL_QUERY_BY_ID_STRING = "delete from %s x where %s in :ids";
 ```
흠 보니 쿼리를 통하여 요청을 보내는 것을 볼 수 있습니다 거의 2번째 생각과 비슷한 느낌입니다.  

하지만 deleteAllInBatch()는 코드에서도 보이시다시피 외래키 관련 삭제는 어렵습니다.

또한 단점으로는 delete()와 deleteAllInBatch() 모두 삭제하기 전에 전체적인 데이터에 대한 select()를 한번 하고 1차캐시에 저장하기에 데이터가 많을수록 메모리가 부담이 됩니다.

3번은 2번을 QueryDSL로 변환한 느낌이라 같이 설명해보도록 하겠습니다.

@Query를 사용하여 작성하면 이런 느낌으로 작성할 수 있을거 같습니다.
``` kotlin
@Modifying
@Query("delete from ImageJpaEntity pt where pt.project.id in :projects")
fun deleteAllByProjects(@Param("projects") projects: List<Long>)
```
그럼 이걸 QueryDSL로 변경하게되면?
```kotlin
  override fun deleteAllByProjects(projects: List<Project>) {
        val projectTechStack = QProjectTechStackJpaEntity.projectTechStackJpaEntity
        jpaQueryFactory.delete(projectTechStack)
            .where(projectTechStack.project.id.`in`(projects.map { it.id }))
            .execute()
    }
```
이런식이 됩니다. 

위 아래는 똑같은 역할을 하는 코드들이죠 여기서 어떤 방식을 쓰는것을 추천하나요?
라고 하게 된다면 저는 프로젝트를 진행하며 쿼리가 엄청나게 복잡해지게 되었을 때

@Query를 사용하였을 때 쿼리가 길어지게 된다면 가독성이 안좋아질거 같다라고 생각하기에
QueryDSL을 추천해드릴거 같습니다.

QueryDSL은 아쉽게도 기본적으론 1차캐시를 사용하지 않는다라는 특징이 있습니다.
(커스텀 가능)

## 해결 방법 및 느낀 점
저는 프로젝트에 QueryDSL을 통하여 위와 같은 문제를 해결하게되었습니다.

깊게 내용을 들어가보니 기능을 구현하는것에 그치지않고 Query 개수, 메모리 부분에도 더욱 신경을 써야겠다고 느끼게 되었습니다.

## 관련 링크

[Github](https://github.com/GSM-MSG/SMS-BackEnd/pull/234)