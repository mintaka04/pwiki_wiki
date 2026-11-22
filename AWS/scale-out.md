
# 처리량 늘리기
https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-throughput-horizontal-scaling-and-batching.html#request-batching

To maximize throughput:
- Scale producers and consumers horizontally by adding more instances of each.
- Use action batching to send or receive multiple messages in a single request, reducing API call overhead.




수평확장 = horizontal scaling

Horizontal scaling involves increasing the number of message producers (which make SendMessage requests) and consumers (which make ReceiveMessage and DeleteMessage requests) in order to increase your overall queue throughput. You can scale horizontally in three ways:

- Increase the number of threads per client
- Add more clients
- Increase the number of threads per client and add more clients




배치
    -   점원이 주문 여러개 받아서 한번에 바리스타한테 전달
수평확장
    1. 스레드 늘리기, 코루틴
        - 바리스타 수가 늘어난것
    2. 워커/클라이언트 늘리기
        - 카페를 더 세우는 것
    3. 둘다 하기.


그리고 SQS는 요청단위로 요금이 청구되어서 배치를 쓰면 (액션배치) 비용 절감도 가능하다...

그런데 또 배치를 쓴다? 그러면 배결 요청에 대한 처리가 느려질 수 있고... 즉, 실시간성이 중요하면 못쓰고.
그리고 또 여기가 빨라지니까 뒤쪽에서 이만큼 늘어난? 요청을 못 받을 수도 있고?
또 SQS가 한번에 받는 요청도 크기 제한이 있대.
또 SQS에서는 상태코드를 200 ok로 줬는데 까보니 10개 중에 몇 개는 실패했다고 body에 메세지가 있다던지...


Important
A batch request might succeed even though individual messages in the batch failed. After a batch request, always check for individual message failures and retry the action if necessary...!!!!!



Note : SQS Cloudwatch 관련.
The metrics can take up to 15 minutes after the queue starts to become available.. !!!!!




---

### EC2 Auto Scaling

https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html

- Auto Scaling Group (ASG) = EC2 인스턴스 모음!
각 그룹에 최소/최대 인스턴스 수를 지정 가능...

태스크 단위가 아니라 인스턴스 단위로 확장된다... 

주요 기능
- 상태 모니터링 (ECS Fargate와 다르게 '인스턴스'의 상태만)
- 근데 더 체크 및 모니터링 하고 싶은게 있으면 추가 설정 가능
- 가용 영역 간 용량 균형 조정 (AZ Rebalance)
- 같은 그룹에서도 다른 인스턴스 유형 사용 가능.
- 스팟 인스턴스도 자동 교체해줌

스팟 인스턴스
    - 좀 싸게 쓰는데 누가 쓰겠다고 하면 비켜줘야함(즉, 중지됨)
온디맨드 인스턴스
    - 그냥 켜서 씀. 그리고 이제 쓴만큼 돈 냄.

근데 어차피 인스턴스 부팅되는 속도가 느린데 뭐가 다르냐... 긴 한데.
스팟 인스턴스를 쓸 경우는 좀 더 빈번하게 그런 경우가 발생할 수 있고,
또 한꺼번에 여러 대가 영향을 받을 수도 있다!

부하테스트 중에 배포는 안해봤다....!

- 배포되었을때 소수의 인스턴스부터 순차적으로 업데이트.. 카나리배포? 즉. 가용성 지키는거...
    - ECS의 경우 새 태스크를 띄우고 기존 태스크를 내림. 이것도 가용성 지키는거...
        - Rolling Update
            - 이전거 놔두고 새로운 버전 태스크를 띄움 -> 준비완료되면 트래픽 넘기고 이전거 종료
            - 이 때 최소 최대 비율 정할 수 있어서 그걸로 속도, 안정성 조절가능하다고.

        인스턴스 10개가 돌고있는데 배포를 할겁니다!

            블루-그린 배포 : 구버전 10개(블루), 신버전 10개(그린) 두고 트래픽을 넘김. 100% 한꺼번에 넘긴다!
            카나리배포 : 소수의 트래픽에게만 먼저 흘려보내고 확인. 작은 규모로 검증하기 위해서. 검증!!! 비율 단계적 조절!!!
                            트래픽 조금만 신버전에 주고 정상적인지 확인 하고 조금씩 비율 늘려나가는거. 안전한지 검증이 목적.
            롤링 업데이트 : 조금씩 갈아 끼운다... 1개씩... 서비스를 멈추지 않고 조금씩 바꿔나간다....  멈추지 않는게 목적.
    - 즉 ECS도 롤링, EC2도 롤링..
- 라이프사이클 훅 : 인스턴스 시작/종료 전에 브레이크. 지정한 동작할때까지 기다려줌.