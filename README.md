# SKANU
 ![image](https://user-images.githubusercontent.com/89397401/130740460-91135a6b-9676-460e-bac1-6f89d1da562f.png)

 
 # 서비스 시나리오

기능적 요구사항
1. 고객이 커피(음료)를 주문(Order)한다.
2. 고객이 지불(Pay)한다.
3. 결제모듈(payment)에 결제를 진행하게 되고 '지불'처리 된다.
4. 결제 '승인' 처리가 되면 주방에서 음료를 제조한다.
5. 고객과 매니저는 마이페이지를 통해 진행상태(OrderTrace)를 확인할 수 있다.
6. 음료가 준비되면 배달(Delivery)을 한다.
7. 고객이 취소(Cancel)하는 경우 지불 및 제조, 배달이 취소가 된다.
8. 결제가 완료되면 구매수량별 stamp를 적립, 결제 취소 시, 취소 수량만큼 stamp 를 차감한다. **(개인과제 기능추가)**

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 등록이 성립되지 않는다. - Sync 호출
2. 장애격리
    1. 결제가 수행되지 않더라도 주문 취소은 365일 24시간 받을 수 있어야 한다  - Async(event-driven), Eventual Consistency
    2. 결제 시스템이 과중되면 주문(Order)을 잠시 후 처리하도록 유도한다  - Circuit breaker, fallback
3. 성능
    1. 마이페이지에서 주문상태(OrderTrace) 확인  - CQRS

# 분석/설계


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: http://labs.msaez.io/#/storming/P3HDhaDCvERl1kR9ZDeRJxSKcBj1/516387ea675136cbac7cb8ac3c903426

### 이벤트 도출
![image](https://user-images.githubusercontent.com/79756040/129881425-3b9d3209-16b3-4d8a-a565-c82a85056980.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/79756040/129881872-bfa9ddb8-1e01-4885-b688-8a68d9770db4.png)

### 완성된 1차 모형
![image](https://user-images.githubusercontent.com/79756040/129881929-c6d1f38e-4115-4b5c-b650-4573852f9dd6.png)

### 완성된 최종 모형 ( 시나리오 점검 후-조별과제 )
![image](https://user-images.githubusercontent.com/79756040/130614202-d1ddaef6-466f-436f-a4a3-51714383d43a.png)

### 개인과제 기능추가
 - stamp 적립을 위한 기능을 추가하였다.
![image](https://user-images.githubusercontent.com/86760678/131473596-a87b8bb5-a074-41e7-a36a-f962b304055f.png)


## 헥사고날 아키텍처 다이어그램 도출 
### 조별과제
![HEXAGONAL2](https://user-images.githubusercontent.com/79756040/130914262-ec9dd0ea-0f13-4195-befd-566cb4de2620.png)
 
### 개인과제
![image](https://user-images.githubusercontent.com/86760678/131475843-24ecfe5b-6e2c-48e7-96ee-748bd84850b2.png)

 
 # 구현
 
 ## 흐름도
 

 
 - 주문(order) 후 주문취소(cancel order) 시, 흐름도
![image](https://user-images.githubusercontent.com/86760678/130794931-a22b6b52-044b-4f96-95b0-86acd56014c7.png)

 - 주문(order) 후 결제수행(pay) 시, 흐름도
![image](https://user-images.githubusercontent.com/86760678/130794970-29d3b38e-ea00-43cc-a4d8-51f5144d3803.png)

 **개인 과제 추가 후, 흐름도**
![image](https://user-images.githubusercontent.com/86760678/131481148-82894e7d-4bff-439a-b2ad-b62a5ff0ac0c.png)


 
 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084, 8088 이다)

```
   cd order
   mvn spring-boot:run

   cd payment
   mvn spring-boot:run

   cd delivery
   mvn spring-boot:run

   cd ordertrace
   mvn spring-boot:run
      
   cd stamp
   mvn spring-boot:run

   cd gateway
   mvn spring-boot:run
  ```
   
# DDD 의 적용

- msaez.io 를 통해 구현한 Aggregate 단위로 Entity 를 선언 후, 구현을 진행하였다.
 Entity Pattern 과 Repository Pattern 을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.
 
**기존 개발된 조별과제 repository를 Git을 통해 내려받은 후, 신규 추가된 stamp service에 대한 소스를 msaez에서 신규 생성된 소스를 로컬에서 머지하는 작업을 진행하였다.**

![image](https://user-images.githubusercontent.com/86760678/131482125-54116ccc-95a4-49c2-94c5-67903862a67a.png)

 ### Order 서비스의 Order.java 

```java
package sktkanumodel;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
//import java.util.List;
//import java.util.Date;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long productId;
    private Integer qty;
    private String paymentType;
    private Long cost;
    private String productName;

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.setOrderStatus("Order complete");

        sktkanumodel.external.Payment payment = new sktkanumodel.external.Payment();
        // mappings goes here
        payment.setOrderId(this.getId());
        payment.setProductId(this.getProductId());
        payment.setProductName(this.getProductName());
        payment.setPaymentStatus("Not Pay");
        payment.setQty(this.getQty());
        payment.setPaymentType(this.getPaymentType());
        payment.setCost(this.getCost());
        OrderApplication.applicationContext.getBean(sktkanumodel.external.PaymentService.class)
            .paid(payment);

        ordered.publishAfterCommit();
    }

    @PostRemove
    public void onPostRemove(){
        OrderCancelled orderCancelled = new OrderCancelled();
        BeanUtils.copyProperties(this, orderCancelled);
        orderCancelled.setOrderStatus("Order Cancelled");
        orderCancelled.publishAfterCommit();
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }
    public Integer getQty() {
        return qty;
    }

    public void setQty(Integer qty) {
        this.qty = qty;
    }
    public String getPaymentType() {
        return paymentType;
    }

    public void setPaymentType(String paymentType) {
        this.paymentType = paymentType;
    }
    public Long getCost() {
        return cost;
    }

    public void setCost(Long cost) {
        this.cost = cost;
    }
    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }
}

```

### Payment 시스템의 PolicyHandler.java

```java
package sktkanumodel;

import sktkanumodel.config.kafka.KafkaProcessor;
// import com.fasterxml.jackson.databind.DeserializationFeature;
// import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class PolicyHandler{
    @Autowired PaymentRepository paymentRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderCancelled_PayCancel(@Payload OrderCancelled orderCancelled)
    {
        if(orderCancelled.validate())
        {
            System.out.println("\n\n##### listener PayCancel : " + orderCancelled.toJson() + "\n\n");
            List<Payment> paymanetsList = paymentRepository.findByOrderId(orderCancelled.getId());
            if(paymanetsList.size()>0) {
                for(Payment payment : paymanetsList) {
                    if(payment.getOrderId().equals(orderCancelled.getId())){
                        System.out.println("##### OrderId :: "+ payment.getId() 
                                                      +" ... "+ payment.getProductName()+" is Cancelled");
                        payment.setPaymentType("ORDER CANCEL");
                        paymentRepository.save(payment);
                    }
                }
            }
        }
    }

    
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){}
}

```

적용 후 REST API 테스트를 통해 정상 동작 확인할 수 있다.
- 주문(Ordered) 수행의 결과

![image](https://user-images.githubusercontent.com/86760678/131693038-73b77e00-af9e-4734-be52-594cb9d46cf0.png)

![image](https://user-images.githubusercontent.com/86760678/131693066-b67492e7-7c90-4482-b651-a22561cb3a7a.png)


### 개인 과제 기능 추가 확인 (( stamp 기능 확인 ))
- 결제(Pay) 수행

![image](https://user-images.githubusercontent.com/86760678/131518386-2bb7e651-05c3-47b7-bfa2-0ef1a36316e7.png)

- stamp 적립 확인

![image](https://user-images.githubusercontent.com/86760678/131518473-ea6a533e-00fb-4cd2-864c-e6049dbf0d98.png)

- 결제취소 수행 및 stamp 감소 확인

![image](https://user-images.githubusercontent.com/86760678/131518791-1945e0e6-634a-4366-a640-6d5da1dbcfe8.png)




# Gateway 적용
API Gateway를 통하여 마이크로 서비스들의 진입점을 통일하였다.

```yaml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/** 
        - id: payment
          uri: http://localhost:8082
          predicates:
            - Path=/payments/** 
        - id: delivery
          uri: http://localhost:8083
          predicates:
            - Path=/deliveries/** 
        - id: ordertrace
          uri: http://localhost:8084
          predicates:
            - Path= /orderTraces/**
        - id: stamp
          uri: http://localhost:8085
          predicates:
            - Path= /stamps/**
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
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/** 
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/** 
        - id: delivery
          uri: http://delivery:8080
          predicates:
            - Path=/deliveries/** 
        - id: ordertrace
          uri: http://ordertrace:8080
          predicates:
            - Path= /orderTraces/**
        - id: stamp
          uri: http://stamp:8080
          predicates:
            - Path= /stamps/**
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

# 폴리그랏 퍼시스턴스
- delivery 서비스의 경우, 다른 마이크로 서비스와 달리 hsql을 구현하였다.
- 이를 통해 서비스 간 다른 종류의 데이터베이스를 사용하여도 문제 없이 동작하여 폴리그랏 퍼시스턴스를 충족함.

### delivery 서비스의 pom.xml
![image](https://user-images.githubusercontent.com/86760678/130350197-5d6071e2-1fb4-42fc-95ca-c44e21619ed5.png)

# 동기식 호출(Req/Res 방식)과 Fallback 처리
- order 서비스의 external/PaymentService.java 내에 결제(paid) 서비스를 호출하기 위하여 FeignClient를 이용하여 Service 대행 인터페이스(Proxy)를 구현

### order/external/PaymentService.java
```java
package sktkanumodel.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
// import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

// import java.util.Date;

@FeignClient(name="payment", url="http://localhost:8082")
public interface PaymentService {
    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void paid(@RequestBody Payment payment);

}

```

### Order 서비스의 Order.java
```java
package sktkanumodel;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
//import java.util.List;
//import java.util.Date;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long productId;
    private Integer qty;
    private String paymentType;
    private Long cost;
    private String productName;

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.setOrderStatus("Order complete");

        sktkanumodel.external.Payment payment = new sktkanumodel.external.Payment();
        // mappings goes here
        payment.setOrderId(this.getId());
        payment.setProductId(this.getProductId());
        payment.setProductName(this.getProductName());
        payment.setPaymentStatus("Not Pay");
        payment.setQty(this.getQty());
        payment.setPaymentType(this.getPaymentType());
        payment.setCost(this.getCost());
        OrderApplication.applicationContext.getBean(sktkanumodel.external.PaymentService.class)
            .paid(payment);

        ordered.publishAfterCommit();
    }

### 이하 생략
```
 **신규 Merge한 소스 기준으로 재테스트 수행**

- payment서비스를 올리지 않을 시, 주문(order) 요청 및 에러 난 화면 표시

![image](https://user-images.githubusercontent.com/86760678/131693735-a7dd02f7-787b-4373-be30-b766f535b86a.png)

- payment 서비스 기동 후 다시 주문 요청

![image](https://user-images.githubusercontent.com/86760678/131693913-96dbfb20-aa9d-46c4-9759-0980361432ca.png)


- payment 서비스에 주문 대기 상태로 저장 확인

![image](https://user-images.githubusercontent.com/86760678/131693969-abc87716-af03-4f02-8f10-e27f2f121aaa.png)


# 비동기식 호출(Pub/Sub 방식)
- Payment 서버스 내 Payment.java에서 아래와 같이 서비스 Pub 구현

```java

///

    @PostPersist
    public void onPostPersist(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();
    }

    @PostUpdate
    public void onPostUpdate(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();
    }

    @PreRemove
    public void onPreRemove(){
        PayCancelled payCancelled = new PayCancelled();
        BeanUtils.copyProperties(this, payCancelled);
        payCancelled.publishAfterCommit();

    }
///

```

- stamp 서비스 내 PolicyHandler.java에서 아래와 같이 Sub 구현

```java
///

@Service
public class PolicyHandler{
    @Autowired StampRepository stampRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPayed_EarnStamp(@Payload Paid paid){

        if(paid.validate())
        {

            if(!paid.getPhoneNumber().isEmpty())
            {
                System.out.println("\n\n##### listener paid : " + paid.toJson() + "\n\n");

                List<Stamp> stampList = stampRepository.findByPhoneNumber(paid.getPhoneNumber());
                if(stampList.size()>0) {
                    for(Stamp stamp : stampList) {
                        if(stamp.getPhoneNumber().equals(paid.getPhoneNumber())){
                            stamp.setStampQty(stamp.getStampQty()+paid.getQty());
                            stampRepository.save(stamp);
                        }
                    }
                }
                else{
                    Stamp newStamp = new Stamp();
                    newStamp.setPhoneNumber(paid.getPhoneNumber());
                    newStamp.setStampQty(paid.getQty());
                    stampRepository.save(newStamp);
                }
            }
        }

    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPayCancelled_EarnStamp(@Payload PayCancelled payCancelled){

        if(!payCancelled.validate()) return;

        System.out.println("\n\n##### listener EarnStamp : " + payCancelled.toJson() + "\n\n");

        if(!payCancelled.getPhoneNumber().isEmpty())
        {
            System.out.println("\n\n##### listener paid : " + payCancelled.toJson() + "\n\n");

            List<Stamp> stampList = stampRepository.findByPhoneNumber(payCancelled.getPhoneNumber());
            if(stampList.size()>0) {
                for(Stamp stamp : stampList) {
                    if(stamp.getPhoneNumber().equals(payCancelled.getPhoneNumber())){
                        stamp.setStampQty(stamp.getStampQty() - payCancelled.getQty());
                        stampRepository.save(stamp);
                    }
                }
            }
        }
    }


```

- 비동기식 호출은 다른 서비스가 비정상이여도 이상없이 동작가능하여, stamp 서비스에 장애가 나도 payment 서비스는 정상 동작을 확인

### stamp 서비스 내림

![image](https://user-images.githubusercontent.com/86760678/131694811-55b25fec-1bd0-4297-b387-f89ca32a35f2.png)

### 결제

![image](https://user-images.githubusercontent.com/86760678/131694929-89f125ab-96fd-4703-85b7-1362cefb1433.png)



# CQRS

viewer 인 ordertraces 서비스를 별도로 구현하여 아래와 같이 view가 출력된다.


### 주문 수행 후, ordertraces

![image](https://user-images.githubusercontent.com/86760678/131696055-83c810a9-18f1-4fc6-9aef-54afd479e275.png)

![image](https://user-images.githubusercontent.com/86760678/131696093-313a79ab-bc34-4013-a98d-596c24fe1c5a.png)


### 결제 수행 후, ordertraces

![image](https://user-images.githubusercontent.com/86760678/131694929-89f125ab-96fd-4703-85b7-1362cefb1433.png)

![image](https://user-images.githubusercontent.com/86760678/131695474-0f8c51b3-0863-4329-9024-8e1e237deb18.png)




# 운영
  
## Deploy / Pipeline

- git에서 소스 가져오기 (개별 과제 진행을 위해 조별 결과물 Fork 후 진행)

```
git clone https://github.com/PARKBYOUNGHWA/skanu.git
```

- Build 및 Azure Container Resistry(ACR) 에 Push 하기
 
```bash

cd ../order
mvn package
docker build -t skccpbh.azurecr.io/order:v1 .
docker push skccpbh.azurecr.io/order:v1

cd ../payment
mvn package
docker build -t skccpbh.azurecr.io/payment:v1 .
docker push skccpbh.azurecr.io/payment:v1

cd ../delivery
mvn package
docker build -t skccpbh.azurecr.io/delivery:v1 .
docker push skccpbh.azurecr.io/delivery:v1

cd ../ordertrace
mvn package
docker build -t skccpbh.azurecr.io/ordertrace:v1 .
docker push skccpbh.azurecr.io/ordertrace:v1

cd ../stamp
mvn package
docker build -t skccpbh.azurecr.io/stamp:v1 .
docker push skccpbh.azurecr.io/stamp:v1

cd ../gateway
mvn package
docker build -t skccpbh.azurecr.io/gateway:v1 .
docker push skccpbh.azurecr.io/gateway:v1

```

- ACR에 정상 Push 완료

![image](https://user-images.githubusercontent.com/86760678/131692476-0300ba0b-fb51-4229-9d17-73fe69e7cb6d.png)


- Kafka 설치 및 배포 완료

![image](https://user-images.githubusercontent.com/86760678/131696352-86b2a913-44b1-407e-91ca-b9619c18c8ea.png)

- Kubernetes Deployment, Service 생성

```sh
cd ..
cd order/kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ../../payment/kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ../../delivery/kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ../../ordertrace/kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ../../stamp/kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ../../gateway/kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml
```

/order/kubernetes/deployment.yml 파일 
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order
  labels:
    app: order
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order
          image: skccpbh.azurecr.io/order:v1
          ports:
            - containerPort: 8080
```

/order/kubernetes/service.yaml 파일 
```yml
apiVersion: v1
kind: Service
metadata:
  name: order
  labels:
    app: order
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: order
```

전체 deploy 완료 현황

![image](https://user-images.githubusercontent.com/86760678/131704455-bdad497a-e0cb-4e67-aaf4-4d0a52ece2dd.png)


## Persistence Volume

- 비정형 데이터를 관리하기 위해 PVC 생성 파일
- 신규 생성한 stamp service에 PVC 생성
**pvc.yml**
```yal
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: stamp-disk
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 1Gi
```
**deploymeny.yml**
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stamp
  labels:
    app: stamp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stamp
  template:
    metadata:
      labels:
        app: stamp
    spec:
      containers:
        - name: stamp
          image: skccpbh/stamp:v1
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
          volumeMounts:
            - name: volume
              mountPath: "/mnt/azure"
      volumes:
      - name: volume
        persistentVolumeClaim:
          claimName: stamp-disk
```

**application.yml**

```yml
logging:
  level:
    root: info
  file: /mnt/azure/logs/stamp.log
```
- 로그 확인

![PVC-LOGS](https://user-images.githubusercontent.com/89397401/130727058-c8b4e2d1-b07e-4b4f-9fdb-9acfe25f5943.png)


## Autoscale (HPA:HorizontalPodAutoscaler)

- 특정 수치 이상으로 사용자 요청이 증가할 경우 안정적으로 운영 할 수 있도록 HPA를 설치한다.

order 서비스에 resource 사용량을 정의한다.

order/kubernetes/deployment.yml
```yml
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "500Mi"
      cpu: "500m"
```

order 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15%를 넘어서면 replica를 10개까지 늘려준다.

```sh
kubectl autoscale deploy vote --min=1 --max=10 --cpu-percent=15
```

![image](https://user-images.githubusercontent.com/89397401/130735894-d889090c-95c6-4fa7-a1b5-aa1311e11eb9.png)

siege를 활용하여, 부하 생성한다. (200명의 동시사용자가 10초간 부하 발생)

```sh
siege -c200 -t10S -v --content-type "application/json" 'http://order:8080/orders POST { "productId": 1, "qty": 2, "paymentType" : "card", "cost" : 2000, "productName" : "RedTea"}'
```

- Autoscale을 확인하기 위해 모니터링한다.

```sh
$ watch kubectl get all
```

- 결과 확인 : 부하 생성 이후 CPU 15% 이상 사용 시 자동으로 POD가 증가하면서 Autoscale 됨을 확인 할 수 있다.

![image](https://user-images.githubusercontent.com/89397401/130736630-9a4de0c5-82d6-416c-a24a-11253d8412f0.png)

## Circuit Breaker

- Spring FeignClient + istio를 활용하여 Circuit Breaker 동작을 확인한다.
- istio 설치 후 istio injection이 enabled 된 namespace를 생성한다.
```
kubectl create namespace istio-test-ns
kubectl label namespace istio-test-ns istio-injection=enabled
```

- namespace label에 istio-injection이 enabled 된 것을 확인한다.  

![image](https://user-images.githubusercontent.com/89397401/130896837-0c6d68ec-f5e0-4ccf-be46-7f089829aaa7.png)

- 해당 namespace에 기존 서비스들을 재배포한다.

- deploy 실행
```Bash
kubectl create deploy order --image x0006319acr.azurecr.io/order:latest -n istio-test-ns
kubectl create deploy payment --image x0006319acr.azurecr.io/payment:latest -n istio-test-ns
kubectl create deploy delivery --image x0006319acr.azurecr.io/delivery:latest -n istio-test-ns
kubectl create deploy gateway --image x0006319acr.azurecr.io/gateway:latest -n istio-test-ns
kubectl create deploy ordertrace --image x0006319acr.azurecr.io/ordertrace:latest -n istio-test-ns
kubectl get all
```

- expose 하기
```Bash
kubectl expose deploy order --type="ClusterIP" --port=8080 -n istio-test-ns
kubectl expose deploy payment --type="ClusterIP" --port=8080 -n istio-test-ns
kubectl expose deploy delivery --type="ClusterIP" --port=8080 -n istio-test-ns
kubectl expose deploy gateway --type="LoadBalancer" --port=8080 -n istio-test-ns
kubectl expose deploy ordertrace --type="ClusterIP" --port=8080 -n istio-test-ns
kubectl get all
```

- 서비스들이 정상적으로 배포되었고, Container가 2개씩 생성된 것을 확인한다. 

![image](https://user-images.githubusercontent.com/89397401/130897459-f4d97481-bfb0-4422-957b-cab08b3fa9ef.png)

- Circuit Breaker 설정을 위해 아래와 같은 Destination Rule을 생성한다.

- Pending Request가 많을수록 오랫동안 쌓인 요청은 Response Time이 증가하게 되므로, 적절한 대기 쓰레드 풀을 적용하기 위해 connection pool을 설정했다.
```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: dr-httpbin
    namespace: istio-test-ns
  spec:
    host: gateway
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 1
          maxRequestsPerConnection: 1
```

- 설정된 Destinationrule을 확인한다. 

![image](https://user-images.githubusercontent.com/89397401/130916203-069469c2-248b-49a5-a23c-eedb442e3373.png)


- siege 를 활용하여 User가 2명인 상황에 대해서 요청을 보낸다. (설정값 c2)
  - siege 는 같은 namespace 에 생성하고, 해당 pod 안에 들어가서 siege 요청을 실행한다.
```
kubectl exec -it (siege POD 이름) -c siege -n istio-test-ns -- /bin/bash

siege -c2 -t30S -v --content-type "application/json" 'http://20.200.225.249:8080/orders POST {"productId": 1, "qty":3, "paymentTyp": "cash", "cost": 1000, "productName": "Coffee"}'
```

- 최종결과 확인
![image](https://user-images.githubusercontent.com/89397401/130823691-3df705c8-b964-4175-80c2-8970f4be57da.png)

- 운영시스템은 죽지 않고 지속적으로 Circuit Breaker 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 
- 약 84%정도 정상적으로 처리되었음.


## Zero-Downtime deploy (Readiness Probe)

- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함
- 생성된 siege Pod 안쪽에서 정상작동 확인
```
kubectl exec -it siege -- /bin/bash
siege -c1 -t2S -v http://order:8080/order
```

- siege 로 배포작업 직전에 워크로드를 모니터링 함
```
siege -c1 -t60S -v http://order:8080/orders --delay=1S
```

- Readiness가 설정되지 않은 yml 파일로 배포 진행
 
![image](https://user-images.githubusercontent.com/44763296/130768667-b3df9f46-d907-40aa-9304-4b0888a9983b.png)

```
kubectl apply -f deployment_without_readiness.yml
```

- 아래 그림과 같이, Kubernetes가 준비가 되지 않은 delivery pod에 요청을 보내서 siege의 Availability 가 100% 미만으로 떨어짐 

![image](https://user-images.githubusercontent.com/44763296/130476167-1b1eca10-ac7f-4065-86b7-69af9dcd7be5.png)


- 정지 재배포 여부 확인 전에, siege 로 배포작업 직전에 워크로드를 모니터링
```
siege -c1 -t60S -v http://order:8080/orders --delay=1S
```

- Readiness가 설정된 yml 파일로 배포 진행
```
readinessProbe:
  httpGet:
    path: '/actuator/health'
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
```

```
kubectl apply -f deployment_with_readiness.yml
```

- 배포 중 pod가 2개가 뜨고, 새롭게 띄운 pod가 준비될 때까지, 기존 pod가 유지됨을 확인
![image](https://user-images.githubusercontent.com/44763296/130477984-32b58cbf-a666-4e52-a377-d99c7a1477b7.png)
![image](https://user-images.githubusercontent.com/44763296/130478069-f3b1156e-bddb-48e3-ace9-ee21d3965206.png)


## Self-healing (Liveness Probe)

- order 서비스의 yml 파일에 liveness probe 설정을 바꾸어서, liveness probe 가 동작함을 확인

- liveness probe 옵션을 추가하되, 서비스 포트가 아닌 8090으로 설정, readiness probe 미적용
```
        livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8090
            initialDelaySeconds: 5
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
```

- order 서비스에 liveness가 적용된 것을 확인
```
kubectl describe po order
```
![image](https://user-images.githubusercontent.com/44763296/130482646-e598d23a-85b5-4e9d-b48b-6c9b99c7955f.png)


- order 에 liveness가 발동되었고, 8090 포트에 응답이 없기에 Restart가 발생함
![image](https://user-images.githubusercontent.com/44763296/130482675-ec20503f-a5d1-4c3d-9261-a89ae7ee17f1.png)
![image](https://user-images.githubusercontent.com/44763296/130482712-e18cda81-f3c8-47f7-abfc-47188cc423ad.png)


