# cna-bookstore

## User Scenario
```
* 고객 또는 고객 운영자는 고객정보를 생성한다.
* 북 관리자는 판매하는 책의 종류와 보유 수량을 생성하고 수정할 수 있다.
* 고객은 책의 주문과 취소가 가능하며 주문 정보의 수정은 없다.
* 고객이 주문을 생성할 때 고객정보와 Book 정보가 있어야 한다.
  - Order -> Customer 동기호출
  - Order -> BookInventory 동기호출
  - Customer 서비스가 중지되어 있더라도 주문은 생성하되 주문상태를 "Customer_Not_Verified"로 설정하여 타 서비스로의 전달은 진행하지 않는다.
* 주문 시에 재고가 없더라도 주문이 가능하다.
  - 주문 상태는 “Ordered”
* 주문 취소는 "Ordered" 상태일 경우만 가능하다.
* 배송준비는 주문 정보를 받아 이뤄지며 재고가 부족한 경우, 책 입고가 이뤄져서 재고 수량이 충분한 경우에 배송 준비가 완료되었음을 알린다.
* 배송은 주문 생성 정보를 받아서 배송을 준비하고 주문 상품 준비 정보를 받아서 배송을 시작하며 배송이 시작되었음을 주문에도 알린다.
  - 주문 생성 시 배송 생성
  - 상품 준비 시 배송 시작  
* 배송을 시행하는 외부 시스템(물류 회사 시스템) 또는 배송 담당자는 배송 단계별로 상태는 변경한다. 변경된 배송 상태는 주문에 알려 반영한다.
* 주문 취소되더라도 고객은 MyPage에서 주문 이력을 모두 조회할 수 있다.
* Delivery Status 변경 시 Kakao 서비스로 Alarm Message를 생성한다.
* Kakao Alram Message 생성 시 CustomerId가 존재하는 경우에만 Message를 전송한다. (Req/Rep)
```
## 아키텍처
```
* 모든 요청은 단일 접점을 통해 이뤄진다.

```

## Cloud Native Application Model
![Alt text](cna-bookstore-jw-2.PNG?raw=true "Optional Title")

## 구현 점검

### 모든 서비스 정상 기동 
```
* Httpie Pod 접속
kubectl exec -it httpie -- bash

* API 
http http://gateway:8080/customers
http http://gateway:8080/myPages
http http://gateway:8080/books
http http://gateway:8080/deliverables
http http://gateway:8080/stockInputs
http http://gateway:8080/orders
http http://gateway:8080/deliveries
```

### Kafka 기동 및 모니터링 용 Consumer 연결
```
kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic cnabookstore --from-beginning
```

### 고객 생성
```
http POST http://gateway:8080/customers customerName="hong gil dong"
http POST http://gateway:8080/customers customerName="bak na re"
```

### 책 정보 생성
```
$ http POST http://gateway:8080/books bookName="alice in a wonderland" stock=100
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 01:53:45 GMT
Location: http://bookinventory:8080/books/1
transfer-encoding: chunked

{
    "_links": {
        "book": {
            "href": "http://bookinventory:8080/books/1"
        }, 
        "self": {
            "href": "http://bookinventory:8080/books/1"
        }
    }, 
    "bookName": "alice in a wonderland", 
    "stock": 100
}

$ http POST http://gateway:8080/books bookName="quobadis?" stock=50
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 01:54:38 GMT
Location: http://bookinventory:8080/books/2
transfer-encoding: chunked

{
    "_links": {
        "book": {
            "href": "http://bookinventory:8080/books/2"
        }, 
        "self": {
            "href": "http://bookinventory:8080/books/2"
        }
    }, 
    "bookName": "quobadis?", 
    "stock": 50
}

```

### 주문 생성
```
http POST http://gateway:8080/orders bookId=1 customerId=1 deliveryAddress="bundang gu" quantity=50
http POST http://gateway:8080/orders bookId=1 customerId=2 deliveryAddress="incheon si" quantity=100
```

##### Message 전송 확인 결과
```
{"eventType":"Ordered","timestamp":"20200909024119","orderId":4,"bookId":1,"customerId":2,"quantity":100,"deliveryAddress":"incheon si","orderStatus":"ORDERED","me":true}
```

##### Deliveriy 확인 결과
```
root@httpie:/# http http://gateway:8080/deliveraries
{
    "_links": {
        "delivery": {
            "href": "http://delivery:8080/deliveries/4"
        }, 
        "self": {
            "href": "http://delivery:8080/deliveries/4"
        }
    }, 
    "deliveryAddress": "incheon si", 
    "deliveryStatus": "CreateDelivery", 
    "orderId": 4
}

```

