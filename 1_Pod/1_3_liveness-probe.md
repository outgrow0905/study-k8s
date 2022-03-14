## liveness probe
k8s에서는 기본적으로 컨테이너의 상태를 체크하여, crash 가 발생할 경우 자동으로 이를 재시작한다.
하지만, 어플리케이션에 일부 버그가 있어 pod가 비정상상태로 유지된 경우는 어떻게 대처할 수 있을까?
컨테이너 자체는 정상상태이기 때문에 이를 인지할 수 있는 방안을 마련해야 한다.

liveness probe를 사용해보자.


#### liveness probe 세팅하기
liveness probe는 세 가지로 세팅할 수 있다.
http get, tcp, exec 조건이다.

http get은 지정한 ip, port로 http get 요청을 보내고 응답 코드를 체크한다.
tcp는 지정된 컨테이너로 tcp 3-way handshake 성공여부로 pod의 상태를 체크한다.
exec는 컨테이너 내부에서 지정된 명령어를 사용하고 종료상태코드를 체크한다.

#### http get
~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: libeness-pod
    image: nginx
    livenessProbe:
      httpGet:
        port: 80
        path: /
~~~

pod를 조회해보면, liveness 에 다른 조건들을 설정할 수 있다는 것을 알 수 있다.

delay=0s: 컨테이너 시작 후, 0초 후에 liveness probe가 작동한다.
timeout=1s: 응답 제한시간이 1초이다.
period=10s: 체크 주기는 10초이다.
success=1: 성공으로 간주하는 횟수
failure=3: 실패로 간주하는 횟수

~~~sh
$ kubectl describe po liveness-pod | grep Liveness
~~~


#### http get 주의사항1
http get endpoint에 인증이 필요한지 여부를 체크해야 한다.
인증이 필요한데, 이를 구현하지 않으면 무한 재시작 할 것이다.

#### http get 주의사항2
어플리케이션 내부의 상태만 체크해야 한다.
예를 들어 외부 요인인 데이터베이스 서버의 상태를 체크한다면, 어플리케이션을 다시 시작하더라도 이를 해결하지 못할 것이다.

#### http get 주의사항3
가볍게 유지해야 한다.
프로브의 연산은 컨테이너의 자원(cpu, memory)를 사용하므로, 이를 무겁게 한다면 컨테아너가 사용할 수 있는 자원이 줄어들게 된다.