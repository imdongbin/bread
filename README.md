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
- 동시 사용자 100명, 5초 동안 실시
```
# 시즈 부하 발생
siege -c100 -t5S -v --content-type "application/json" 'http://localhost:8081/orders POST {"item": "cake", "qty": 1}'

** SIEGE 4.0.2
** Preparing 100 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.42 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.43 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.44 secs:     217 bytes ==> POST http://localhost:8081/orders
...
HTTP/1.1 201     1.08 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.08 secs:     217 bytes ==> POST http://localhost:8081/orders

# 과도한 요청으로 인한 CB 동작, 요청 차단
HTTP/1.1 500     1.22 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.22 secs:     248 bytes ==> POST http://localhost:8081/orders

# 정상화
HTTP/1.1 201     1.50 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.50 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.52 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.55 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.56 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.57 secs:     217 bytes ==> POST http://localhost:8081/orders

# 다시 CB 동작
HTTP/1.1 500     1.59 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.60 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.60 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.60 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.60 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.61 secs:     248 bytes ==> POST http://localhost:8081/orders

# 다시 정상화
HTTP/1.1 201     1.64 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.65 secs:     217 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.78 secs:     217 bytes ==> POST http://localhost:8081/orders
...

# 서비스가 죽지 않고 열고 닫힘을 반복.

Lifting the server siege...
Transactions:                     86 hits
Availability:                  84.31 %
Elapsed time:                   4.92 secs
Data transferred:               0.02 MB
Response time:                  3.06 secs
Transaction rate:              17.48 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                   53.54
Successful transactions:          86
Failed transactions:              16
Longest transaction:            4.86
Shortest transaction:           0.42
```
- 84.31% 성공, 15.69% 실패

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

## 6. Deploy
- k8s에 MSA 배포
- 아래는 예시로 order 배포 script
```
mvn package
docker build -t 496278789073.dkr.ecr.ap-northeast-1.amazonaws.com/skcc13-order:v1 .
docker images
docker login --username AWS -p $(aws ecr get-login-password --region ap-southeast-1) 496278789073.dkr.ecr.ap-northeast-1.amazonaws.com/
aws ecr create-repository --repository-name skcc13-order --region ap-northeast-1
docker push 496278789073.dkr.ecr.ap-northeast-1.amazonaws.com/skcc13-order:v1
kubectl create deploy order --image=496278789073.dkr.ecr.ap-northeast-1.amazonaws.com/skcc13-order:v1 -n psn
kubectl expose deploy order --type="ClusterIP" --port=8080 -n psn
```
- k8s에 siege 설치 하여 rest api 테스트
```
# 시즈 설치 및 실행
kubectl run seige --image=ghcr.io/gkedu/siege-nginx -n psn
kubectl exec -it seige-74d7df4cd9-5t7mg -c seige -n psn -- /bin/bash

# rest api 테스트
# 빵 주문
http order:8080/orders item="cake" qty=1

# 결제
http pay:8080/payments orderId=? price=25000 action=apply

# 주문 확인 해서 수락 하기
http bakery:8080/bakeries orderId=? status="Confirmed"

# 주문 상태 갱신 확인
root@seige-74d7df4cd9-5t7mg:/# http order:8080/orders/1

HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 24 Feb 2021 08:37:04 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://order:8080/orders/1"
        },
        "self": {
            "href": "http://order:8080/orders/1"
        }
    },
    "item": "cake",
    "qty": 1,
    "status": "Confirmed"
}

```

