# CoronaConsulting
코로나시대 학생 상담 시스템 // 08422 심우현 개인과제

![coronaconsulting](https://user-images.githubusercontent.com/88808251/134907278-92578800-cdc1-46ba-ab37-fa12f1d9f886.png)

# Table of contents

- [도서 대여 관리 시스템](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)


# 서비스 시나리오

기능적 요구사항

A. 요청
1.	학생이 상담을 위해 선생님을 조회할 수 있다.
2.	학생이 상담 신청 시 오프라인이거나 상담중 상태이면 상담이 시작하지 않는다.
3.	학생은 선생님을 즐겨찾기 할 수 있고 선생님의 인기count가 올라간다.
4.	선생님이 상담시 해당 선생님은 상담 불가능한 상태로 변경된다.
5.	상담이 끝나면 선생님은 상담 가능한 상태로 변경된다.
  
B. 상담
1.	상담 후 비디오 파일로 저장되고 학생은 다시 볼 수 있다.
2.	상담은 선생님이 종료할 수 있다.
3.	상담이 끝나면 선생님은 상담 count가 올라간다.
  
C. 선생님
1.	선생님은 접속하게 되면 상담 대기하게 되고 종료하면 오프라인이 된다.
2.	선생님이 온라인상태가 될 때 즐겨찾기한 학생에게 알림이 간다.
3.	선생님의 소개정보를 등록할 수 있다.
  
  
비기능적 요구사항
1.	트랜잭션  
i.	선생님이 Online상태가 아니면 상담이 불가하다. (Sync 호출)  
2.	장애격리  
i.	상담 기능이 수행되지 않더라도 신청 기능은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency  
ii.	신청시스템이 과중되면 사용자를 잠시동안 받지 않고 신청 진행을 잠시후에 하도록 유도한다 Circuit breaker, fallback  
3.	성능  
i.	학생은 상담현황을 조회 할 수 있어야 한다 CQRS  
ii.	즐겨찾기한 선생님이 온라인 상태가 되면 학생에게 카톡 등으로 알림을 줄 수 있어야 한다 Event driven  



# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
![image](https://user-images.githubusercontent.com/89369983/133180330-14093197-2864-4b9c-8e5b-6ec6555e49f5.png)

## TO-BE 조직 (Vertically-Aligned)
![tobe](https://user-images.githubusercontent.com/88808251/135012695-46b05be3-79f2-4af3-a99f-9eee6e6dd855.png)

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  https://labs.msaez.io/#/storming/oRS26n0xXJVnWLBNfmup2aNvFtb2/3a5c2c0715fca64cfc1fcd833bbb2858


### 이벤트 도출
![event도출](https://user-images.githubusercontent.com/88808251/134918063-8df61b12-ba52-48a9-988c-ef84b2aaff3d.png)

### 부적격 이벤트 탈락
![부적격event](https://user-images.githubusercontent.com/88808251/134919018-7c324b29-92a7-4fbf-8bcf-a12a572f0ae4.png)


### 액터, 커맨드 부착하여 읽기 좋게
![action](https://user-images.githubusercontent.com/88808251/134998195-ffd37bdc-a11b-42e0-ab0f-a0840ba082c4.png)

### 어그리게잇으로 묶기
![aggregate](https://user-images.githubusercontent.com/88808251/134998585-1ae62a7e-d086-4a2f-b045-4532ba839fee.png)

### 바운디드 컨텍스트로 묶기
![boundedct](https://user-images.githubusercontent.com/88808251/134999161-d3fdf9e6-ed78-42ee-8aa9-477d602fbf13.png)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)
![policy](https://user-images.githubusercontent.com/88808251/135000918-4030b4dd-b488-4fcb-8817-67a8f513048d.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![policy_ct](https://user-images.githubusercontent.com/88808251/135001394-d736343e-5ab0-4eef-a821-347ba02ab7c5.png)

### 완성된 1차 모형
![complete](https://user-images.githubusercontent.com/88808251/135001570-01184fec-b58b-4e45-9d26-7356fad095b8.png)

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
![검증1](https://user-images.githubusercontent.com/88808251/135002618-f0c1dfe1-4349-40f4-9480-98e30ca94bc4.png)
![검증2](https://user-images.githubusercontent.com/88808251/135002555-e8b6c1fd-07ec-49cb-9c0f-158f0d4b8591.png)


## 헥사고날 아키텍처 다이어그램 도출
    
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리
    
![image](https://user-images.githubusercontent.com/88808251/135005740-574f1c77-e9aa-4d39-bda0-b6c2c8498444.png)



# 구현

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라,구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084이다)

```shell
cd Request
mvn spring-boot:run

cd Consulting
mvn spring-boot:run 

cd Teacher
mvn spring-boot:run 

cd MyPage 
mvn spring-boot:run

cd gateway
mvn spring-boot:run 
```
## DDD(Domain-Driven-Design)의 적용
msaez Event-Storming을 통해 구현한 Aggregate 단위로 Entity 를 정의 하였으며,
Entity Pattern 과 Repository Pattern을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.

Bookrental 서비스의 rental.java

```java

package book.rental.system;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Rental_table")
public class Rental {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long rentalId;
    private Integer bookId;
    private String bookName;
    private Integer price;
    private Date startDate;
    private Date returnDate;
    private Integer customerId;
    private String customerPhoneNo;
    private String rentStatus;

    @PostPersist
    public void onPostPersist(){

        //  서적 대여 시 상태변경 후 Publish 
        BookRented bookRented = new BookRented();
        BeanUtils.copyProperties(this, bookRented);
        bookRented.publishAfterCommit();

    }

    @PostUpdate 
    public void onPostUpdate(){

        if("RETURN".equals(this.rentStatus)){           // 반납 처리 Publish
            BookReturned bookReturned = new BookReturned();
            BeanUtils.copyProperties(this, bookReturned);
            bookReturned.publishAfterCommit();

        } else if("DELAY".equals(this.rentStatus)){     // 반납지연 Publish
            ReturnDelayed returnDelayed = new ReturnDelayed();
            BeanUtils.copyProperties(this, returnDelayed);
            returnDelayed.publishAfterCommit();
        }
    }    

    public Long getRentalId() {
        return rentalId;
    }

    public void setRentalId(Long rentalId) {
        this.rentalId = rentalId;
    }
    public Integer getBookId() {
        return bookId;
    }

    public void setBookId(Integer bookId) {
        this.bookId = bookId;
    }
    public String getBookName() {
        return bookName;
    }
    .. getter/setter Method 생략
```

 Payment 서비스의 PolicyHandler.java
 rental 완료시 Payment 이력을 처리한다.
```java
package book.rental.system;

import book.rental.system.config.kafka.KafkaProcessor;

import java.util.Optional;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{
    @Autowired PaymentRepository paymentRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverBookRented_PayPoint(@Payload BookRented bookRented){

        if(!bookRented.validate()) return;

        if("RENT".equals(bookRented.getRentStatus())){

            Payment payment =new Payment();

            payment.setBookId(bookRented.getBookid());
            payment.setCustomerId(bookRented.getCustomerId());
            payment.setPrice(bookRented.getPrice());
            payment.setRentalId(bookRented.getRentalId());
            paymentRepository.save(payment);
        }else{
            System.out.println("\n\n##### listener PayPoint Process Failed : Status -->" +bookRented.getRentStatus() + "\n\n");
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whatever(@Payload String eventString){}


}

```

 BookRental 서비스의 RentalRepository.java


```java
package book.rental.system;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="rentals", path="rentals")
public interface RentalRepository extends PagingAndSortingRepository<Rental, Long>{


}
```

## 적용 후 REST API 의 테스트
각 서비스들의 Rest API 호출을 통하여 테스트를 수행하였음

```shell
책 대여 처리
http post localhost:8081/rent bookId=1 price=1000 startDate=20210913 returnDate=20211013 customerId=1234 customerPhoneNo=01012345678 rentStatus=RENT

책 대여를 위한 예치금 적립
http post localhost:8086/point customerId=1234 point=10000

책 등록 
http post localhost:8082/book bookId=1 price=1000 bookName=azureMaster
```

## Gateway 적용
GateWay 구성를 통하여 각 서비스들의 진입점을 설정하여 라우팅 설정하였다.
```yaml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: Request
          uri: http://localhost:8081
          predicates:
            - Path=/requests/** 
        - id: Consulting
          uri: http://localhost:8082
          predicates:
            - Path=/consultings/** 
        - id: Teacher
          uri: http://localhost:8083
          predicates:
            - Path=/teachers/** 
        - id: MyPage
          uri: http://localhost:8084
          predicates:
            - Path= /myPages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: Request
          uri: http://Request:8080
          predicates:
            - Path=/requests/** 
        - id: Consulting
          uri: http://Consulting:8080
          predicates:
            - Path=/consultings/** 
        - id: Teacher
          uri: http://Teacher:8080
          predicates:
            - Path=/teachers/** 
        - id: MyPage
          uri: http://MyPage:8080
          predicates:
            - Path= /myPages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
```
## CQRS 적용
mypage(View)는 Materialized View로 구현하여, 타 마이크로서비스의 데이터 원본에 Join SQL 등 구현 없이도 내 서비스의 화면 구성과 잦은 조회가 가능하게 구현 하였음.

상담시작(Consulting) Transaction 발생 후 myPage 조회 결과 

- 상담시작

![image](https://user-images.githubusercontent.com/88808251/135192383-58c9e215-0e42-491e-ab54-6477f71f1f12.png)

- mypage 상담 현황 조회

![image](https://user-images.githubusercontent.com/88808251/135192397-8784dcb6-cf40-4f43-9b10-24fedb4ce337.png)



## 폴리글랏 퍼시스턴스
mypage 서비스의 DB와 Rental/Payment/Point 서비스의 DB를 다른 DB를 사용하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다.

|서비스|DB|pom.xml|
| :--: | :--: | :--: |
|Request| H2 |![image](https://user-images.githubusercontent.com/88808251/135046054-5bfa2e3a-63d0-453c-ac8b-e9c678a9ff0c.png)|
|Consulting| H2 |![image](https://user-images.githubusercontent.com/88808251/135046054-5bfa2e3a-63d0-453c-ac8b-e9c678a9ff0c.png)|
|Teacher| H2 |![image](https://user-images.githubusercontent.com/88808251/135046054-5bfa2e3a-63d0-453c-ac8b-e9c678a9ff0c.png)|
|MyPage| HSQL |![image](https://user-images.githubusercontent.com/88808251/135045637-045ab8d9-683c-4314-b39b-1418a5032cd7.png)|


# 운영

## Deploy/ Pipeline

- yml 파일 이용한 Docker Image Push/deploy/서비스생성

```sh

cd gateway
az acr build --registry rentbook --image user1010.azurecr.io/gateway:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd Request
az acr build --registry rentbook --image user1010.azurecr.io/request:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd Consulting
az acr build --registry rentbook --image user1010.azurecr.io/consulting:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd Teacher
az acr build --registry rentbook --image grp03.azurecr.io/teacher:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml


cd ..
cd MyPage
az acr build --registry rentbook --image user1010.azurecr.io/mypage:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

```
![image](https://user-images.githubusercontent.com/88808251/135107129-25fd1f90-e2dd-4a8d-af12-16bb4ba6b333.png)

## Config Map
- 변경 가능성이 있는 항목은 ConfigMap을 사용하여 설정 변경 할수 있도록 구성   
  - Rental 서비스에서 바라보는 Point 서비스 url 일부분을 ConfigMap 사용하여 구현 함

configmap 설정 및 소스 반영

![image](https://user-images.githubusercontent.com/88808251/135128231-c955161c-0761-4b57-aa36-a46dc4909edd.png)

configmap 구성 내용 조회

![image](https://user-images.githubusercontent.com/88808251/135126592-878fcb79-ee1e-4fa2-9796-3621ef064d20.png)


## Autoscale (HPA)
프로모션 등으로 사용자 유입이 급격한 증가시 안정적인 운영을 위해 Rental 자동화된 확장 기능을 적용 함

Resource 설정 및 유입전 현황 
![image](https://user-images.githubusercontent.com/88808251/135133397-4e2796d5-aaf2-43e8-8a33-191fa7b63c45.png)

- request 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정(CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘리도록 설정)

```sh
$ kubectl autoscale deploy request --min=1 --max=10 --cpu-percent=15
```

서비스에 Traffic 유입(siege를 통해 워크로드 발생)

![image](https://user-images.githubusercontent.com/88808251/135136956-f56248e9-5f14-4a53-8ae6-b23d0db6cf26.png)

부하테스트 결과 HPA 반영  
![image](https://user-images.githubusercontent.com/88808251/135136863-1de5f0d4-7bed-4382-a60e-0f5e05ea2e43.png)

## Circuit Breaker
  * Istio 다운로드 및 PATH 추가, 설치
  * rentbook namespace에 Istio주입
  * istio yaml파일로 설치
![image](https://user-images.githubusercontent.com/88808251/135148262-9b004236-72f6-4131-b97d-b3d829b14c26.png)
![image](https://user-images.githubusercontent.com/88808251/135203705-8ea220f7-d143-4bb0-acc0-e9f9ad65f8ec.png)

  * Transaction이 과도할 경우 Circuit Breaker 를 통하여 장애격리 처리
![image](https://user-images.githubusercontent.com/88808251/135203482-c76e5011-582c-4721-89b2-672ac8ff2f03.png)


## Zero-Downtime deploy (Readiness Probe)

  * readiness 미 설정 상태에서, 배포중 siege 테스트 진행 
  - request 서비스 배포 중 정상 실행중 서비스 요청은 성공(201), 배포중인 서비스에 요청은 실패 (503 ) 확인
![image](https://user-images.githubusercontent.com/88808251/135155004-dfc80bfa-ed43-451e-b453-628e9008b785.png)

  * deployment.yml에 readiness 설정 및 적용 후 siege 테스트 진행시 안정적인 서비스 응답확인
![image](https://user-images.githubusercontent.com/88808251/135154956-c335d9cf-1315-4bc3-b6c0-deb2e26d3f98.png)


    
## Self-healing (Liveness Probe)

  * Liveness Probe 설정 및 restart 시도 점검을 위해 잘못된값 설정으로 변경 후 RESTARTS 처리 테스트 결과
  - 설정전, restart 횟수 증가
![image](https://user-images.githubusercontent.com/88808251/135204058-61a24185-0cb0-43ab-8357-d9d205caed2e.png)  
- 설정후, 설정전의 Pod는 리스타트 끝에 CrashLoopBackOff, 새로 생성한 Pod는 정상 Running
![image](https://user-images.githubusercontent.com/88808251/135204339-72db98ee-96d3-49d7-ae49-894de9a1b0cf.png)
![image](https://user-images.githubusercontent.com/88808251/135204329-fc380e72-e1a4-4c96-bd68-bdd8cf8a456f.png)
![image](https://user-images.githubusercontent.com/88808251/135204346-2ca3a7b3-9792-4603-8f74-0f6d015ce8dc.png)

