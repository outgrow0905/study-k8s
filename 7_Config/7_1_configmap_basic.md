## Docker CMD & ENTRYPOINT
docker CMD 명령과 ENTRYPOINT 명령을 이해해보자.

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
command는 ENTRYPOINT에 매칭되고, args는 CMD에 매칭된다.

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