##### Deliverables 확인 결과
```
root@httpie:/# http http://gateway:8080/deliverables
           {
                "_links": {
                    "deliverable": {
                        "href": "http://bookinventory:8080/deliverables/5"
                    }, 
                    "self": {
                        "href": "http://bookinventory:8080/deliverables/5"
                    }
                }, 
                "orderId": 4, 
                "quantity": 100, 
                "status": "Stock_Lacked"
            }
```

### 주문 준비
```
root@httpie:/# http POST http://gateway:8080/stockInputs bookId=1 quantity=200
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:47:04 GMT
transfer-encoding: chunked

{
    "bookId": 1, 
    "id": 8, 
    "inCharger": null, 
    "quantity": 200
}
```
```
{"eventType":"DeliveryPrepared","timestamp":"20200909024704","id":null,"orderId":4,"status":"Delivery_Prepared","me":true}
{"eventType":"DeliveryStatusChanged","timestamp":"20200909024704","id":4,"orderId":4,"deliveryStatus":"Shipped","me":true}
```
##### 재고 수량 변경 확인 결과
```
root@httpie:/# http http://gateway:8080/books/1 
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:53:26 GMT
transfer-encoding: chunked
{
    "_links": {
        "book": {
            "href": "http://bookinventory:8080/books/1"
        }, 
        "self": {
            "href": "http://bookinventory:8080/books/1"
        }
    }, 
    "bookName": "alice in a wonderland", 
    "stock": 150
}
```

##### 주문 상태 변경 확인 결과
```
root@httpie:/# http http://gateway:8080/orders/4
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:55:30 GMT
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://order:8080/orders/4"
        }, 
        "self": {
            "href": "http://order:8080/orders/4"
        }
    }, 
    "bookId": 1, 
    "customerId": 2, 
    "deliveryAddress": "incheon si", 
    "orderStatus": "Shipped", 
    "quantity": 100
}

root@httpie:/# http http://gateway:8080/deliverables/5
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:55:07 GMT
transfer-encoding: chunked

{
    "_links": {
        "deliverable": {
            "href": "http://bookinventory:8080/deliverables/5"
        }, 
        "self": {
            "href": "http://bookinventory:8080/deliverables/5"
        }
    }, 
    "orderId": 3, 
    "quantity": 100, 
    "status": "Delivery_Prepared"
}
```

### 배송 상태 변경
![Alt text](delivery_ShippingCompleted.PNG?raw=true "Optional Title")

### 배송 상태 Kakao Message 확인
![Alt text](kakao_alarmMessage.PNG?raw=true "Optional Title")

### 고객 Mypage 이력 확인
![Alt text](kakao_myPages.PNG?raw=true "Optional Title")

### 장애 격리
```
Customer 서비스 중지 시 Kakao Alarm 미 전송
@PrePersist 에서 try / catch 처리
```

## CI/CD 점검

## Circuit Breaker 점검

```
Hystrix Command
	5000ms 이상 Timeout 발생 시 CircuitBearker 발동

CircuitBeaker 발생
	http http://kakao:8080/selectKakaoAlarmInfo?id=0
		- 잘못된 쿼리 수행 시 CircuitBeaker
		- 10000ms(10sec) Sleep 수행
		- 5000ms Timeout으로 CircuitBeaker 발동
		- 10000ms(10sec) 
```

```
실행 결과
```
![Alt text](kakao_circuitBreaker.PNG?raw=true "Optional Title")
```
root@httpie:/# http http://kakao:8080/selectKakaoAlarmInfo?id=1
HTTP/1.1 200 
Content-Length: 38
Content-Type: text/plain;charset=UTF-8
Date: Wed, 09 Sep 2020 15:43:06 GMT

Book Delivery Stauts Changed : Shipped

root@httpie:/# http http://kakao:8080/selectKakaoAlarmInfo?id=0
HTTP/1.1 200 
Content-Length: 17
Content-Type: text/plain;charset=UTF-8
Date: Wed, 09 Sep 2020 15:43:17 GMT

CircuitBreaker!!!
```

