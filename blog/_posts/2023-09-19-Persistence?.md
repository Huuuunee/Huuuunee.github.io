---
layout: post
title: 영속성이란?
image: 
  path: /assets/img/blog/jeremy-bishop@0,5x.jpg
description: >
  영속성에 대해서 알아보겠습니다.
sitemap: false
---

## **영속성이란?**

Spring Jpa를 공부하다보면 persistence라는 말을 자주 듣는다.

> persistence 고집 , (없어지지 않고 오래 동안)지속됨

뜻과 같이 **Jpa**에서 영속성이란 Entity를 영구적으로 저장해주는 환경을 의미 합니다.

### **EntitiyManagerFactory**

- 생성하는데 비용이 크게 들기 때문에 애플리케이션 전체에서 하나만 생성후 공유하도록 설계되어 있다.
- 여러 스레드가 접근하여도 안전하다. 따라서 서로 다른 스레드 간에 공유가 가능하다.

### **EntityManager**

- 생성하는데 비용이 크게 들지 않는다
- 여러 스레드가 동시에 접근하면 동시성 문제가 발생하기 떄문에 스레드 간에 절대 공유하지 않는다.

> EntityManger은 데이터베이스와의 연결이 꼭 필요하기 전까지(트랜잭션을 시작할 때)커넥션을 얻지 않습니다.

### 사용하는 방법

```java
EntityMangerFactory entityMangerFactory = Persistence.createEntityMangerFactory("person"); // 파라미터  : 영속성 단위 이름

EntityManager entityManger = entityMangerFactory.createEntityManger();
```

## **JPA 영속성 컨텍스트란?**

위에서도 말했다시피 영속성 컨텍스트(persistence context)는 **엔티티를 영구 저장하는 환경**이라는 뜻입니다.애플리케이션과 데이터베이스 사이에서 객체를 보관하는 가상의 데이터베이스 같은 역할을 합니다.

### **영속성 컨텍스트의 특징**

**영속성 컨텍스트의 식별값**  
영속성 컨텍스트는 엔티티를 식별자 값으로 구분한다. 따라서 영속 상태는 식별자 값이 반드시 있어야 합니다.

**영속성 컨텍스트와 데이터베이스 저장**  
JPA는 보통 트랜잭션을 커밋하는 순간 영속성 컨텍스트에 새로 저장된 엔티티를 데이터 베이스에 반영하는데 이를 flush라 한다.

### **영속성 컨텍스트의 장점**

1. 1차캐시
2. 동일성 보장
3. 트랜잭션을 지원하는 쓰기 지연
4. 변경감지
5. 지연 로딩