## 8. Autoscale (HPA)
- 결제 서비스 폭주 상황을 가정 하여 pay 서비스에 HPA 적용
```
# pay 서비스 > deployment.yml 파일 수정 및 apply 적용
# 아래 cpu 스펙 추가
          resources:
            requests:
              cpu: "200m"
            limits:
              cpu: "500m"

# cpu target <unknown> 예방 위해 metrics-server 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# CPU 사용량이 50% 이상 넘어가면 레플리카를 10개 까지 증가 하는 것으로 HPA 설정 
kubectl autoscale deployment pay --cpu-percent=50 --min=1 --max=10 -n psn

# 시즈로 pay 부하 발생 시킴
siege -c30 -t30S -v http://pay:8080/payments

# pay 모니터링
# 5개 까지 늘어났다가 사용량이 감소하면서 다시 1개로 감소 하였음
root@labs--749286782:/home/project# kubectl get deploy pay -w -n psn
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
pay    1/1     1            1           2m44s
pay    1/4     1            1           3m53s
pay    1/4     1            1           3m53s
pay    1/4     1            1           3m53s
pay    1/4     4            1           3m53s
pay    1/5     4            1           4m8s
pay    1/5     4            1           4m8s
pay    1/5     4            1           4m8s
pay    1/5     5            1           4m8s
pay    2/5     5            2           5m10s
pay    3/5     5            3           5m11s
pay    4/5     5            4           5m17s
pay    5/5     5            5           5m32s
pay    5/1     5            5           9m43s
pay    5/1     5            5           9m43s
pay    1/1     1            1           9m43s

# 부하 발생 전 평소 상황. CPU 6% 수준이다.
root@labs--749286782:/home/project# kubectl get all -n psn
NAME                         READY   STATUS    RESTARTS   AGE
pod/bakery-ddc95f6d6-gwwqj   1/1     Running   0          3h8m
pod/order-676db985bb-z7hmc   1/1     Running   0          3h19m
pod/pay-7d675f54b8-xhcq6     1/1     Running   0          2m36s
pod/seige-74d7df4cd9-5t7mg   1/1     Running   0          3h16m
...
NAME                                      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/pay   Deployment/pay   6%/50%    1         10        1          5m34s

# 부하가 발생하여 CPU가 250% 수준까지 올라갔고 pay pod들도 설정에 따라 늘어나고 있다.
root@labs--749286782:/home/project# kubectl get all -n psn
NAME                         READY   STATUS              RESTARTS   AGE
pod/bakery-ddc95f6d6-gwwqj   1/1     Running             0          3h9m
pod/order-676db985bb-z7hmc   1/1     Running             0          3h20m
pod/pay-7d675f54b8-5msrc     0/1     ContainerCreating   0          2s
pod/pay-7d675f54b8-bzqzt     0/1     Running             0          2s
pod/pay-7d675f54b8-jp5zh     0/1     ContainerCreating   0          2s
pod/pay-7d675f54b8-xhcq6     1/1     Running             0          3m55s
pod/seige-74d7df4cd9-5t7mg   1/1     Running             0          3h17m
...
NAME                                      REFERENCE        TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/pay   Deployment/pay   250%/50%   1         10        1          6m52s

# pay 서비스가 늘어나면서 CPU는 8%까지 감소 하였다.
root@labs--749286782:/home/project# kubectl get all -n psn

NAME                         READY   STATUS    RESTARTS   AGE
pod/bakery-ddc95f6d6-gwwqj   1/1     Running   0          3h11m
pod/order-676db985bb-z7hmc   1/1     Running   0          3h22m
pod/pay-7d675f54b8-5msrc     1/1     Running   0          116s
pod/pay-7d675f54b8-bzqzt     1/1     Running   0          116s
pod/pay-7d675f54b8-jp5zh     1/1     Running   0          116s
pod/pay-7d675f54b8-mgpnc     1/1     Running   0          101s
pod/pay-7d675f54b8-xhcq6     1/1     Running   0          5m49s
pod/seige-74d7df4cd9-5t7mg   1/1     Running   0          3h19m
...
NAME                                      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/pay   Deployment/pay   8%/50%    1         10        5          8m46s

# 부하가 감소하면서 다시 1개로 감소
root@labs--749286782:/home/project# kubectl get all -n psn
NAME                         READY   STATUS    RESTARTS   AGE
pod/bakery-ddc95f6d6-gwwqj   1/1     Running   0          3h19m
pod/order-676db985bb-z7hmc   1/1     Running   0          3h30m
pod/pay-7d675f54b8-xhcq6     1/1     Running   0          13m
pod/seige-74d7df4cd9-5t7mg   1/1     Running   0          3h27m
...
NAME                                      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/pay   Deployment/pay   2%/50%    1         10        1          27m
```

