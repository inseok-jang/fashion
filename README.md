# * 온라인 의류 쇼핑몰 *
![online-clothes-shopping-on-internet-illustration-network-sale-payment-market-consumer-choose-garment-clothing-store_109722-1432](https://user-images.githubusercontent.com/54835264/162344434-0429aeea-da1d-4e41-8c36-cbaaee4e7b0b.jpg)


## 분석설계

### 서비스 시나리오
#### 기능적 요구사항
1. 점주가 아이템을 등록 한다.
2. 구매자가 아이템을 주문한다.
3. 주문과 동시에 결제가 진행된다.
4. 결제가 되면 주문이 전달된다.
5. 판매자가 주문을 확인하여 배송을 시작한다.
6. 고객이 주문을 취소할 수 있다.
7. 주문이 취소되면 점주 확인 후 배송이 취소된다.

#### 비기능적 요구사항
1. 트랜잭션
 * 결제가 되지 않은 주문건은 거래가 성립되지 성립되지 않아야 한다. (Sync 호출)

2. 장애격리
 * 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다. [Async (event-driven), Eventual Consistency]
 * 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다. [Circuit breaker, fallback]

3. 성능
* 구매자가 상점관리에서 확인할 수 있는 구매 정보 및 배송상태 등을 주문시스템에서 한번에 확인할 수 있어야 한다 [CQRS]


### 바운디드 컨텍스트, 이벤트, 유저, 어그리게잇 등 설정
<img width="1211" alt="스크린샷 2022-04-08 오전 10 06 12" src="https://user-images.githubusercontent.com/54835264/162343389-322b4f31-7b8b-40bd-a2ef-b3ac9eb1be79.png">


### Pub-Sub, Req-Res 연결 / 완성본 검증
<img width="1151" alt="스크린샷 2022-04-07 오후 1 52 20" src="https://user-images.githubusercontent.com/54835264/162343407-fec965ac-4da4-4105-bbca-8c4dc8e3b8a3.png">


## SAGA
* OrderPlaced — (sync) —> Pay
* OrderPlaced — (async) —> StoreAccepted — (async) —> DeliveryStarted

Order.java
```java
@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long productId;
    private Integer qty;
    private String productName;

    @PostPersist
    public void onPostPersist(){
        OrderPlaced orderPlaced = new OrderPlaced();
        BeanUtils.copyProperties(this, orderPlaced);
        orderPlaced.publishAfterCommit();
        
        fashion.external.Payment payment = new fashion.external.Payment();
        payment.setId(getid());

        OrderApplication.applicationContext.getBean(fashion.external.PaymentService.class).pay(payment);
        orderCancelled.publishAfterCommit();
    }
```

PolicyHandler - Store
```java
@Service
public class PolicyHandler{
    @Autowired
    StoreRepository storeRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderPlaced_StoreAccepted(@Payload OrderPlaced orderPlaced){

        if(orderPlaced.isMe()){
           Store store = new Store();
           store.setOrderId(orderPlaced.getId());
           store.setProductId(orderPlaced.getProductId());
           store.setProductName(orderPlaced.getProductName());
           storeRepository.save(store);
        }
    }
}

```

PolicyHandler.java - Delivery
```java
@Service
public class PolicyHandler{
    @Autowired
    DeliveryRepository deliveryRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverStoreAccepted_StartDelivery(@Payload StoreAccepted storeAccepted){

        if(storeAccepted.isMe()){
           Delivery delivery = new Delivery();
           delivery.setOrderId(storeAccepted.getId());
           delivery.setProductId(storeAccepted.getProductId());
           delivery.setProductName(storeAccepted.getProductName());
           deliveryRepository.save(delivery);
        }
    }
}
```

* 주문 생성 -> 수락 -> 배달 시작

http http://fashion-order:8080/orders productId=1 productName=“Outwear-1 qty=1

* 이벤트 확인

> {"eventType":"OrderPlaced","timestamp":"20220407112753","id":1,"productId":1,"qty":1,"productName":"“Outwear-1","me":true}
> {"eventType”:”StoreAccepted","timestamp":"20220407112950","id":1,"orderId":1,"productId":1,"productName":"“Outwear-1","me":true}
> {"eventType":"DeliveryStarted","timestamp":"20220407113210","id":1,"orderId":1,"productId":1,"productName":"“Outwear-1","me":true}


## CQRS
* 주문확인에 대한 뷰서비스를 제공

OrderView.java
```java
@StreamListener(KafkaProcessor.INPUT)
public void when_CREATE_orderPlaced(@Payload OrderPlaced orderPlaced){
        OrderStatus orderStatus = new OrderStatus();
        orderStatus.setOrderId(orderPlaced.getId());
        orderStatus.setProductId(orderPlaced.getProductId());
        orderStatus.setQty(orderPlaced.getQty());
        orderStatus.setProductName(orderPlaced.getProductName());
        orderStatus.setOrderStatus(OrderPlaced.class.getSimpleName());
        repository.save(orderStatus);
}

@StreamListener(KafkaProcessor.INPUT)
public void when_UPDATE_DeliveryStarted(@Payload DeliveryStarted deliveryStarted){
        OrderStatus orderStatus = repository.findById(deliveryStarted.getOrderId()).orElse(null);;
        if( orderStatus != null ){
            orderStatus.setDeliveryId(deliveryStarted.getId());
            orderStatus.setDeliveryStatus(DeliveryStarted.class.getSimpleName());
            repository.save(orderStatus);
        }
}
```

* 주문 및 주문상태 확인

http http://fashion-order:8080/orders productId=1 productName=“Outwear-2” qty=1

http http://fashion-orderview:8080/orderStatuses

>    "orderStatuses" : [ {
>      "productId" : 1,
>      "qty" : 1,
>      "productName" : “Outwear-2",
>      "orderStatus" : "OrderPlaced",
>      "deliveryId" : 1,
>      "deliveryStatus" : "DeliveryStarted",
>      "_links" : {
>        "self" : {
>          "href" : "http://fashion-orderview:8080/orderStatuses/1"
>        },
>        "orderStatus" : {
>          "href" : "http://fashion-orderview:8080/orderStatuses/1"
>        }
>      }
>    } ]


## Correlation / Compensation
* Fashion Store 프로젝트에서는 PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현 
* 이벤트 클래스 안의 변수로 전달받아 서비스간 연관된 처리 구현
* 주문 -> 점주승인 -> 배달시작 -> 주문취소 -> 취소승인 -> 배달취소

### 주문
Order.java
```java
@PreRemove
public void onPreRemove(){
    OrderCancelled orderCancelled = new OrderCancelled();
    BeanUtils.copyProperties(this, orderCancelled);
    orderCancelled.publishAfterCommit();
}
```

*주문

 http http://fashion-order:8080/orders productId=1 productName=“Outwear-1” qty=1

* 이벤트 확인

> {"eventType":"OrderPlaced","timestamp":"20220407124339","id":1,"productId":1,"qty":1,"productName":"“Outwear-1”","me":true}
> {"eventType”:”StoreAccepted”,”timestamp":"20220407124339","id":1,"productId":1,"qty":1,"productName":"“Outwear-1”","me":true}
> {"eventType":"DeliveryStarted","timestamp":"20220407124339","id":1,"orderId":1,"productId":1,"productName":"“Outwear-1”","me":true}

### 주문취소

StoreOrder.java
```java
    @PreRemove
    public void onPreRemove(){
    StoreCancelled storeCancelled = new StoreCancelled();
    BeanUtils.copyProperties(this, storeCancelled);
    storeCancelled.publishAfterCommit();
```


PolicyHandler.java - Store
```java
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderCancelled_StoreCancelled(@Payload OrderCancelled orderCancelled){
    if(orderCancelled.isMe()){
        List<StoreOrder> storeOrderList = storeRepository.findByOrderId(orderCancelled.getId());
        if ((storeOrderList != null) && !storeOrderList.isEmpty()){
            storeRepository.deleteAll(storeOrderList);
        }
    }
```

Delivery.java
```java
    @PreRemove
    public void onPreRemove(){
    DeliveryCancelled deliveryCancelled = new DeliveryCancelled();
    BeanUtils.copyProperties(this, deliveryCancelled);
    deliveryCancelled.publishAfterCommit();
```

PolicyHandler.java - Delivery
```java
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverStoreCancelled_DeleteDelivery(@Payload StoreCancelled storeCancelled){
        List<Delivery> deliveryList = deliveryRepository.findByOrderId(storeCancelled.getId());
        if ((deliveryList != null) && !deliveryList.isEmpty()){
            deliveryRepository.deleteAll(deliveryList);
    }
```

* 주문 취소

http DELETE http://fashion-order:8080/orders/1

* 이벤트 확인

> {"eventType":"OrderCancelled","timestamp":"20220407124417","id":1,"productId":1,"qty":1,"productName":"“Outwear-1”","me":true}
> {"eventType”:”StoreCancelled","timestamp":"20220407124417","id":1,"productId":1,"qty":1,"productName":"“Outwear-1”","me":true}
> {"eventType":"DeliveryCancelled","timestamp":"20220407124417","id":1,"orderId":1,"productId":1,"productName":"“Outwear-1”","me":true}


## Req / Resp
* FeignClient 구현

pom.xml
```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

        <!-- feign client -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```

Application.java
```java
@SpringBootApplication
@EnableFeignClients
public class Application {
…
}
```

Order.java
```java
@PostPersist
private void callDeliveryStart(){

    // 배송 시작
    DeliveryService deliveryService = Application.applicationContext.getBean(DeliveryService.class);
    deliveryService.startDelivery(delivery);
}
```


DeliveryService.java
```java
@FeignClient(name ="delivery", url="${api.url.delivery}")
public interface DeliveryService {

   @RequestMapping(method = RequestMethod.POST, value = "/deliveries", consumes = "application/json")
    void startDelivery(Delivery delivery);

}
```

* 주문

http localhost:8081/orders productId=1 quantity=1 customerId=“cust1” customerName=“jang” customerAddr="seoul"

* 배달 확인

http localhost:8082/deliveries


> HTTP/1.1 200 
> Content-Type: application/hal+json;charset=UTF-8
> Date: Thu, 07 Apr 2022 05:48:11 GMT
> Transfer-Encoding: chunked
>
> {
>     "_embedded": {
>         "deliveries": [
>            {
>                "_links": {
>                    "delivery": {
>                        "href": "http://localhost:8082/deliveries/1"
>                    },
>                   "self": {
>                        "href": "http://localhost:8082/deliveries/1"
>                    }
>                },
>                "customerId": "“cust1”",
>                "customerName": "“jang”",
>                "deliveryAddress": "seoul",
>                "deliveryId": 1,
>               "deliveryState": "DeliveryStarted",
>                "orderId": null,
>                "productId": 1,
>                "productName": “Outwear-1”,
>                "quantity": 1
>            }
>        ]
>    },
>    "_links": {
>        "profile": {
>           "href": "http://localhost:8082/profile/deliveries"
>        },
>        "search": {
>            "href": "http://localhost:8082/deliveries/search"
>        },
>        "self": {
>            "href": "http://localhost:8082/deliveries"
>        }
>    }
>}


## Gateway
* Istio Ingress Gateway 를 통한 단일진입점 구현

VirtualService 생성
```yaml
kubectl apply -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: fasion-vs
spec:
  hosts:
    - "*"
  gateways:
  - fashion-gw
  http:
  - match:
    - uri:
         prefix: /api/fashion-order
    rewrite:
      uri: /
    route:
    - destination:
         host: fashion-order
         port:
           number: 8080
EOF
```

Istio Gateway 생성
```yaml
kubectl apply -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: fashion-gw
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
```
* default 네임스페이스에 Istio 주입

kubectl label namespace default istio-injection=enabled

* 게이트웨이 외부 서비스주소 확인

kubectl -n istio-system get service/istio-ingressgateway

> NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP                                                                PORT(S)                >                                                     AGE
> istio-ingressgateway   LoadBalancer   10.100.4.126   ae30b0fdcdd964a2aab068187a080e3f-17329660.ca-central-1.elb.amazonaws.com 15021:31179/TCP,80:32099/TCP,443:30894/TCP,31400:30542/TCP,15443:31328/TCP   71m


* Gateway 를 통한 서비스 접근 확인

curl http://ae30b0fdcdd964a2aab068187a080e3f-17329660.ca-central-1.elb.amazonaws.com/api/fashion-order/products

> {
>  "_embedded" : {
>    "products" : [ {
>      "@id" : 1,
>      "name" : “product-1”,
>      "price" : 20000,
>      "stock" : 1000000,
>      "imageUrl" : "",
>      "_links" : {
>        "self" : {
>          "href" : "http://ae30b0fdcdd964a2aab068187a080e3f-17329660.ca-central-1.elb.amazonaws.com/products/1"
>        },
>        "product" : {
>          "href" : "http://ae30b0fdcdd964a2aab068187a080e3f-17329660.ca-central-1.elb.amazonaws.com/products/1"
>        },
>        "productOptions" : {
>          "href" : "http://ae30b0fdcdd964a2aab068187a080e3f-17329660.ca-central-1.elb.amazonaws.com/products/1/productOptions"
>        }
>      }
>    }, ...


## Deploy / Pipeline
* AWS CodeBuild 를 마이크로 서비스 별로 생성
<img width="1314" alt="스크린샷 2022-04-08 오전 12 36 03" src="https://user-images.githubusercontent.com/54835264/162343492-03ccb129-894e-467a-9981-47181a5f0b8b.png">

* EKS에 배포 후 Deployment 확인
<img width="1100" alt="스크린샷 2022-04-08 오전 12 37 20" src="https://user-images.githubusercontent.com/54835264/162343497-b6952ceb-8b4e-4f3b-a630-ca7a848f1381.png">


## Circuit Breaker
* Istio DestionationRule 을 통한 서킷브레이커 구현

* 테스트를 위한 서비스 파드 레플리카 증가

kubectl scale deploy fashion-delivery --replicas=3

DestinationRule 생성
```
kubectl apply -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: fashion-dr
spec:
  host: fashion-delivery
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

* siege 생성

kubectl create deploy siege --image=ghcr.io/acmexii/siege-nginx:latest

* siege 컨테이너 진입

kubectl exec -it pod/siege -- /bin/bash

* RR패턴으로 3개 Replica 접근 확인
http http://fashion-delivery:8080/actuator/echo


> root@siege-75d5587bf6-2h657:/# http http://fashion-delivery:8080/actuator/echo
> HTTP/1.1 200 OK
> content-length: 47
> content-type: text/plain;charset=UTF-8
> date: Thu, 07 Apr 2022 09:29:26 GMT
> server: envoy
> x-envoy-upstream-service-time: 22
> 
> fashion-delivery-6fd58757b9-6gmwg/192.168.17.23
> 
> root@siege-75d5587bf6-2h657:/# http http://fashion-delivery:8080/actuator/echo
> HTTP/1.1 200 OK
> content-length: 47
> content-type: text/plain;charset=UTF-8
> date: Thu, 07 Apr 2022 09:29:28 GMT
> server: envoy
> x-envoy-upstream-service-time: 274
> 
> fashion-delivery-6fd58757b9-m8dlk/192.168.78.83
> 
> root@siege-75d5587bf6-2h657:/# http http://fashion-delivery:8080/actuator/echo
> HTTP/1.1 200 OK
> content-length: 47
> content-type: text/plain;charset=UTF-8
> date: Thu, 07 Apr 2022 09:29:30 GMT
> server: envoy
> x-envoy-upstream-service-time: 258
> 
> fashion-delivery-6fd58757b9-mhrk6/192.168.35.36

* actuator 를 통한 서비스 오류 발생

http PUT http://localhost:8080/actuator/down

> HTTP/1.1 200 
> Content-Type: application/json;charset=UTF-8
> Date: Thu, 07 Apr 2022 09:31:45 GMT
> Transfer-Encoding: chunked
> 
> {
>     "status": "DOWN"
> }

* 하나의 서비스 파드를 Down 한 이후 서킷브레이커 동작 확인
> root@siege-75d5587bf6-2h657:/# http http://fashion-delivery:8080/actuator/echo
> HTTP/1.1 200 OK
> content-length: 48
> content-type: text/plain;charset=UTF-8
> date: Thu, 07 Apr 2022 09:38:19 GMT
> server: envoy
> x-envoy-upstream-service-time: 4
> 
> fashion-delivery-6fd58757b9-dk6t9/192.168.19.157
> 
> root@siege-75d5587bf6-2h657:/# http http://fashion-delivery:8080/actuator/echo
> HTTP/1.1 200 OK
> content-length: 47
> content-type: text/plain;charset=UTF-8
> date: Thu, 07 Apr 2022 09:38:21 GMT
> server: envoy
> x-envoy-upstream-service-time: 3
> 
> fashion-delivery-6fd58757b9-mhrk6/192.168.35.36
> 
> root@siege-75d5587bf6-2h657:/# http http://fashion-delivery:8080/actuator/echo
> HTTP/1.1 200 OK
> content-length: 47
> content-type: text/plain;charset=UTF-8
> date: Thu, 07 Apr 2022 09:38:22 GMT
> server: envoy
> x-envoy-upstream-service-time: 6
> 
> fashion-delivery-6fd58757b9-mhrk6/192.168.35.36
> 
> root@siege-75d5587bf6-2h657:/# http http://fashion-delivery:8080/actuator/echo
> HTTP/1.1 200 OK
> content-length: 48
> content-type: text/plain;charset=UTF-8
> date: Thu, 07 Apr 2022 09:38:25 GMT
> server: envoy
> x-envoy-upstream-service-time: 7
> 
> fashion-delivery-6fd58757b9-dk6t9/192.168.19.157


## Autoscale(HPA)
* CPU 200m 설정
```yaml
spec:
containers:
    resources:
      requests:
        cpu: "200m"
```

* HPA(Horizonal Pod Autoscale)를 cpu임계치=20%, 최대Pod수=3, 최저Pod수=1 로 설정

kubectl autoscale deployment fashion-store --cpu-percent=20 --min=1 --max=3


* siege 컨테이너 내부에서 동시사용자 1명, 20초간 부하 테스트를 실행

kubectl exec -it siege -- /bin/bash
siege -c1 -t10S -v http://fashion-store:8080/products


> HTTP/1.1 200     0.02 secs:    2599 bytes ==> GET  /products
> HTTP/1.1 200     0.01 secs:    2599 bytes ==> GET  /products
> 
> Lifting the server siege...
> Transactions:                    688 hits
> Availability:                 100.00 %
> Elapsed time:                   9.10 secs
> Data transferred:               1.71 MB
> Response time:                  0.01 secs
> Transaction rate:              75.60 trans/sec
> Throughput:                     0.19 MB/sec
> Concurrency:                    0.99
> Successful transactions:         688
> Failed transactions:               0
> Longest transaction:            0.58
> Shortest transaction:           0.00

* CPU 증가에 따른 오토스케일 확인
<img width="733" alt="스크린샷 2022-04-08 오전 12 53 51" src="https://user-images.githubusercontent.com/54835264/162343458-39fedaa7-bdee-4274-8e96-e4e8f1237ec6.png">

## Self-healing(Liveness Probe)
* Liveness Probe 설정 후 재배포 실행
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 120
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 5
```

* 서비스 오류 발생

> 2022-04-07 16:07:18.646 ERROR 1 --- [           main] o.s.boot.SpringApplication               : Application run failed

* 파드 상태 및 리스타트 확인

kubectl get po       
```cmd
NAME                               READY    STATUS             RESTARTS    AGE
fashion-store-7bdd7b8d8b-9nl67     1/2      CrashLoopBackOff   5           5m14s
```

## Zero-downtime deploy(Readiness Probe)

* HPA 제거

kubectl delete hpa fashion-store

* siege 컨테이너 진입 후 store 부하테스트 실행

kubectl exec -it siege -- /bin/bash
siege -c1 -t10S -v http://fashion-store:8080/products

* Availability 75.44% 확인
```cmd
HTTP/1.1 503     0.05 secs:      91 bytes ==> GET  /products
HTTP/1.1 503     0.08 secs:      91 bytes ==> GET  /products

Lifting the server siege...
Transactions:                    215 hits
Availability:                  75.44 %
Elapsed time:                   9.33 secs
Data transferred:               0.54 MB
Response time:                  0.04 secs
Transaction rate:              23.04 trans/sec
Throughput:                     0.06 MB/sec
Concurrency:                    0.99
Successful transactions:         215
Failed transactions:              70
Longest transaction:            0.42
Shortest transaction:           0.00
```

* Readiness Probe 설정 후 재배포 실행
```yaml
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
```
* Availability 100% 확인
```cmd
Lifting the server siege...
Transactions:                    680 hits
Availability:                 100.00 %
Elapsed time:                   9.02 secs
Data transferred:               1.71 MB
Response time:                  0.01 secs
Transaction rate:              75.60 trans/sec
Throughput:                     0.19 MB/sec
Concurrency:                    0.99
Successful transactions:         680
Failed transactions:               0
Longest transaction:            0.58
Shortest transaction:           0.00
```

## Config Map / Persustemce Volume
* Spring-Boot Profile 세팅을 위해 OS 환경 변수 SPRING_PROFILES_ACTIVE, TZ 설정

컨피그맵 생성
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: store-config
data:
 profile-k8s: “dev,k8s”
 timezone_seoul: “Asia/Seoul”
```

Deployment.yaml 내 컨피그맵 참조 선언
```yaml
spec:
  containers:
      env:
        - name: SPRING_PROFILES_ACTIVE 
          valueFrom:
            configMapKeyRef:
              name: store-config     
              key: profile-k8s 
        - name: TZ
          valueFrom:
            configMapKeyRef:
              name: store-config
              key: timezone_seoul
```

