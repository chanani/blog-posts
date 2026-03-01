---
title: "Mysql Replication 트러블슈팅"
date: "2025-09-21"
tags: [ "Replication", "트러블슈팅", "Master/Slave" ]
description: "Replication 트러블슈팅"
---

## 🫢 문제 상황

Spring Boot에서 Master/Slave 패턴으로 DB 구조를 구현했지만,
<strong style="color:#ee2323;">@Transaction(readOnly = true)</strong>로 설정한 메서드가 계속 Master DB(3306)로 접근하는 문제가 발생하였습니다.
Slave DB(3307)로 라우팅되어야 하지만 실제로는 읽기 전용 설정이 <strong style="color:#ee2323;">false</strong>로 인식되는 문제가 발생하고 있었습니다.

## ⚙️ 환경설정

#### DB 설정(application.yml)

``` yaml
spring:
  datasource:
    master:
      hikari:
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/bank?serverTimezone=Asia/Seoul
        read-only: false
        username: root
        password: password!
    slave:
      hikari:
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3307/bank?serverTimezone=Asia/Seoul
        read-only: true
        username: root
        password: password!
```

#### 라우팅 데이터소스 설정

``` java
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? DataSourceType.SLAVE : DataSourceType.MASTER;
    }
}
```

#### 서비스 클래스

``` java
@Service
@Transactional(readOnly = true)
public class DefaultUserService implements UserService {
    
    @Override
    public List<UsersResponse> getUsers() {
        return userRepository.findAll();
    }
}
```

## 🧐 문제 분석

#### 로그 분석 결과

```
=== 라우팅 결정 ===
ReadOnly: false
Selected: MASTER
==================
=== 트랜잭션 상태 ===
Active: true
ReadOnly: true
===================
```

위 로그에서 활인할 수 있듯이 라우팅 결정 시점에서는 ReadOnly가 false,
실제 트랙잭션 시작 후에는 ReadOnly가 true로 작동되고 있습니다.

#### 근본 원인

Spring의 AOP 순서 때문에 발생하는 문제입니다. 실제 순서를 먼저 확인해보겠습니다.

> 메서드 호출 시 Spring AOP 체인
> 1. RoutingDataSource.determineCurrentLookupKey() 호출 (트랜잭션 시작 전)
> 2. @Transactional AOP 적용 (트랜잭션 시작)
> 3. 실제 메서드 실행

**핵심 문제** : 데이터 소스 라우팅 결정이 트랙잭션 시작보다 먼저 일어나기 때문에
<strong style="color:#ee2323;">TransactionSynchronizationManager.isCurrentTransactionReadOnly()</strong>가 항상
<strong style="color:#ee2323;">false</strong>를 반환합니다.

#### 왜 이런 구조일까 ?

1. Hibernate/JPA 동작 방식 : 쿼리 실행 전에 미리 DataSource를 결정해야합니다.
2. Spring AOP 설계 : 각 AOP는 독립적으로 동작하도록 설계되어있습니다.
3. Connection Pool 관리 : 데이터베이스 연결을 미리 준비해야 성능상 유리합니다.

## ✅ 해결 방법

TreadLocal을 활용해 <strong style="color:#ee2323;">@Transactional</strong>어노테이션 정보를 미리 캐싱하는 방식으로 해결했습니다.

#### TransactionContextHolder 생성

``` java
// 스레드별로 트랙잭션의 읽기 전용 상태를 저장하는 컨텍스트 홀더
@Component
public class TransactionContextHolder {
    private static final ThreadLocal<Boolean> readOnlyContext = new ThreadLocal<>();

    public static void setReadOnly(boolean readOnly) {
        readOnlyContext.set(readOnly);
    }

    public static boolean isReadOnly() {
        Boolean readOnly = readOnlyContext.get();
        return readOnly != null && readOnly;
    }

    public static void clear() {
        readOnlyContext.remove();
    }
}
```

#### TransactionContextInterceptor 생성

- @Order(0) : 가장 먼저 실행되어 다른 AOP보다 우선순위를 가짐
- 메서드 및 클래스 레벨의 @Transactional 어노테이션 정보 추출
- ThreadLocal에 readOnly 상태 저장
- 메서드 실행 완료 후 ThreadLocal 정리

``` java
@Component
@Aspect
@Order(0)  // 가장 높은 우선순위로 실행
public class TransactionContextInterceptor {

    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object setTransactionContext(ProceedingJoinPoint joinPoint) throws Throwable {
        // 메서드의 @Transactional 어노테이션 정보 가져오기
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Transactional transactional = signature.getMethod().getAnnotation(Transactional.class);

        if (transactional == null) {
            // 클래스 레벨 어노테이션 확인
            transactional = joinPoint.getTarget().getClass().getAnnotation(Transactional.class);
        }

        try {
            if (transactional != null) {
                TransactionContextHolder.setReadOnly(transactional.readOnly());
            }
            return joinPoint.proceed();
        } finally {
            TransactionContextHolder.clear();
        }
    }
}
```

RoutingDataSource 수정

``` java
public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionContextHolder.isReadOnly();
        return isReadOnly ? DataSourceType.SLAVE : DataSourceType.MASTER;
    }
}
```

## ✔️ 해결 후 결과

```
=== AOP에서 설정 ===
ReadOnly: true
==================
=== 라우팅 결정 (ThreadLocal) ===
ReadOnly from ThreadLocal: true
Selected: SLAVE
==============================
p6spy - connection 21| url jdbc:mysql://localhost:3307/bank
```

#### 결론

Spring Boot에서 Master/Slave DB 라우팅을 구현할 때 <strong style="color:#ee2323;">@Transactional(readOnly=true)</strong>
설정만으로는 자동 라우팅이 되지 않습니다. 이는 Spring AOP의 실행 순서상 데이터 소스 라우팅 결정이 트랙잭션
시작보다 먼저 일어나기 때문입니다.
많은 해결 방법을 찾아봤는데 하나하나 찾아가다 보니 결론은 '순서'라는 답을 찾았고,
ThreadLocal과 AOP를 활용한 우선순위 제어가 실용적인 해결책임을 확인했습니다.
오늘도 쾌락을 느꼈고, 앞으로도 문제를 찾아 재미있게 문제를 해결해보려고합니다 ☺️