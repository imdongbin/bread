# bread
빵 배달 주문을 주제로 개인 과제 작성

# 분석/설계
<img width="624" alt="bread_model" src="https://user-images.githubusercontent.com/58290368/108818006-1afe1680-75fc-11eb-9c63-249f823c6059.png">

# 구현

## 1. Saga
## 2. CQRS
## 3. Correlation
- 고객이 여러 정보를 한 번에 확인할 수 있도록 (View)Mypage 구현
```
# Mypage 호출
http localhost:8084/mypages

# 반환값
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 23 Feb 2021 09:23:38 GMT
Transfer-Encoding: chunked

{
    "_embedded": {
        "mypages": [
            {
                "_links": {
                    "mypage": {
                        "href": "http://localhost:8084/mypages/1"
                    },
                    "self": {
                        "href": "http://localhost:8084/mypages/1"
                    }
                },
                "action": "apply",
                "bakeryId": 1,
                "item": "cake",
                "orderId": 1,
                "paymentId": null,
                "price": 25000.0,
                "qty": 1,
                "status": "Confirmed"
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8084/profile/mypages"
        },
        "search": {
            "href": "http://localhost:8084/mypages/search"
        },
        "self": {
            "href": "http://localhost:8084/mypages"
        }
    }
}
```

## 4. Req/Resp
- 주문(order)->결제(pay) 동기식 호출
- FeignClient 활용
```
# (order) PaymentService.java

package bread.external;

@FeignClient(name="pay", url="${api.pay.url}")
public interface PaymentService {
   @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);
}
```
- 주문을 받은 직후 결제 요청 처리
```
# (order) Order.java

    @PostPersist
    public void onPostPersist(){
        bread.external.Payment payment = new bread.external.Payment();
        payment.setOrderId(getId());
        OrderApplication.applicationContext.getBean(bread.external.PaymentService.class)
            .pay(payment);
    }
```
- 동기 호출에서는 결제 시스템 장애 시 주문을 받지 못하는 것을 확인
```
# 결제(pay) 서비스 중지 (ctrl+c)

# 빵 2건 주문
http localhost:8081/orders item="cake" qty=1    #Fail (500) 
http localhost:8081/orders item="donut" qty=5   #Fail (500)

# 결제(pay) 서비스 재기동

# 빵 2건 재주문
http localhost:8081/orders item="cake" qty=1    #Success (201) 
http localhost:8081/orders item="donut" qty=5   #Success (201)
```

## 5. Gateway
- spring boot gateway 활용
```
# gateway 서비스 가동 (8088 포트)

# gateway 통한 orders/1 조회
http localhost:8088/orders/1

# 반환값 (http localhost:8081/orders/1 의 결과값과 동일)
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 23 Feb 2021 09:53:33 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "item": "cake",
    "qty": 1,
    "status": "Confirmed"
}
```

## 7. Circuit Breaker
- Spring FeignClient + Hystrix 옵션으로 구현
- Hystrix 설정
```
# order 서비스 > application.yml 파일 수정

feign:
  hystrix:
    enabled: true

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610
```
- 호출 받는 결제(pay) 서비스의 가공 부하 발생 처리
```
# pay 서비스 > Payment.java 수정

@PrePersist
    public void onPrePersist(){ // 부하 발생 처리

        if("cancle".equals(action)) {
            // 취소 요청 분기
            PayCanceled payCanceled = new PayCanceled();
            BeanUtils.copyProperties(this, payCanceled);
            payCanceled.publish();
        } else {
            PayApproved payApproved = new PayApproved();
            BeanUtils.copyProperties(this, payApproved);

            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                @Override
                public void beforeCommit(boolean readOnly) {
                    payApproved.publish();
                }
            });

            // 부하 발생을 위한 코드
            try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
```
- siege 툴을 사용한 가공 부하 발생
- 동시 사용자 100명, 20초 동안 실시
```
siege -c100 -t20S --content-type "application/json" 'http://localhost:8081/orders POST {"item": "cake", "qty": 1}'

{	"transactions":			          32,
	"availability":			       86.49,
	"elapsed_time":			       19.49,
	"data_transferred":		        0.01,
	"response_time":		       11.47,
	"transaction_rate":		        1.64,
	"throughput":			        0.00,
	"concurrency":			       18.83,
	"successful_transactions":	          32,
	"failed_transactions":		           5,
	"longest_transaction":		       19.14,
	"shortest_transaction":		        0.00
}
```
- 86.49% 성공, 13.51% 실패

## 11. Polyglot
- customer 서비스 DB를 기존 H2에서 hsql로 변경
```
# customer 서비스 > pom.xml 수정
<!--		<dependency>-->
<!--			<groupId>com.h2database</groupId>-->
<!--			<artifactId>h2</artifactId>-->
<!--			<scope>runtime</scope>-->
<!--		</dependency>-->

		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<version>2.4.0</version>
			<scope>runtime</scope>
		</dependency>
```
- customer 서비스 재기동
- 케이크 주문을 발생 시킨 후 mypage 조회
```
# mypage 호출
http localhost:8084/mypages/1

# 기존 h2 사용할 때와 동일 하게 정상 조회됨
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Tue, 23 Feb 2021 09:58:08 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/1"
        },
        "self": {
            "href": "http://localhost:8084/mypages/1"
        }
    },
    "action": "apply",
    "bakeryId": 3,
    "item": "cake",
    "orderId": 6,
    "paymentId": null,
    "price": 25000.0,
    "qty": 1,
    "status": "Confirmed"
}
```

# 운영

## 6. Deploy/ Pipeline
k8s에 create로 배포

## 8. Autoscale (HPA)
공부해서 하자

## 9. Zero-downtime deploy (Readiness Probe)
공부해서 하자

## 10. Config Map/ Persistence Volume
컨피그 맵 공부 해서 하자

## 12. Self-healing (Liveness Probe)
공부해서 하자
