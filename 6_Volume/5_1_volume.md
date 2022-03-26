## Volume
컨테이너의 파일시스템은 일시적이다. 컨테이너의 생성주기와 함께하는데, 이는 컨테이너가 삭제되면 파일시스템도 같이 삭제된다는 것을 의미한다.  
이로 인한 첫 번째 문재는 컨테이너 재시작 시 파일시스템의 유실이다. kubelet이 컨테이너를 깨끗한 상태로 재시작한다.   
이는 Pod에서 여러개의 컨테이너가 파일시스템을 공유할 때에 더 문제이다. 

kubernetes는 이를 volume 추상화를 통해 해결한다. 

kubernetes는 여러 유형의 volume을 제공한다. Pod는 동시에 여러 유형의 volume을 동시에 사용할 수도 있다.  
ephemeral(일시적인) volume은 Pod의 생명주기를 공유하지만, persistent(영구적인) volume은 Pod의 생명주기와 무관하게 존재할 수 있다.
Pod가 종료되면 kubernetes는 ephemeral volume을 삭제하지만 persistent volume은 삭제하지 않는다는 의미이다.
 
## emptyDir
`emptyDir` 은 Pod가 node에 할당될 때에 생성되며, Pod가 해당 node에서 운영되는 동안만 존재한다.  
이름처럼 `emptyDir`은 비어진채로 시작한다. 같은 Pod에 속한 모든 컨테이너들은 같은 `emptyDir` volume에 읽고 쓸 수 있다.  
그리고 각 컨테이너에서는 원하는 path에 `emptyDir`를 mount할 수 있다.  

`emptyDir`은 해당 Pod가 올라간 node의 실제 디스크에 생성되므로, node의 유형에 따라 성능이 결정된다.    

kubernetes는 이를 보완하기 위해 node의 메모리를 사용할 수 있는 기능도 제공한다.    
`emptyDir.medium`를 `"Memory"`로 설정한다면, RAM 기반의 파일시스템인 `tmpfs`에 마운트한다.   
이는 속도가 빠르지만 컨테이너의 메모리 제한만큼만 사용할 수 있기 때문의 사용에 주의해야 한다.  
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)의 `SizeMemoryBackedVolumes` 으로 크기를 조절할 수 있다.  
별도의 세팅이 없다면 Linux host의 50% 만큼 메모리를 사용할 수 있다.

#### example
~~~
$ vi fortuneloop.sh
#!/bin/bash
trap "exit" SIGINT
while :
do
        echo $(date) Writing fortune to /var/htdocs/index.html
        /usr/games/fortune > /var/htdocs/index.html
        sleep 10
done

$ vi Dockerfile
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT /bin/fortuneloop.sh

$ docker build -t outgrow0905/fortune .
$ docker push outgrow0905/fortune
~~~

~~~
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    name: fortune
  clusterIP: None
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: fortune
  labels: 
    name: fortune
spec:
  containers:
  - image: outgrow0905/fortune
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

위의 예시는 `html-generator` 컨테이너가 10초마다 `/var/htdocs/index.html` 에 파일을 생성한다.  
그리고 `web-server` 컨테이너는 `/usr/share/nginx/html` 경로를 읽는다.
두 컨테이너의 `/var/htdocs` 경로와 `/usr/share/nginx/html` 경로는 공유 volume 이다. 

## nfs




## Reference
- https://kubernetes.io/docs/concepts/storage/volumes/