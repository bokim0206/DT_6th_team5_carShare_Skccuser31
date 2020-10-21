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
carShareGateway의 application.yml 에 적용

```
![image](https://user-images.githubusercontent.com/16017769/96737608-d453a200-13f8-11eb-8aa6-4d8e04795912.png)

## 폴리글랏 퍼시스턴스

CQRS 를 위한 customerpage 서비스만 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.

```
pom.xml 에 적용

```
![image](https://user-images.githubusercontent.com/16017769/96690883-31cafd00-13bf-11eb-9513-aa4375aaf265.png)



# 운영

## CI/CD 설정

stock에 대해 repository를 구성하였고, CI/CD플랫폼은 AWS의 CodeBuild를 사용했다.
Git Hook 설정으로 연결된 GitHub의 소스 변경 발생 시 자동 배포된다.
![image](https://user-images.githubusercontent.com/16017769/96723971-38229e80-13ea-11eb-997c-d21ac945deab.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

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
![image](https://user-images.githubusercontent.com/70302900/96588949-38099c80-131f-11eb-9e37-5f1846fca268.png)


* 키알리(kiali)화면에 서킷브레이커 동작 확인
![image](https://user-images.githubusercontent.com/70302900/96589002-46f04f00-131f-11eb-92b7-dd13ce203382.png)


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- Deployment 배포시 resource 설정 적용
![image](https://user-images.githubusercontent.com/42608068/96592913-e44d8200-1323-11eb-8d94-386116ecaf2c.png)

- replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 5프로를 넘어서면 replica 를 10개까지 늘려준다
![image](https://user-images.githubusercontent.com/42608068/96592628-8de04380-1323-11eb-8288-2288a9e189ec.png)
- 오토스케일이 어떻게 되고 있는지 HPA 모니터링을 걸어둔다, 어느정도 시간이 흐른 후, 스케일 아웃이 벌어지는 것을 확인할 수 있다
![image](https://user-images.githubusercontent.com/16017769/96661016-17c0f880-1386-11eb-86a9-6788ba45bd1a.png)
- kubectl get으로 HPA을 확인하면 CPU 사용률이 135%로 증가됐다.
![image](https://user-images.githubusercontent.com/16017769/96661066-30311300-1386-11eb-8d6c-7b6e2f67f83a.png)

## 무정지 재배포
- Readiness Probe 및 Liveness Probe 설정(buildspec.yml 설정)

![image](https://user-images.githubusercontent.com/42608068/96593140-24146980-1324-11eb-88d5-7dee61001832.png)

### Readiness Probe 설정
- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함 Git hook 연동 설정되어 Github의 소스 변경 발생 시 자동 빌드 배포됨
![image](https://user-images.githubusercontent.com/16017769/96661148-5c4c9400-1386-11eb-8f4f-9b83cab19b8c.png)


## Liveness Probe
- pod 삭제

![image](https://user-images.githubusercontent.com/16017769/96661174-6d95a080-1386-11eb-9f76-ab9a995c6286.png)

- 자동 생성된 pod 확인

![image](https://user-images.githubusercontent.com/16017769/96661206-81d99d80-1386-11eb-8b9d-539e36ef02e8.png)


## ConfigMap 사용

시스템별로 또는 운영중에 동적으로 변경 가능성이 있는 설정들을 ConfigMap을 사용하여 관리합니다.
Application에서 특정 도메일 URL을 ConfigMap 으로 설정하여 운영/개발등 목적에 맞게 변경가능합니다.  

* my-config.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: carshare
data:
  api.payment.url: http://carsharepayment:8080
```
my-config라는 ConfigMap을 생성하고 key값에 도메인 url을 등록한다. 

* carshareorder/buildsepc.yaml (configmap 사용)
```
 cat  <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          namespace: $_NAMESPACE
          labels:
            app: $_PROJECT_NAME
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: $_PROJECT_NAME
          template:
            metadata:
              labels:
                app: $_PROJECT_NAME
            spec:
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$_AWS_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                  ports:
                    - containerPort: 8080
                  env:
                    - name: api.payment.url
                      valueFrom:
                        configMapKeyRef:
                          name: my-config
                          key: api.payment.url
                  imagePullPolicy: Always
                
        EOF
```
Deployment yaml에 해단 configMap 적용

* PaymentService.java
```
@FeignClient(name="payment", contextId = "payment", url="${api.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
url에 configMap 적용

* kubectl describe pod carshareorder-bdd8c8c4c-l52h6  -n carshare
```
Containers:
  carshareorder:
    Container ID:   docker://f3c983b12a4478f3b4a7ee5d7fea308638903eb62e0941edd33a3bce5f5f6513
    Image:          496278789073.dkr.ecr.ap-southeast-2.amazonaws.com/carshareorder:9289bba10d5b0758ae9f6279d56ff77b818b8b63
    Image ID:       docker-pullable://496278789073.dkr.ecr.ap-southeast-2.amazonaws.com/carshareorder@sha256:95395c95d1bc19ceae8eb5cc0b288b38dc439359a084610f328407dacd694a81
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 21 Oct 2020 02:13:01 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:  500m
    Requests:
      cpu:        200m
    Liveness:     http-get http://:8080/ delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:    http-get http://:8080/ delay=30s timeout=2s period=5s #success=1 #failure=10
    Environment:
      api.payment.url:  <set to the key 'api.payment.url' of config map 'my-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-5gx6w (ro)

```
kubectl describe 명령으로 컨테이너에 configMap 적용여부를 알 수 있다. 


