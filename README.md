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
스프링 부트 게이트 웨이로 구현

## 11. Polyglot
hsql로 구현

# 운영

## 6. Deploy/ Pipeline
k8s에 create로 배포

## 7. Circuit Breaker
hystrix 으로 구현

## 8. Autoscale (HPA)
공부해서 하자

## 9. Zero-downtime deploy (Readiness Probe)
공부해서 하자

## 10. Config Map/ Persistence Volume
컨피그 맵 공부 해서 하자

## 12. Self-healing (Liveness Probe)
공부해서 하자




