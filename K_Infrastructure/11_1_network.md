## network
flannel CNI를 기준으로 패킷을 추적해보려고 한다.


## nodePort 패킷을 따라가보자.
~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname
spec:
  selector:
    app: hostname
  ports:
  - port: 8080
    nodePort: 31921
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname
  labels:
    app: hostname
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hostname
  template:
    metadata:
      labels:
        app: hostname
    spec:
      containers:
      - name: hostname
        image: outgrow0905/hostname
        ports:
        - containerPort: 8080
~~~

#### iptables 확인
위의 세팅을 하였을 떄, iptables 가 어떻게 세팅되는 지 확인해보자.
tcpdump는 iptables 보다 먼저 적용되므로, iptables의 정책이 적용되기 전의 패킷을 볼 수 있다.

~~~
tcpdump -i eth0 -ne tcp  port 31921 -vv -c 10
~~~