## 9. Zero-downtime deploy (Readiness Probe)
- HPA 제거
```
root@labs-38102048:/home/project/bread/order# kubectl delete hpa pay -n psn
horizontalpodautoscaler.autoscaling "pay" deleted
```
- CB 제거
```
# order 서비스 내 application.yml 파일에서 hystrix 설정 제거
# 호출 받는 결제(pay) 서비스의 가공 부하 발생 처리 로직 제거
```
- siege로 배포 과정 모니터링
```
siege -c30 -t120S -v http://order:8080/orders

HTTP/1.1 200     0.20 secs:     344 bytes ==> GET  /orders
HTTP/1.1 200     0.27 secs:     344 bytes ==> GET  /orders
...
```
- Readiness Probe 설정 안된 상태, 새 버전으로 배포
```
kubectl set image deployment.apps/order order=496278789073.dkr.ecr.ap-northeast-1.amazonaws.com/skcc13-order:v2 -n psn
```
- siege 결과 확인
```
Transactions:                   2921 hits
Availability:                  73.50 %
Elapsed time:                   9.09 secs
Data transferred:               0.96 MB
Response time:                  0.09 secs
Transaction rate:             321.34 trans/sec
Throughput:                     0.11 MB/sec
Concurrency:                   29.17
Successful transactions:        2921
Failed transactions:            1053
Longest transaction:            0.52
Shortest transaction:           0.00
```
- 배포 과정에서 Availability가 평소 100%서 73.50%로 악화 되는 것을 확인
- 원인은 쿠버네티스가 아직 준비가 채 안된 업데이트 서비스가 생성 되자 마자 서비스 유입을 시키기 때문
- 이를 방지 하기 위해 Readiness Probe 설정
```
# deployment.yaml 의 readiness probe 의 설정:
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10

# 변경 내용 적용
kubectl apply -f kubernetes/deployment.yaml
```
- 동일한 시나리오로 재배포한 후 siege 결과 확인
```
Transactions:                  83801 hits
Availability:                 100.00 %
Elapsed time:                 119.98 secs
Data transferred:              27.49 MB
Response time:                  0.04 secs
Transaction rate:             698.46 trans/sec
Throughput:                     0.23 MB/sec
Concurrency:                   29.82
Successful transactions:       83801
Failed transactions:               0
Longest transaction:            1.16
Shortest transaction:           0.00
```
- 배포 과정에서 Availability가 100.00%로 확인 되기 때문에 무정지 배포 성공으로 판단.

## 10. Config Map
- Config Map의 환경변수를 읽어서 출력 하는 NodeJS 프로그램을 인터넷 브라우저 통해 호출 하는 방식으로 구현
```
# ConfigMap 생성
kubectl create configmap hello-cm --from-literal=language=java -n psn
kubectl get cm -n psn
kubectl get cm hello-cm -o yaml -n psn

# ConfigMap의 환경변수를 읽어 출력하는 NodeJS 어플리케이션
var os = require('os');
var http = require('http');
var handleRequest = function(request, response) {
    response.writeHead(200);
    response.end(" my prefered language is "+process.env.LANGUAGE+ "\n");
//log 
    console.log("["+
            Date(Date.now()).toLocaleString()+ 
            "] "+os.hostname());
}
var www = http.createServer(handleRequest); 
www.listen(8080);
```
- deploy, service 배포 후 external-ip 확인
<img width="1089" alt="스크린샷 2021-02-25 오후 1 46 39" src="https://user-images.githubusercontent.com/58290368/109104769-f3c85600-776f-11eb-9eee-b3f8e010bdf9.png">

- 인터넷 브라우저에서 external-ip로 접근하여 결과물 확인
<img width="862" alt="스크린샷 2021-02-25 오후 1 49 50" src="https://user-images.githubusercontent.com/58290368/109104974-5f122800-7770-11eb-9f40-67c9a3ae8ac3.png">

## 12. Self-healing (Liveness Probe)
공부해서 하자
