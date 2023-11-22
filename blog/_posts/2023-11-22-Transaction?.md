---
layout: post
title: @Transaction은 뭘까?
image: 
  path: /assets/img/blog/jeremy-bishop@0,5x.jpg
description: >
  트랜잭션에 대해서 공부 해보았습니다.
sitemap: false
---

## @Transaction?
----
저희가 자주 사용하던 @Transaction은 어떻게 등장하게 되었고 동작할까에 대하여 공부하였습니다.

## JDBC 상에서 트랜잭션을 사용하기
<br>

데이터베이스에서 단순한 **INSERT** 문 하나만 실행하여도 auto commit 기능이 동작하여
데이터베이스에 쿼리 결과가 반영되는 모습을 확인할 수 있습니다.
<br>

개발을 하다보니 여러개의 커밋을 하나의 논리적인 단위로 묶어 실행시켜야 하는 일이 자주 있습니다. 이 단위는 **원자적(atomic)** 으로 동작해야하죠. 저흰 이것을 트랜잭션이라고 칭합니다,  
auto commit이 활성된 상태에서 각각의 단일 쿼리가 별개의 트랜잭션 상태로 동작하게 되면 원자성을 지키지 못하고, 롧백 또한 어렵습니다.  
  
JDBC를 사용할때 저흰 위와 같은 상황을 방지하여 원자성을 보장하기 위하여 auto commit 기능이 아닌 직접 커밋과 롤백의 시점을 지정해줘야 하죠.

Connection 은 setAutoCommit() 이라는 메소드를 통하여 auto commit을 활성화 할지 하지 않을지를 제어할 수 있게 도와줍니다. 

한번 상황을 구성하여 보겠습니다
<br>

## 상황
---

UserDao가 존재 하고 회원가입 기능을 구현후 회원가입이 성공하였을때 user 테이블에 유저를 추가한 후, message라는 테이블에 회원가입 축하 메시지를 추가해야 하는 코드를 작성해보도록 하겠습니다.  
이 전체적인 작업은 하나의 트랜잭션에 묶여 동작해야합니다.

``` java
public class UserDao {
    public void saveUser(final Connection connection, final String name) {
        try {
            String sql = "INSERT INTO user(name) VALUES(?)";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
             preparedStatement.setString(1, name);
            preparedStatement.execute();
        } catch (final SQLException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class MessageDao {

    public void saveMessage(final Connection connection, final String message) {
        try {
            String sql = "INSERT INTO message(content) VALUES(?)";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, message);
            preparedStatement.execute();
        } catch (final SQLException e) {
            e.printStackTrace();
        }
    }
}
```

코드를 살펴보았을때 이상한점이 발견됩니다  메소드 내부에서 Connection을 생성하여 사용하는 것이 아닌 파라미터로 주입받아 사용하죠. 이유는 트랜잭션을 사용하기 위해서는 트랜잭션을 구성하는 여러개의 쿼리가 동일한 커넥션에서 실행되어야 하여 주입받아 사용하는것이지요. **UserServic** 를 보게 되면 이해가 쉬울겁니다.

``` java
public class UserService {
     // ...
    public void register(final String name) {
        Connection connection = null;
        try {
            connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);
            connection.setAutoCommit(false);

            userDao.saveUser(connection, name);
            messageDao.saveMessage(connection, name + "님 가입을 환영합니다.");

            connection.commit();
        } catch (final SQLException e) {
            try {
                connection.rollback();
            } catch (final SQLException ignored) {
            }
        } finally {
            try {
                connection.close();
            } catch (final SQLException ignored) {
            }
        }
    }
}
```

서비스 레이어의 관심사는 비즈니스 로직입니다. 하지만 트랜잭션을 하나로 합치기 위해선 UserDao의 쿼리와 MessageDao의 쿼리는 서로 동일한 커넥션을 사용해야하죠. 그러다 보니 비즈니스 로직에 집중해야하는 UserService에도 커넥션을 생성하고, 각 DAO에 주입하는 쿼리가 생긴 것입니다.

