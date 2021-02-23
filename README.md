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
오더랑 결제 관계 구현

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