```
소스 코드

  @GetMapping("/selectKakaoAlarmInfo")
  @HystrixCommand(fallbackMethod = "fallback", commandProperties = {
          @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000"),
          @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000")
  })
  public String selectKakaoAlarmInfo(@RequestParam long id) throws InterruptedException {

   if (id <= 0) {
    System.out.println("@@@ CircuitBreaker!!!");
    Thread.sleep(10000);
    //throw new RuntimeException("CircuitBreaker!!!");
   } else {
    Optional<KakaoAlarm> kakaoAlarm = kakaoAlarmRepository.findById(id);
    return kakaoAlarm.get().getKakaoMessage();
   }

   System.out.println("$$$ SUCCESS!!!");
   return " SUCCESS!!!";
  }

  private String fallback(long id) {
   System.out.println("### fallback!!!");
   return "CircuitBreaker!!!";
  }
```

## Autoscale 점검
### 설정 확인
```
application.yaml 파일 설정 변경
(https://k8s.io/examples/application/php-apache.yaml 파일 참고)
 resources:
  limits:
    cpu: 500m
  requests:
    cpu: 200m
```
### 점검 순서
```
1. HPA 생성 및 설정
	kubectl autoscale deploy bookinventory --min=1 --max=10 --cpu-percent=30
	kubectl get hpa bookinventory -o yaml
2. 모니터링 걸어놓고 확인
	kubectl get hpa bookinventory -w
	watch kubectl get deploy,po
3. Siege 실행
  siege -c10 -t60S -v http://gateway:8080/books/
```
### 점검 결과
![Alt text](images/HPA_test.PNG?raw=true "Optional Title")

## Readiness Probe 점검
### 설정 확인
```
readinessProbe:
  httpGet:
    path: '/orders'
    port: 8080
  initialDelaySeconds: 12
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
```
### 점검 순서
#### 1. Siege 실행
```
siege -c2 -t120S  -v 'http://gateway:8080/orders
```
#### 2. Readiness 설정 제거 후 배포
#### 3. Siege 결과 Availability 확인(100% 미만)
```
Lifting the server siege...      done.

Transactions:                    330 hits
Availability:                  70.82 %
Elapsed time:                 119.92 secs
Data transferred:               0.13 MB
Response time:                  0.02 secs
Transaction rate:               2.75 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    0.07
Successful transactions:         330
Failed transactions:             136
Longest transaction:            1.75
Shortest transaction:           0.00
```
#### 4. Readiness 설정 추가 후 배포

#### 6. Siege 결과 Availability 확인(100%)
```
Lifting the server siege...      done.

Transactions:                    443 hits
Availability:                 100.00 %
Elapsed time:                 119.60 secs
Data transferred:               0.51 MB
Response time:                  0.01 secs
Transaction rate:               3.70 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    0.04
Successful transactions:         443
Failed transactions:               0
Longest transaction:            0.18
Shortest transaction:           0.00
 
FILE: /var/log/siege.log
```

## Liveness Probe 점검
### 설정 확인
```
livenessProbe:
  httpGet:
    path: '/isHealthy'
    port: 8080
  initialDelaySeconds: 120
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 5
```
### 점검 순서 및 결과
#### 1. 기동 확인
```
http http://gateway:8080/orders
```
#### 2. 상태 확인
```
oot@httpie:/# http http://order:8080/isHealthy
HTTP/1.1 200 
Content-Length: 0
Date: Wed, 09 Sep 2020 02:14:22 GMT
```

#### 3. 상태 변경
```
root@httpie:/# http http://order:8080/makeZombie
HTTP/1.1 200 
Content-Length: 0
Date: Wed, 09 Sep 2020 02:14:24 GMT
```
#### 4. 상태 확인
```
root@httpie:/# http http://order:8080/isHealthy
HTTP/1.1 500 
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Wed, 09 Sep 2020 02:14:28 GMT
Transfer-Encoding: chunked

{
    "error": "Internal Server Error", 
    "message": "zombie.....", 
    "path": "/isHealthy", 
    "status": 500, 
    "timestamp": "2020-09-09T02:14:28.338+0000"
}
```
#### 5. Pod 재기동 확인
```
root@httpie:/# http http://order:8080/isHealthy
http: error: ConnectionError: HTTPConnectionPool(host='order', port=8080): Max retries exceeded with url: /makeZombie (Caused by NewConnectionError('<requests.packages.urllib3.connection.HTTPConnection object at 0x7f5196111c50>: Failed to establish a new connection: [Errno 111] Connection refused',))

root@httpie:/# http http://order:8080/isHealthy
HTTP/1.1 200 
Content-Length: 0
Date: Wed, 09 Sep 2020 02:36:00 GMT
```