Service에서는 커넥션을 생성하고, 트랜잭션의 시작과 끝을 설정하는 코드가 비즈니스 로직과 뒤섞이게 되는 단점이 있고,
DAO 에서는 데이터 엑세스 기술이 Service 레이어에 종속되는 단점이 있습니다.  
<br>

### 트랜잭션 동기화
<br>

Connection을 파라미터로 전달하는 코드 먼저 제거 해보도록 하겠습니다.  
스프링에선 위와 같은 문제를 **트랜잭션 동기화(transaction synchronization)** 로 해결하고 있었는데요,   
트랜잭션 동기화는 트랜잭션을 시작하기 위해 생성한 **Connection** 객체를 별도의 공간에 보관하고,   
커넥션이 필요한 곳에서 커넥션을 꺼내어 사용하는 방식입니다.

이 동기화 작업은 쓰레드마다 독립적으로 Connection 객체를 보관히기에 멀티 쓰레드 환경에서도 걱정없이 사용할 수 있습니다.

스프링에서는 **TransactionSynchronizationManager** 라는 클래스를 사용하여 트랜잭션을 동기화합니다.  
이 클래스를 이용하여 코드를 개선해보도록 하겠습니다. 커넥션을 가져올땐 DataSource가 필요하여 yml에 값을 추가 해주겠습니다.

``` 
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
```

그리고 DataSource 빈을 주입받기 위허여 Service에선 @Service, DAO 에선 @Repository 어노테이션을 추가하여줬습니다. 또한 Service에선 DAO들을 생성하지 않고 주입받도록 변경하였습니다.

``` java
@Service
public class UserService {

    private final DataSource dataSource;
    private final UserDao userDao;
    private final MessageDao messageDao;

    public UserService(final DataSource dataSource, final UserDao userDao, final MessageDao messageDao) {
        this.dataSource = dataSource;
        this.userDao = userDao;
        this.messageDao = messageDao;
    }
}
```

UserService에서 TransactionSynchronizationManager 클래스를 사용하여 트랜잭션 동기화 작업을 활성화해보고, DataSourceUtils 라는 클래스에서 커넥션을 가져오도록 하겠습니다.

```java
@Service
public class UserService {
    public void register(final String name) {
        TransactionSynchronizationManager.initSynchronization();
        // 트랜잭션 동기화 초기화

        Connection connection = DataSourceUtils.getConnection(dataSource);
        // 커넥션 획득

        try {
            connection.setAutoCommit(false);
            //쿼리 발생시 auto commit 해제

            userDao.saveUser(name);
            messageDao.saveMessage(name + "님 가입을 환영합니다.");

            connection.commit();
            // 위 두 작업이 되었을때 commit 실행

        } catch (final SQLException e) {
            try {
                connection.rollback();
            } catch (final SQLException ignored) {
            }
        } finally {
            DataSourceUtils.releaseConnection(connection, dataSource);
            // 커넥션 자원 해제
        }
    }
}

```
UserDao와 MessageDao 는 전처럼 외부에서 Connection을 전달 받지 않고, DataSoruceUtils 를 사용하여 커넥션을 가져오도록 변경하였다.

``` java
@Repository
public class UserDao {

    private final DataSource dataSource;

    public UserDao(final DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void saveUser(final String name) {
        Connection connection = DataSourceUtils.getConnection(dataSource);

        try {
            String sql = "INSERT INTO user(name) VALUES(?)";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, name);
            preparedStatement.execute();
        } catch (final SQLException e) {
            e.printStackTrace();
        }
    }
}

```

## 트랜잭션 추상화

