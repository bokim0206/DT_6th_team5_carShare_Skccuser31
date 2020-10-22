# DT_6th_team5_carShare (자동차 공유 서비스)

5팀 자동차 공유 서비스 CNA개발 실습을 위한 프로젝트

# 구현 Repository
 1. 접수관리 : https://github.com/srwzz/carShareOrder.git
 1. 결제관리 : https://github.com/srwzz/carSharePayment.git
 1. 배송관리 : https://github.com/srwzz/carShareDelivery.git
 1. 고객페이지 : https://github.com/srwzz/carShareStatusview.git
 1. 게이트웨이 : https://github.com/srwzz/carShareGateway.git
 1. (추가)재고관리 : https://github.com/srwzz/carShareStock.git


# Table of contents

- [서비스 시나리오](#서비스-시나리오)
  - [시나리오 테스트결과](#시나리오-테스트결과)
- [분석/설계](#분석설계)
- [구현](#구현)
  - [DDD 의 적용](#ddd-의-적용)
  - [Gateway 적용](#Gateway-적용)
  - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
  - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
  - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
- [운영](#운영)
  - [CI/CD 설정](#cicd설정)
  - [서킷 브레이킹 / 장애격리](#서킷-브레이킹-/-장애격리)
  - [오토스케일 아웃](#오토스케일-아웃)
  - [무정지 재배포](#무정지-재배포)
  

# 서비스 시나리오

## 기능적 요구사항
1. 고객이 공유차를 선택하여 렌탈한다.
1. 고객이 결제하여 접수한다.
1. 업체가 공유차를 고객위치로 가져다놓는다.
1. 고객이 렌탈을 취소할 수 있다.
1. 렌탈이 취소되면 배송이 취소된다.
1. 고객이 자신의 렌탈 정보를 조회한다.
1. (추가)주문 접수시 재고 증감

## 비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 접수가 성립되지 않아야 한다(Sync 호출)
    1. (추가)주문 접수시 재고가 0인경우 접수가 성립죄지 않아야 한다(Sync 호출)
1. 장애격리
    1. 배송관리 기능이 수행되지 않더라도 접수는 정상적으로 처리 가능하다(Async(event-driven), Eventual Consistency)
    1. 접수시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다(Circuit breaker, fallback)
    1. (추가)주문 취소시 재고 관리가 되지 않더라도 접수는 취소되어야 한다.(Async(event-driven), Eventual Consistency)
1. 성능
    1. 고객이 본인의 렌탈 상태 및 이력을 접수시스템에서 확인할 수 있어야 한다(CQRS)
    1. (추가)고객이 접수 후 재고 확인을 할 수 있다(CQRS)


# 분석/설계
 AS-IS 조직 (Horizontally-Aligned) -> TO-BE 조직 (Vertically-Aligned)

## 이벤트스토밍
* MSAEz 로 모델링한 이벤트스토밍 결과:  

![image](https://user-images.githubusercontent.com/16017769/96807026-8b2f3c80-1450-11eb-855c-eb872de98dbd.png)

![image](https://user-images.githubusercontent.com/16017769/96681709-af3c4080-13b2-11eb-9945-f825a14ee5c8.png)

### 완성본에 대한 비기능적 요구사항을 커버하는지 검증
    1. 트랜잭션
    - 고객의 주문에 따라 재고가 감소된다. > Sync
    - 고객의 주문취소에 따라 재고가 증가된다. > Async
    2. 장애격리
    - 재고 관리에 주문 취소는 정상적으로 처리 가능하다 > Async(event driven)
    - 서킷 브레이킹 프레임워크 > istio-injection + DestinationRule
    3. 성능
    - 고객은 본인의 예약의 재고 정보를 확인할 수 있다 > CQRS

## 헥사고날 아키텍처 다이어그램 도출

![image](https://user-images.githubusercontent.com/16017769/96807067-a0a46680-1450-11eb-92cd-141f4cf40b97.png)

![image](https://user-images.githubusercontent.com/16017769/96682871-608fa600-13b4-11eb-9726-1e8a3da71314.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

   
# 구현
## DDD 의 적용
분석/설계 단계에서 도출된 MSA는 총 5개(stock 추가)로 아래와 같다.
* customerpage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|---|---|
| order | 접수 관리 | 8081 | http://localhost:8081/orders |http://carshareorder:8080/orders |
| delivery | 배송 관리 | 8082 | http://localhost:8082/deliveries | http://carsharedelivery:8080/deliveries |
| customerpage | 상태 조회 | 8083 | http://localhost:8083/customerpages | http://carsharestatusview:8080/customerpages |
| payment | 결제 관리 | 8084 | http://localhost:8084/payments | http://carsharepayment:8080/payments |
| stock | 재고 관리 | 8085 | http://localhost:8085/stocks | http://carsharestock:8080/stocks |

## Gateway 적용
```
carShareGateway의 application.yml 에 stock 서비스 추가

```
![image](https://user-images.githubusercontent.com/16017769/96737608-d453a200-13f8-11eb-8aa6-4d8e04795912.png)

## 폴리글랏 퍼시스턴스

CQRS 를 위한 stock 서비스는 인메모리 DB인 hsqldb 사용 적용 함.

```
pom.xml 에 적용

```
![image](https://user-images.githubusercontent.com/16017769/96690883-31cafd00-13bf-11eb-9513-aa4375aaf265.png)


## 동기식 호출 과 Fallback 처리
분석단계에서의 조건 중 하나로 접수(order)->stock 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

FeignClient 서비스 구현
```
@FeignClient(name="stock", contextId = "stock", url="${api.stock.url}", fallback =StockServiceFallback.class)
public interface StockService {

    @RequestMapping(method= RequestMethod.POST, path="/stocks")
    public void reduce(@RequestBody Stock stock);


```
접수요청을 받은 직후(@PostPersist) 재고 서비스를 요청하도록 처리
```
        carshare.external.Stock stock = new carshare.external.Stock();
        stock.setOrderId(this.getId());
        OrderApplication.applicationContext.getBean(carshare.external.StockService.class)
            .reduce(stock);

```

동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 재고 서비스가 장애가 나면 접수요청 못받는다는 것을 확인

```
#stock 서비스를 잠시 내려놓음
#접수요청 처리
http localhost:8081/orders productId=1001 qty=1 status="order" stock=1  #Fail
http localhost:8081/orders productId=1002 qty=3 status="order" stock=1  #Fail

#결제 서비스 재기동
cd carsharepayment
mvn spring-boot:run

#접수요청 처리 성공
http localhost:8081/orders productId=1001 qty=1 status="order" stock=1  #Success
http localhost:8081/orders productId=1002 qty=3 status="order" stock=1  #Success
```


##  비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트
오더 취소가 이루어진 후에 재소 증가는 동기식이 아니라 비동기식으로 처리하여 재고 시스템의 처리를 위해 주문 취소가 블로킹되지 않도록 처리한다.
이를 위하여 취소 기록을 남긴 후에 곧바로 취소 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)

```
@Entity
@Table(name="Order_table")
public class Order {

 ...
    @PostPersist
    public void onPostPersist(){
        Ordercanlled ordercanlled = new Ordercanlled();
        BeanUtils.copyProperties(this, ordercanlled);
        ordercanlled.publishAfterCommit();    
    }
}
```

Stock에서는 취소 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

```

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderCancelled_Add(@Payload OrderCancelled orderCancelled){

        if(orderCancelled.isMe()){
            Stock stock = new Stock();
            stock.setOrderId(paid.getOrderId());
            stockRepository.save(stock) ;
            
        }
    }
```

오더 취소 서비스는 재고관리와 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 재고 서비스가 내려간 상태라도 오더 취소는 문제가 없다.


# 운영

## CI/CD 설정

stock에 대해 repository를 구성하였고, CI/CD플랫폼은 AWS의 CodeBuild를 사용했다.
Git Hook 설정으로 연결된 GitHub의 소스 변경 발생 시 자동 배포된다.
![image](https://user-images.githubusercontent.com/16017769/96723971-38229e80-13ea-11eb-997c-d21ac945deab.png)


## 서킷 브레이킹 / 장애격리


### 서킷 브레이킹 istio-injection + DestinationRule

* istio-injection 적용 (기 적용완료)
```
kubectl label namespace carshare istio-injection=enabled 
```

* 서킷 브레이커 pending time 설정
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order
  namespace: carshare
spec:
  host: carshareorder
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 30
        maxRequestsPerConnection: 100
    outlierDetection:
      interval: 5s
      consecutiveErrors: 1
      baseEjectionTime: 5m
      maxEjectionPercent: 100
EOF
```


* 부하테스트 툴(Siege) 설치 및 Order 서비스 Load Testing 
  - 동시 사용자 5명
  - 2초 실행 
  siege -c5 -t2S -v http://a78a3fd18616c45ca8aed88956de4fee-222272609.ap-southeast-2.elb.amazonaws.com:8080/stocks
```
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /stocks
HTTP/1.1 200     0.04 secs:     360 bytes ==> GET  /stocks
HTTP/1.1 200     0.04 secs:     360 bytes ==> GET  /stocks
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /stocks
HTTP/1.1 200     0.05 secs:     360 bytes ==> GET  /stocks
HTTP/1.1 200     0.05 secs:     360 bytes ==> GET  /stocks
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /stocks
HTTP/1.1 503     0.02 secs:      81 bytes ==> GET  /stocks
HTTP/1.1 200     0.04 secs:     360 bytes ==> GET  /stocks
HTTP/1.1 503     0.02 secs:      81 bytes ==> GET  /stocks
HTTP/1.1 503     0.02 secs:      81 bytes ==> GET  /stocks
HTTP/1.1 503     0.02 secs:      81 bytes ==> GET  /stocks
HTTP/1.1 200     0.03 secs:     360 bytes ==> GET  /stocks

Lifting the server siege...
Transactions:                     87 hits
Availability:                  65.91 %
Elapsed time:                   1.12 secs
Data transferred:               0.03 MB
Response time:                  0.06 secs
Transaction rate:              77.68 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                    4.85
Successful transactions:          87
Failed transactions:              45
Longest transaction:            0.10
Shortest transaction:           0.01
 ```

  
* 키알리(kiali)화면에 서킷브레이커 동작 확인


![image](https://user-images.githubusercontent.com/16017769/96751257-e983fd00-1407-11eb-8884-28e51b516dcd.png)

![image](https://user-images.githubusercontent.com/16017769/96751289-f56fbf00-1407-11eb-9200-1d3b195f17e8.png)



### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 
Targets unkown 처리 불가로 미처리

![image](https://user-images.githubusercontent.com/16017769/96752998-336de280-140a-11eb-90d0-467d3cfa5dbe.png)

## 무정지 재배포
- Readiness Probe 및 Liveness Probe 설정(buildspec.yml 설정)

![image](https://user-images.githubusercontent.com/42608068/96593140-24146980-1324-11eb-88d5-7dee61001832.png)

### Readiness Probe 설정
- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함 Git hook 연동 설정되어 Github의 소스 변경 발생 시 자동 빌드 배포됨
![image](https://user-images.githubusercontent.com/16017769/96661148-5c4c9400-1386-11eb-8f4f-9b83cab19b8c.png)


## Liveness Probe
```
미처리
```

## ConfigMap 사용

시스템별로 또는 운영중에 동적으로 변경 가능성이 있는 설정들을 ConfigMap을 사용하여 관리합니다.
Application에서 특정 도메일 URL을 ConfigMap 으로 설정하여 운영/개발등 목적에 맞게 변경가능합니다.  

* my-config.yaml
![image](https://user-images.githubusercontent.com/16017769/96756286-d45e9c80-140e-11eb-86bd-863486e95a54.png)

my-config라는 ConfigMap을 생성하고 key값에 도메인 url을 등록한다. 

* carsharestock/buildsepc.yaml (configmap 사용)

![image](https://user-images.githubusercontent.com/16017769/96756046-75992300-140e-11eb-879d-845e3e70e122.png)

Deployment yaml에 해단 configMap 적용

* StockService.java
```
@FeignClient(name="stock", contextId = "stock", url="${api.stock.url}")
public interface StockService {

    @RequestMapping(method= RequestMethod.POST, path="/stocks")
    public void stock(@RequestBody Stock stock);

}
```
url에 configMap 적용

* kubectl describe pod carsharestock-74cbf5f6dc-9bqgd  -n carshare
```
Containers:
  carshareorder:
    Container ID:   docker://f373677a0487760940bc736d38517e0fc3a03c953428bc86e8f5fd6b1ca8bd0c
    Image:          467263215646.dkr.ecr.ap-southeast-2.amazonaws.com/user31carsharestock:v2
    Image ID:       docker-pullable://467263215646.dkr.ecr.ap-southeast-2.amazonaws.com/user31carsharestock@sha256:9589e46f56a5aec63aaf4114bc0ebf8f46a3bfb8a2630fb59c5ec87f840a15f9
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 21 Oct 2020 16:10:45 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:  500m
    Requests:
      cpu:        200m
    Liveness:     http-get http://:8080/ delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:    http-get http://:15021/healthz/ready delay=1s timeout=1s period=2s #success=1 #failure=30
    Environment:
      api.payment.url:  <set to the key 'api.payment.url' of config map 'my-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-5gx6w (ro)

```
kubectl describe 명령으로 컨테이너에 configMap 적용여부를 알 수 있다. 


