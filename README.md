
# 티켓예약 시스템

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [티켓예약 시스템](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [적용 후 REST API의 테스트](#적용-후-REST-API의-테스트)
    - [비동기식 호출/ 시간적 디커플링/ 장애격리/ 최종(Eventual) 일관성 테스트](#비동기식-호출--시간적-디커플링--장애격리--최종Eventual-일관성-테스트)
    - [API 게이트웨이](#API-)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)


# 서비스 시나리오

기능적 요구사항
1. 티켓시스템에서 admin이 예약가능한 티켓을 등록한다.
1. 예약시스템에서 user가 티켓을 예약한다.
1. 예약이 발생하면 티켓시스템에서 전달받아 admin이 티켓의 예약가능 여부를 확인한다.
1. 티켓 예약이 가능하면 결제시스템에 결제를 요청한다.
1. user는 예약을 취소할 수 있다.
1. user와 admin이 티켓리스트를 중간중간 조회한다.

비기능적 요구사항
1. 트랜잭션
    1. 모든 트랜잭션은 비동기식으로 구성한다. 
1. 장애격리
    1. 티켓시스템 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 admin이 결제 요청을 잠시후에 하도록 유도한다. 고객에게는 Pending상태로 보여준다. Circuit breaker, fallback
1. 성능
    1. user가 예약에 대한 티켓상태를 예약시스템(프론트엔드)에서 티켓리스트로 확인할 수 있어야 한다. CQRS


# 체크포인트

- 분석 설계
  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍  

### 이벤트 도출
![image](https://user-images.githubusercontent.com/12521968/81823357-36b59600-956f-11ea-9b34-a7806425e9c4.png)

### 부적격 이벤트 탈락
    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - 티켓재고확인 이벤트와 티켓예약가능함/티켓예약불가능함은 중복된 과정으로 포괄적인 티켓재고확인 이벤트를 제외함

### 액터, 커맨드, 폴리시 부착
![image](https://user-images.githubusercontent.com/12521968/81826995-4d5dec00-9573-11ea-87b3-3108a6bf9a76.png)

### 어그리게잇, 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/12521968/81827940-39ff5080-9574-11ea-874b-cd64402efa5b.png)

    - 각각 예약, 티켓, 결제 관련해서 연결된 커맨드와 이벤트에 의해 트랜잭션이 유지되어야 하는 단위로 어그리게잇하고 바운더리 컨텍스트를 나눈다. 
    - 도메인 서열 분리 
        - Core Domain:  예약시스템: 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain:   티켓시스템 : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   결제시스템 : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음

### 폴리시의 이동과 컨텍스트 매핑 (붉은 선은 Pub/Sub, 푸른선은 Req/Resp)

![image](https://user-images.githubusercontent.com/12521968/81834698-42f42000-957c-11ea-9cb0-690e0b2057be.jpeg)

### 완성된 모형

![image](https://user-images.githubusercontent.com/12521968/81820429-93af4d00-956b-11ea-8936-a66201bb883b.png)

    - View Model 추가
    - 실시간성보다는 서비스 안전성을 위해 동기식 호출(req/resp)을 비동기식 호출(pub/sub)로 수정

### 완성본에 대한 기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/12521968/81831479-5e5d2c00-9578-11ea-9335-4e4955f3dceb.jpeg)

    - 티켓시스템에서 admin이 예약가능한 티켓을 등록한다. (ok)    
    - 예약시스템에서 user가 티켓을 예약한다. (ok)
    - 예약이 발생하면 티켓시스템에서 전달받아 admin이 티켓의 예약가능 여부를 확인한다. (ok)
    - 티켓 예약이 가능하면 결제시스템에 결제를 요청한다. (ok)
    
![image](https://user-images.githubusercontent.com/12521968/81831553-73d25600-9578-11ea-94c9-d4042be8e07b.jpeg)  

    - user는 예약을 취소할 수 있다. (ok)
    
    - user와 admin이 티켓리스트를 중간중간 조회한다. (ok) - view(ticketlist)의 구현 


### 완성본에 대한 비기능적 요구사항을 커버하는지 검증
    - 트랜잭션 (1)
      . user가 예약한 건에 관해서 admin이 티켓의 상태가 예약가능한 상태인지 확인 후 결제를 진행한다.
    - 장애격리
      . 티켓시스템 기능이 수행되지 않더라도 예약시스템은 정상 작동되고, 결제시스템 기능이 수행되지 않더라도 티켓시스템은 정상적으로 작동된다. Async (event-driven), Eventual Consistency
    - 성능
      . user는 예약시스템(프론트엔드)의 티켓리스트에서 예약 요청한 티켓이 예약된 상태인지 확인할 수 있다. CQRS 



## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/12521968/81839579-b731c200-9582-11ea-8f94-61d7f1c4ccfd.jpg)


# 구현

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다. (각자의 포트넘버는 8081 ~ 808n 이다)

```
# ticket 서비스 //port number: 8081
cd ticket
mvn spring-boot:run

# reservation 서비스 //port number: 8082
cd reservation
mvn spring-boot:run

# payment 서비스 //port number: 8083
cd payment
mvn spring-boot:run
```


## DDD 의 적용

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다(예시는 reservation 마이크로 서비스).  
  이때 가능한 현업에서 사용하는 언어(유비쿼터스 랭귀지)를 그대로 사용하여 모델링시 영문화 하였다.

```  
package ticketreservation;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String ticketid;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Reserved reserved = new Reserved();
        BeanUtils.copyProperties(this, reserved);
        reserved.publish();
    }

    @PostRemove
    public void onPostRemove(){
        Reservationcanceled reservationcanceled = new Reservationcanceled();
        BeanUtils.copyProperties(this, reservationcanceled);
        reservationcanceled.publish();
    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getTicketid() {
        return ticketid;
    }

    public void setTicketid(String ticketid) {
        this.ticketid = ticketid;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}
```

Entity Pattern과 Repository Pattern을 적용하여 JPA를 통하여 다양한 데이터소스 유형(RDB)에 대한 별도의 처리가 없도록, 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST의 RestRepository를 적용하였다.
```
package ticketreservation;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface ReservationRepository extends PagingAndSortingRepository<Reservation, Long>{
}
```


## 적용 후 REST API 의 테스트

기능적 요구사항 시나리오 1. 티켓시스템에서 admin이 예약가능한 티켓을 등록한다.

- ticket 서비스에서 티켓 등록요청
```
http POST localhost:8081/tickets ticketid="1" status="registered"
```
![image](https://user-images.githubusercontent.com/12521968/81847735-f6fea680-958e-11ea-883a-6fb421ab249c.png)

- reservation 서비스 수신확인
![image](https://user-images.githubusercontent.com/12521968/81848139-9f146f80-958f-11ea-9c0c-845f7c1916a5.png)

- reservation 서비스의 ticketlist 저장 확인
```
http localhost:8082/ticketlists/1
```
![image](https://user-images.githubusercontent.com/12521968/81847925-493fc780-958f-11ea-9df5-bf451bb248f9.png)


- local kafka 리스너 확인
```
kafka-console-consumer --bootstrap-server http://localhost:9092 --topic ticketreservation --from-beginning
```
![image](https://user-images.githubusercontent.com/12521968/81848283-d4b95880-958f-11ea-8424-8b10c83b2e14.png)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트
모든 시스템은 장애 상태나 유지보수 상태일 때, 잠시 시스템이 내려가도 작동에 문제가 없게 하기 위해 동기식이 아닌 비동기식으로 처리한다. 그 중 ticket 시스템에서 admin의 티켓 등록이 이루어진 후에 reservation 시스템에 이를 알려주는 행위를 예를 들었다.

- ticket 시스템에 티켓이 등록되었다는 기록을 남긴 후에 티켓등록 이벤트를 카프카로 송출한다.(Publish)
```
    @PostPersist
    public void onPostPersist(){
        Ticketregistered ticketregistered = new Ticketregistered();
        BeanUtils.copyProperties(this, ticketregistered);
        ticketregistered.publishAfterCommit();
    }
```

- reservation 시스템에서는 티켓등록 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록(ticketlist에 저장) PolicyHandler 를 구현한다:
(CQRS)
```
package ticketreservation;

import ticketreservation.config.kafka.KafkaProcessor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;
import java.util.Optional;

@Service
public class TicketlistViewHandler {


    @Autowired
    private TicketlistRepository ticketlistRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void whenTicketregistered_then_CREATE_1 (@Payload Ticketregistered ticketregistered) {
        try {
            if (ticketregistered.isMe()) {
                // view 객체 생성
                Ticketlist ticketlist = new Ticketlist();
                // view 객체에 이벤트의 Value 를 set 함
                ticketlist.setTicketid(ticketregistered.getTicketid());
                // view 레파지 토리에 save
                ticketlistRepository.save(ticketlist);
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whenPaycompleted_then_UPDATE_1(@Payload Paycompleted paycompleted) {
        try {
            if (paycompleted.isMe()) {
                // view 객체 조회
                List<Ticketlist> ticketlistList = ticketlistRepository.findByTicketid(paycompleted.getTicketid());
                for(Ticketlist ticketlist : ticketlistList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    ticketlist.setStatus(paycompleted.getStatus());
                    // view 레파지 토리에 save
                    ticketlistRepository.save(ticketlist);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```


ticket 시스템은 reservation, payment 시스템과 완전히 분리되어 있으며, 이벤트 수신에 따라 처리되기 때문에 시스템이 유지보수로 인해 잠시 내려간 예약을 받아 ticketlist 저장하는데에 아무 문제 없다.


- reservation 시스템을 잠시 내려 놓음

![dv-11](https://user-images.githubusercontent.com/12521968/81849834-0fbc8b80-9592-11ea-8497-571c966e9af9.png)


- 티켓등록 처리
```
http POST localhost:8081/tickets ticketid=”1” status=”registered”   #Success
http POST localhost:8081/tickets ticketid=”2” status=”registered”   #Success
```

- 예약상태 확인
```
http localhost:8081/tickets     
``` 

![dv-12](https://user-images.githubusercontent.com/12521968/81850408-e4866c00-9592-11ea-8195-f5a0ae359378.png)


- 예약 완료 상태까지 Event 진행 확인

![dv-13](https://user-images.githubusercontent.com/12521968/81850486-ff58e080-9592-11ea-9d84-801aa08cc862.png)


- ticket 시스템 재기동 후 ticket 시스템에 Update 되었는지 확인(CQRS)
  고객이 숙소에 예약 신청한 내역을 ticketlist view에서 확인할 수 있다.

![dv-14](https://user-images.githubusercontent.com/12521968/81850843-7f7f4600-9593-11ea-9345-f8669f8dc5f2.png)

![dv-15](https://user-images.githubusercontent.com/63624005/81765184-1b666e80-950e-11ea-8722-60464240fe71.png)



# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, pipeline build script 는 각 프로젝트 폴더 이하에 azure-pipelines.yml 에 포함되었다.

*devops를 활용하여 pipeline을 구성하였고, CI CD 자동화를 구현하였다.

![cicd1](https://user-images.githubusercontent.com/12521968/81890030-efb3b900-95df-11ea-9898-22f631e01210.PNG)

- 아래와 같이 pod 가 정상적으로 올라간 것을 확인하였다.

![cicd2](https://user-images.githubusercontent.com/12521968/81890215-651f8980-95e0-11ea-8fff-be89c9339fee.PNG)

- 아래와 같이 쿠버네티스에 모두 서비스로 등록된 것을 확인할 수 있다.

![cicd3](https://user-images.githubusercontent.com/12521968/81890324-b4fe5080-95e0-11ea-8023-7bee37ac6014.PNG)

### 이벤트 정상작동 확인

```
root@httpie:/# http POST ticket:8080/tickets ticketid="1"
```
![http1](https://user-images.githubusercontent.com/12521968/81891156-b9c40400-95e2-11ea-93fc-15d077991a0b.PNG)

```
root@httpie:/# http reservation:8080/ticketlists/1
```
![http2](https://user-images.githubusercontent.com/12521968/81891224-dbbd8680-95e2-11ea-9ecb-21d9ab616756.PNG)


### 오토스케일 아웃

시스템을 안정되게 운영할 수 있도록 보완책으로 자동화된 확장 기능을 적용하고자 한다.

- 안정성이 중요한 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 5개까지 늘려준다:

```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=15
kubectl get all
```
![auto](https://user-images.githubusercontent.com/12521968/81891270-f98aeb80-95e2-11ea-8201-60e927bfc38f.PNG)



- 워크로드를 동시사용자 10명으로 20초 동안 걸어준다.

```
kubectl exec -it httpie bin/bash
siege -c2 -t20S  -v --content-type "application/json" 'http://payment:8080/payments POST {"ticketid":2,"status":"Y"}'
```

## 무정지 재배포


