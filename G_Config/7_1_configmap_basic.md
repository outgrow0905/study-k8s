## Docker CMD & ENTRYPOINT
docker `CMD` 명령과 `ENTRYPOINT` 명령을 이해해보자.

#### example
INTERVAL 변수를 받을 수 있도록 fortune 스크립트를 수정해보자.

~~~
$ vi fortuneloop.sh

#!/bin/bash
trap "exit" SIGINT
INTERVAL=$1
echo Configured to generate new fortune every $INTERVAL seconds
mkdir -p /var/htdocs
while :
do
        echo $(date) Writing fortune to /var/htdocs/index.html
        /usr/games/fortune > /var/htdocs/index.html
        sleep $INTERVAL
done
~~~
~~~
$ vi Dockerfile

FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]

$ docker build -t outgrow0905/fortune:args .
$ docker push outgrow0905/fortune:args
$ docker run -it outgrow0905/fortune:args    ## 기본 10초로 실행
$ docker run -it outgrow0905/fortune:args 2  ## 환경변수 2초로 세팅하여 실행
~~~



## kubernetes command & args
kubernetes에서 실행하려는 컨테이너를 정의할 때에 command와 args를 통해, ENTRYPOINT와 CMD를 재정의 할 수 있다.  

`command`는 `ENTRYPOINT`에 매칭되고, `args`는 `CMD`에 매칭된다.
이 말은, `command`는 `ENTRYPOINT`를 덮어쓰고, `args`는 `CMD`를 덮어쓴다는 의미이다.  
`command`가 `CMD`를 덮어써야 할 것 같은데, 그렇지 않다. 잘 구분하자.

`ENTRYPOINT ["sleep"]`, `args: ["2"]` 이면, `sleep 2` 의 명령어로 이미지가 실행될 것이다.  
`ENTRYPOINT ["sleep"]`, `command: ["echo", "hello"]`이면, `sleep` 명령어는 무시되고 `echo hello`의 명령어만 실행될 것이다.  
`ENTRYPOINT ["sleep"]`이 있는데 `CMD`와 `args`로 아무것도 전달하지 않는다면, 오류가 날 것이다. 

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
  labels:
    name: fortune
spec:
  containers:
  - image: outgrow0905/fortune:args
    args: ["2"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
~~~

## env
위의 예제에서 $INTERVAL를 command 로 받는 부분을 삭제하고, 이를 주입하는 방법을 알아본다. 

~~~
$ vi fortuneloop.sh

#!/bin/bash
trap "exit" SIGINT
echo Configured to generate new fortune every $INTERVAL seconds
mkdir -p /var/htdocs
while :
do
        echo $(date) Writing fortune to /var/htdocs/index.html
        /usr/games/fortune > /var/htdocs/index.html
        sleep $INTERVAL
done

$ docker build -t outgrow0905/fortune:env .
$ docker push outgrow0905/fortune:env
~~~

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
  labels:
    name: fortune
spec:
  containers:
  - image: outgrow0905/fortune:env
    env:
      - name: INTERVAL  # 컨테이너 생성시에 환경변수를 주입한다.
        value: "5"
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
~~~


## 동일한 이미지 태그로 update하여 push 할 때 주의할 점
어플리케이션을 수정하고 동일한 이미지 태그로 변경사항을 push 하는 것은 해서는 안될 일이지만, 일하다보면 충분히 그럴 수 있다고 생각한다.  

다만 이런 경우 염두해두어야 할 부분이 있다.  
워커노드에서 이미지를 한 번 가져와서 저장하면 동일 이미지로 Pod를 생성할 때에 다시 pull 받아오지 않는 것이 기본정책이라는 점이다.  
이런 경우 최악으로 발생할 수 있는 시나리오는, 해당 이미지가 저장되지 않는 노드는 변경된 이미지로 Pod를 생성하고, 이미 저장되어 있던 노드는 변경된 부분이 반영되지 않은 Pod를 띄우는 것이다.

이것을 설정하는 태그는 `imagePullPolicy`이다. 태그가 latest가 아니라면, 기본값은 `ifNotPresent`(저장되어있지 않을 경우에만 pull)이다.  
이를 `Always`로 변경하면 해결된다.  

가장 좋은 방법은 `Always` 정책을 쓰는 것이 아니라, 이미지 변경 시마다 새로운 태그로 관리하는 것임을 잊지말자.
 
