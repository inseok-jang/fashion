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


### 바운디드 컨텍스트, 이벤트, 유저, 어그리게잇 등 설정 / Pub-Sub, Req-Res 연결
<img width="1211" alt="스크린샷 2022-04-08 오전 10 06 12" src="https://user-images.githubusercontent.com/54835264/162343389-322b4f31-7b8b-40bd-a2ef-b3ac9eb1be79.png">


### 완성본 검증
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

http  http://fashion-orderview:8080/orderStatuses

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

#### 주문
Order.java
```java
@PreRemove
public void onPreRemove(){
    OrderCancelled orderCancelled = new OrderCancelled();
    BeanUtils.copyProperties(this, orderCancelled);
    orderCancelled.publishAfterCommit();
}
```
 http http://fashion-order:8080/orders productId=1 productName=“Outwear-1” qty=1
 
> {"eventType":"OrderPlaced","timestamp":"20220407124339","id":1,"productId":1,"qty":1,"productName":"“Outwear-1”","me":true}
> {"eventType”:”StoreAccepted”,”timestamp":"20220407124339","id":1,"productId":1,"qty":1,"productName":"“Outwear-1”","me":true}
> {"eventType":"DeliveryStarted","timestamp":"20220407124339","id":1,"orderId":1,"productId":1,"productName":"“Outwear-1”","me":true}

#### 주문취소



## Req / Resp
## Gateway
## Deploy / Pipeline
<img width="1314" alt="스크린샷 2022-04-08 오전 12 36 03" src="https://user-images.githubusercontent.com/54835264/162343492-03ccb129-894e-467a-9981-47181a5f0b8b.png">
<img width="1100" alt="스크린샷 2022-04-08 오전 12 37 20" src="https://user-images.githubusercontent.com/54835264/162343497-b6952ceb-8b4e-4f3b-a630-ca7a848f1381.png">



## Circuit Breaker
## Autoscale(HPA)

<img width="733" alt="스크린샷 2022-04-08 오전 12 53 51" src="https://user-images.githubusercontent.com/54835264/162343458-39fedaa7-bdee-4274-8e96-e4e8f1237ec6.png">

## Self-healing(Liveness Probe)


## Zero-downtime deploy(Readiness Probe)


## Config Map / Persustemce Volume
