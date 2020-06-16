<img src="https://user-images.githubusercontent.com/62231786/84779482-6a2c8a00-b01f-11ea-8e94-e06a3501ae01.png" width="40%"/>

# 이러닝 시스템

# Table of contents

- [서비스 시나리오](#서비스-시나리오)
- [분석/설계](#분석설계)
- [구현:](#구현-)
  - [DDD 의 적용](#ddd-의-적용)
  - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
  - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
- [운영](#운영)
  - [CI/CD 설정](#cicd설정)
  - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
  - [오토스케일 아웃](#오토스케일-아웃)
  - [무정지 재배포](#무정지-재배포)
- [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
1. 관리자가 과정을 등록한다.
1. 학생이 회원가입을 한다.
1. 학생이 과정을 선택하고 수강신청한다.
1. 학생이 수강신청 과정을 결제한다.
1. 결제가 완료되면 수강신청 승인된다.
1. 학생이 과정 수강을 완료되면 수료 승인된다.
1. 학생이 수강신청 취소를 하면 결제가 취소된다.
1. 관리자가 과정을 삭제하면 수강신청 내역이 취소된다.
1. 학생이 수강내역 상태를 조회한다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 수강신청은 승인되지 않는다. > Sync 호출 
1. 장애격리
    1. Certification시스템 기능이 수행되지 않아도 수강완료(Lecture Completed) 는 수행된다. > Async (event-driven), Eventual Consistency
    1. Payment시스템이 과중되면 결제를 잠시후에 하도록 유도한다. > Circuit breaker, fallback
1. 성능
    1. 학생은 과정수강이력을 확인할 수 있다. > CQRS


# 분석/설계

## Event Storming 결과

<img src="https://user-images.githubusercontent.com/62231786/84783162-e5903a80-b023-11ea-9fb6-ffc453c534cf.png"/>

<img src="https://user-images.githubusercontent.com/62231786/84783192-ef19a280-b023-11ea-9208-d5dd9246c6ce.jpg"/>


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/62231786/84791968-4d4b8300-b02e-11ea-8602-3c6f2a4699ae.png)


# 구현:

## DDD 의 적용

분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.

| MSA | 기능 | port |
|---|:---:|:---:|
| Course | 과정 관리 | 8081 |
| Lecture | 수강 관리 | 8082 |
| Certification | 수료 관리 | 8083 |
| Payment | 결제 관리 | 8084 |
| Student | 학생 관리 | 8085 |

- REST API 의 테스트
```
# 수강신청
http POST http://localhost:8082/lectures studentId=1 courseId=1 status=requested

# 수강신청 결제
http PATCH http://localhost:8084/payments/1 status="Payment Approved"http localhost:8083/주문처리s orderId=1

# 수강신청 상태 확인
http GET http://localhost:8082/lectures/1

```


## 동기식 호출 과 Fallback 처리

수강신청(Lecture) > 결제(Payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리함.

- 동기식 호출 구현 : FeignClient

```
# (Lecture) Service.java

FeignClient 소스 추가
```

- 동기식 호출
```
# Order.java (Entity)

    소스 추가
```

- 동기식 호출 테스트

```
# 결제서비스 중단

# 수강신청
http POST http://localhost:8082/lectures studentId=1 courseId=1 status=requested   #Fail

# 결제서비스 재기동

# 수강신청
http POST http://localhost:8082/lectures studentId=1 courseId=1 status=requested #Success
```

- Fallback 서비스 구현
```
    소스 추가
```
- Fallback 테스트

```
# ..
```

## 비동기식 호출 테스트

- 비동기식 발신 구현 : publish

```
소스 추가
```

- 비동기식 수신 구현 : PolicyHandler

```
소스 추가
```

- 비동기식 호출 테스트
```
# 수료서비스 중단

# 수강완료
http PATCH http://localhost:8082/lectures/1 status=completed   #Success

# 주문상태 확인
http localhost:8080/orders     # 주문상태 안바뀜 확인

#상점 서비스 기동
cd 상점
mvn spring-boot:run

#주문상태 확인
http localhost:8080/orders     # 모든 주문의 상태가 "배송됨"으로 확인
```


# 운영

## CI/CD 설정

## 장애격리


### 오토스케일 아웃


## 무정지 재배포


## 헥사고날 아키텍처 변화 