위에서 Connection을 전달하는 코드를 제거하였지만 아직 Service에는 Connection 을 통해 직접적으로 트랜잭션 경계를 설정하는 코드가 살아있다. 스프링은 위를 트랜잭션 추상화로 해결하였다 PlatformTransactionManager 인터페이스를 보게되면

``` java
public interface PlatformTransactionManager extends TransactionManager {

		TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
         throws TransactionException;

		void commit(TransactionStatus status) throws TransactionException;

		void rollback(TransactionStatus status) throws TransactionException;
}
```
3개의 메소드를 정의하고 있다.  
PlatformTransactionManager는 트랜잭션의 시작지점과 끝 지점,종료는 정상(commit)인지 비정상(rollback)인지를 결정한다.
<br>

스프링은 트랜잭션 전파라는 특징을 가지고 있어, 자유롭게 트랜잭션을 조합하여 시작과 끝을 확장할 수 있기에 begin() 이 아닌 getTransaction()이라는 네이밍을 사용해요.

PlatformTransactionManager의 구현체 중 저흰 DataSourceTransactionManager 를 사용해보겠습니다.

``` java
@Service
public class UserService {

    // ...

    public void register(final String name) {
        PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            userDao.saveUser(name);
            messageDao.saveMessage(name + "님 가입을 환영합니다.");

            transactionManager.commit(status);
        } catch (RuntimeException e) {
            transactionManager.rollback(status);
        }
    }
}
```
DataSourceTransactionManager 를 사용하니 트랜잭션 경계를 설정하는 코드가 좀 더 간결해졌습니다.  
PlatformTransactionManager 를 사용하였을때 트랜잭션 동기화를 하기 위한 코드도 제거 할 수 있었죠, 이로써 트랜잭션 동기화 로직도 추상화된것을 볼 수 있습니다.
<br>

사용한 TransactionDefinition 은 트랜잭션의 네 가지 속성 (Transaction Propagation, Isolation Level, Timeout, Read Only)를 나타내는 인터페이스이다. DefaultTransactionDefinition 이라는 구현체를 사용하여 트랜잭션을 가져왔다.

TransactionStatus 는 현재 트랜잭션의 ID와 구분 정보를 담고 있어 커밋이나 롤백 시키는 트랜잭션을 식별할 때 사용됩니다.

``` java
public void registerSpecial(final String name) {
    DataSourceTransactionManager txManager = new DataSourceTransactionManager(dataSource);
    TransactionStatus status = txManager.getTransaction(new DefaultTransactionDefinition());

    try {
        register(name);
        messageDao.saveMessage(name + "님은 특별 회원입니다.");

        txManager.commit(status);
    } catch (RuntimeException e) {
        txManager.rollback(status);
    }
}
```
위 메소드에서 트랜잭션이 전파되는지 확인해보았습니다.  
registerSpecial() 메소드에서 register() 메소드를 내부적으로 호출하여 트랜잭션 전파를 유도했을때. 테스트 결과 두 트랜잭션이 조합되어 원자성을 잘 띄는 것을 확인할 수 있었습니다.

이 글을 읽다보면 어? 나는 이런식으로 된 코드를 거의 보지 못했는데 어떻게 트랜잭션 추상화를 누리고 있던거지? 하였을때
PlatformTransactionManager의 [Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/PlatformTransactionManager.html)을 읽어보았을때

<br>

>Applications can use this directly, but it is not primarily meant as an API: Typically, applications will work with either TransactionTemplate or declarative transaction demarcation through AOP.
<br>

<br>

**AOP를 통한 선언적 트랜잭션 경계**... 저희가 질리도록 사용하였던 @Transacation 어노테이션을 이야기하는 것입니다.
평상시 사용하던 @Transaction 어노테이션이 내부 동작이 어떻게 이루어진지 동작하는 것인지를 알아본겁니다.

<br>

# 마치며

평상시 사용하는 트랜잭션이 편하게 사용할때 안에서 어떤 동작들이 이루어지는지 이해하게 되었습니다.