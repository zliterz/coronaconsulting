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

책 대여(Rental) Transaction 발생 후 myPage 조회 결과 

- 예치금 적립

![image](https://user-images.githubusercontent.com/89369983/133173724-2be57ffb-2b53-4dec-ad46-1f82df895192.png)

- 도서 대여

![image](https://user-images.githubusercontent.com/89369983/133173763-cc9effde-4845-4fa1-9d85-8d2782dd5141.png)

- mypage 대여 현황 조회

![image](https://user-images.githubusercontent.com/89369983/133173774-563b0f9b-0538-4345-b14c-369b7c610e95.png)



## 폴리글랏 퍼시스턴스
mypage 서비스의 DB와 Rental/Payment/Point 서비스의 DB를 다른 DB를 사용하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다.

|서비스|DB|pom.xml|
| :--: | :--: | :--: |
|Request| H2 |![image](https://user-images.githubusercontent.com/88808251/135046054-5bfa2e3a-63d0-453c-ac8b-e9c678a9ff0c.png)|
|Consulting| H2 |![image](https://user-images.githubusercontent.com/88808251/135046054-5bfa2e3a-63d0-453c-ac8b-e9c678a9ff0c.png)|
|Teacher| H2 |![image](https://user-images.githubusercontent.com/88808251/135046054-5bfa2e3a-63d0-453c-ac8b-e9c678a9ff0c.png)|
|MyPage| HSQL |![image](https://user-images.githubusercontent.com/88808251/135045637-045ab8d9-683c-4314-b39b-1418a5032cd7.png)|


## 동기식 호출과 Fallback 처리
책 대여를 위해서는 사용자 예치금이 적립되어 있어야 하며, 예치금은 책대여 금액 이상 적립되어 있어야 하는 요구사항이 있음
해당 처리는 동기 호출이 필요하다고 판단하여 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 구현 하였음 

Rental 서비스 내 PointService

```java
package book.rental.system.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;


@FeignClient(name="Point", url="http://${api.url.Point}:8080")
public interface PointService {
    @RequestMapping(method= RequestMethod.GET, path="/points/checkPoint")
    public boolean checkPoint(@RequestParam Long customerId, @RequestParam Long price);
    
}
```

Rental 서비스 내 Req/Resp

```java
    @PostPersist
    public void onPostPersist() throws Exception{
        // 예치금이 책대여비용이상 보유하고 있는지 점검
        if(RentalApplication.applicationContext.getBean(book.rental.system.external.PointService.class)
        .checkPoint(this.customerId, this.price)){
            BookRented bookRented = new BookRented();
            BeanUtils.copyProperties(this, bookRented);
            bookRented.publishAfterCommit();
        }
        else{
            throw new Exception("Customer Point Check Exception !!");
        }

    }
```

Point 서비스 내 Rental 서비스 Feign Client 요청 대상

```java
  @RestController
 public class PointController {

    @Autowired
    PointRepository pointRepository;

    @RequestMapping(value = "/points/checkPoint",
        method = RequestMethod.GET,
        produces = "application/json;charset=UTF-8")
    public boolean checkPoint(HttpServletRequest request, HttpServletResponse response) {
        boolean status = false;

        Long customerId = Long.valueOf(request.getParameter("customerId"));
        Long price = Long.valueOf(request.getParameter("price"));
        
        Optional<Point> point = pointRepository.findByCustomerId(customerId);
        if(point.isPresent()){
            Point pointValue = point.get();
            System.out.println("##### /point/checkPoint  pointValue.getPoint() : #####"+pointValue.getPoint());
            //Point 가 차감포인트 보다 큰지 점검 
            if(pointValue.getPoint() - price > 0) {
                status = true;
            }
        }

        return status;
    }
```

동작 확인


책대여 요청 시 가용 예치금이 있으면 대여 처리됨 
![saga2](https://user-images.githubusercontent.com/33479996/133034467-b75bd437-f5f7-40f7-8d0c-5fc79abca509.PNG)



책대여 요청 시 가용 예치금이 부족하면 대여 처리 안됨 
![image](https://user-images.githubusercontent.com/89369983/133175765-10ee24d6-36da-483f-89a1-c25a1f7c11c1.png)


## SAGA패턴 

SAGA 패턴은 각 서비스의 트랜잭션은 단일 서비스 내의 데이터를 갱신하는 일종의 로컬 트랜잭션 방법이고 서비스의 트랜잭션이 완료 후에 다음 서비스가 트리거 되어, 트랜잭션을 실행하는 방법입니다.

현재 도서 대여 시스템에도 SAGA 패턴에 맞추어서 작성되어 있다.

**SAGA 패턴에 맞춘 트랜잭션 실행**

![saga1](https://user-images.githubusercontent.com/33479996/133033751-af4b9595-276a-4b9d-9148-4e4a0bc46b61.PNG)

현재 도서 대여 시스템은 SAGA 패턴에 맞추어서 rental 서비스의 rental 생성이 완료되면 Payment 서비스를 트리거하게 되어 payment를 업데이트하여
rental 서비스에서 주문을 수신하게 작성되어 있다.

아래와 같이 실행한 결과이다.

![saga2](https://user-images.githubusercontent.com/33479996/133034467-b75bd437-f5f7-40f7-8d0c-5fc79abca509.PNG)

위와 같이 rental 서비스에서 주문을 생성하게 될 경우 아래와 같이 Payment 서비스에서 payment 상태를 업데이트 하게 된다. 

![SAGA3](https://user-images.githubusercontent.com/33479996/133034677-5998fe8a-ae92-4bc7-91ba-485580d1f8db.PNG)


위와 같이 Payment 서비스에서 상태를 업데이트 하면서 이벤트를 발신하게 되고 이를 수신 받은 MyPage 서비스에서 Point를 아래와 같이 차감하게 된다.

![SAGA4](https://user-images.githubusercontent.com/33479996/133035070-22465fee-236b-465e-a135-876c184b715a.PNG)

![SAGA5](https://user-images.githubusercontent.com/33479996/133035437-dab9c7ce-4316-4d64-af03-a4458ac163d3.PNG)


# 운영

## Deploy/ Pipeline

- yml 파일 이용한 Docker Image Push/deploy/서비스생성

```sh

cd gateway
az acr build --registry rentbook --image grp03.azurecr.io/gateway:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd Rental
az acr build --registry rentbook --image grp03.azurecr.io/rental:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd Point
az acr build --registry rentbook --image grp03.azurecr.io/point:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd Payment
az acr build --registry rentbook --image grp03.azurecr.io/payment:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml


cd ..
cd book
az acr build --registry rentbook --image grp03.azurecr.io/book:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml


cd ..
cd View
az acr build --registry rentbook --image grp03.azurecr.io/view:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

```

## Config Map
- 변경 가능성이 있는 항목은 ConfigMap을 사용하여 설정 변경 할수 있도록 구성   
  - Rental 서비스에서 바라보는 Point 서비스 url 일부분을 ConfigMap 사용하여 구현 함

configmap 설정 및 소스 반영

![image](https://user-images.githubusercontent.com/89369983/133117696-5f3b4fd7-e850-4caa-901b-b3171baca69b.png)

configmap 구성 내용 조회

![image](https://user-images.githubusercontent.com/89369983/133117908-934c3b40-99a2-4905-9963-6692d6c82a28.png)


## Autoscale (HPA)
프로모션 등으로 사용자 유입이 급격한 증가시 안정적인 운영을 위해 Rental 자동화된 확장 기능을 적용 함

Resource 설정 및 유입전 현황 
![image](https://user-images.githubusercontent.com/89369983/133118106-ab6141ad-bd66-40bd-a14c-bcf21ecf5e23.png)

- rental 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정(CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘리도록 설정)

```sh
$ kubectl autoscale deploy booking --min=1 --max=10 --cpu-percent=15
```

서비스에 Traffic 유입(siege를 통해 워크로드 발생)

![image](https://user-images.githubusercontent.com/89369983/133118622-1a8e337b-b522-44fa-81c2-4b67877144d3.png)

부하테스트 결과 HPA 반영
![image](https://user-images.githubusercontent.com/89369983/133118351-4315f1b0-85b9-46ea-b23a-9d90ac21f6d5.png)

## Circuit Breaker
  * Istio 다운로드 및 PATH 추가, 설치
  * rentbook namespace에 Istio주입
![image](https://user-images.githubusercontent.com/89369983/133118751-c6ce8e89-a0ba-4655-bd7d-3da68f269bed.png)

  * Transaction이 과도할 경우 Circuit Breaker 를 통하여 장애격리 처리
![image](https://user-images.githubusercontent.com/89369983/133118831-9e6c5580-bc7a-4690-9084-250fcfbb47ee.png)


## Zero-Downtime deploy (Readiness Probe)

  * readiness 미 설정 상태에서, 배포중 siege 테스트 진행 
  - rental 서비스 배포 중 정상 실행중 서비스 요청은 성공(201), 배포중인 서비스에 요청은 실패 (503 ) 확인
![image](https://user-images.githubusercontent.com/89369983/133118944-e973c8c8-6e3c-4072-9e3c-f6dff07b56bc.png)

  * deployment.yml에 readiness 설정 및 적용 후 siege 테스트 진행시 안정적인 서비스 응답확인
![image](https://user-images.githubusercontent.com/89369983/133119028-cdc334ef-72e0-43ac-a603-9a38ee5e0ed8.png)


    
## Self-healing (Liveness Probe)

  * Liveness Probe 설정 및 restart 시도 점검을 위해 잘못된값 설정으로 변경 후 RESTARTS 처리 테스트 결과
![image](https://user-images.githubusercontent.com/89369983/133119195-0aee658f-fe25-46dd-abc2-2b125372f55c.png)


