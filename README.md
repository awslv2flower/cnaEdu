######################################
CNA 2일차 
######################################
[주제] 주문/배송을 활용한 EventStorming 설계 및 CNA구현
요건> 주문을 하면, 주문정보를 바탕으로 배송이 시작된다.
요건> 고객이 주문취소를 하게 되면, 주문정보는 삭제되나,
        배송에서는 (사후, 활용 위해) 취소 장부를 별도 저장한다.
요건> 주문 취소는 반드시 배송 취소가 전제대외야 한다.
요건> 주문과 배송 MSA는 게이트웨이를 통해 고객과 통신한다.

http://msaez.io/ 에서 NEW Project 수행.
1. Event(주황색) 추가 (더블클릭후 속성 정의. Post Persist:DB에 저장되고 난 후)
 - Pre Persist : DB commit 이전에 처리(seq 등 생성 이전). Post Persist : DB commit 이후에 처리.(seq 등 생성 이후)
2. order command 는 POST로. cancel command는 DELETE로
3. orderCanceled 에서는 Pre remove로 (보통 삭제는 Pre로 주는게 맞다.)
4. 다운받은 소스에서 gateway에 resources/application.yml에서 predicates 부분에 ,로 구분해주기.
          predicates:
            - Path=/deliveries/**,/cancellations/**
			
* 주키퍼 수행.
D:\kafka_2.13-2.6.0\bin\windows\ 로 경로이동
zookeeper-server-start.bat ../../config/zookeeper.properties
* KAFKA수행
D:\kafka_2.13-2.6.0\bin\windows\ 로 경로이동
kafka-server-start.bat ../../config/server.properties
* kafka-console-consumer --bootstrap-server localhost:9092 --topic shop --from-beginning 로 토픽 모니터링.
* netstat -ano | findstr "PID :808" 로 떠있는 프로세스 확인후 있으면 kill하기
  (taskkill /pid 12345 /f )
 
* 오더 발생시켜보기
http http://localhost:8081 (주문서버에 테스트)
http POST http://localhost:8081/orders productId="1001" qty=5

# shipped policy 구현하기
* delivery > src > main > java > shop > PolicyHandler 에서 wheneverOrdered_Ship 메소드 구현.
    http://msaschool.io/#/%EC%84%A4%EA%B3%84--%EA%B5%AC%ED%98%84--%EC%9A%B4%EC%98%81%EB%8B%A8%EA%B3%84/04_%EA%B5%AC%ED%98%84/04_%EB%8F%84%EA%B5%AC(MSAEz)%EA%B8%B0%EB%B0%98%20CNA%EA%B5%AC%ED%98%84
    에서 step-6하기

# OrderCanceled 에서 cancel command로 Req/Res(동기식) 호출하기
* order > src > main > java > shop > external > CancellationService 에서 interface 구현.
@FeignClient(name="delivery", url="http://localhost:8082") 로 url부분(원래는 delivery:8080으로 되있던듯) 변경

* order > src > main > java > shop > Order에서 onPreRemove 메소드에 아래 두줄 추가.
cancellation.setOrderId(this.getId());
cancellation.setStatus("Delivery canceled.");

* 주문하기 : http http://localhost:8081/orders productId=1007 qty=5 (POST는 생략가능하다)
{"eventType":"Ordered","timestamp":"20200820144334","id":1,"productId":"1007","qty":5,"me":true}
{"eventType":"Shipped","timestamp":"20200820144334","id":3,"orderId":1,"status":"SHIPPED","me":true}
{"eventType":"Ordered","timestamp":"20200820144417","id":2,"productId":"1010","qty":10,"me":true}
{"eventType":"Shipped","timestamp":"20200820144417","id":4,"orderId":2,"status":"SHIPPED","me":true}

* 주문삭제하기 : http DELETE http://localhost:8081/orders/1
{"eventType":"DeliveryCanceled","timestamp":"20200820144517","id":5,"orderId":1,"status":"Delivery canceled.","me":true}
{"eventType":"OrderCanceled","timestamp":"20200820144516","id":1,"productId":"1007","qty":5,"me":true}

* 전체확인하기 : http GET http://localhost:8081/orders/

* application.yml에서 url을 환경변수 처리
* order > src > main > java > shop > external > CancellationService 에서
@FeignClient(name="delivery", url="${api.url.delivery}") 로 변경.

* order > ~~ > application.yml 에서 아래 두개 추가.
api:
  url:
    delivery: http://localhost:8082
api:
  url:
    delivery: http://delivery:8080
	
# CLOUD에서 (MSAEZ terminal에서) 환경설정하기 (https://workflowy.com/s/msa/27a0ioMCzlpV04Ib#/06b292a9ac9d 참고)
* 접속 : https://class-skcc.signin.aws.amazon.com/console
   ID: ******** pw: ********* (region: ap-northeast-2)
* 로그인후 IAM > 사용자 > 자기 ID > 보안 자격 증명 > 액세스 키 만들기 >
   console에서 aws configure > 위에서 생성된 KEY 넣기.  region : 기본 , ouput : 기본 json 넣기.
* 클러스터 생성 : 
   eksctl create cluster --name CLUSTER명칭-eks --version 1.15 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3

* kubectl get all, kubectl get node 로 생성 확인.

* github repository 생성하기.
https://git-scm.com/ git 다운 및 설치.
CMD 창에 각 class별 디렉토리에 가서.(delivery)
git init
git config --global user.name [ID]
git config --global user.email [EMAIL]
git add .
git status
git commit -m "1st commit.."
git remote add origin https://github.com/[ID]/cna-delivery.git (요 주소는 git에서 repository 생성하고 나면 세팅 방법에 뜨는 주소)
git push -u origin master
이걸 order, gateway 도 생성하기.

다시 telnet에 가서
cd /home/
git clone https://github.com/[ID]/cna-delivery.git (github repository에서 Code 다운부분 클릭후 주소 확인)
이걸 order, gateway 도 생성하기.

