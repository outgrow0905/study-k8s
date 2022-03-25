## DNS for Services and Pods
kubernetes는 Service와 Pod에 DNS record를 생성한다.  
따라서, Service나 Pod에 IP대신에 DNS로 접근할 수 있다.

kubernetes DNS는 DNS Pod와 Service를 관리한다.   
그리고, kubelet은 모든 컨테이너가 DNS를 해석할 때에 DNS Service의 IP를 이용하도록 한다.  

기본적으로 Pod가 DNS를 조회할 때에는, Pod가 소속된 namespace와 cluster 도메인을 추가하여 조회한다.  
따라서, Pod가 어떤 namespace에 속하느냐에 따라 같은 DNS 조회를 하더라도 다른 결과를 받을 수 있다.  
namespace를 명시하지 않은 DNS 조회는 같은 namespace에 한정되어 조회된다.    
따라서, 다른 namespace에 속한 Service를 조회하려면, namespace를 명시해야 한다.

```test``` namespace에 속한 Pod가 ```prod``` namespace에 속한 ```data``` Service에 접근한다고 가정해보자.
data 를 조회하려고 하면 아무 결과도 조회할 수 없다. 다른 namespace이고, 해당 Pod가 소속된 ```test``` namespace 에는 ```data``` Service가 없기 때문이다.

DNS 조회는 Pod의 ```/etc/resolv.conf``` 파일을 조회하여 확장된다.  
위의 예시에서 ```prod``` namespace에 속한 ```data``` 조회는 ```test.data.svc.cluster.local``` 으로 확장되었기 때문에 조회되지 않은 것이다.  
원래 목적을 달성하기 위해서는 ```prod.data.svc.cluster.local``` 으로 명시하여 조회해야 한다.

## DNS for Pod
Pod 생성 시 아래의 형식으로 DNS가 생성된다.  
`pod-ip-address.my-namespace.pod.cluster-domain.example`  

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
      app: webserver
spec:
  containers:
  - image: outgrow0905/hostname
    name: my-pod
    ports:
    - containerPort: 8080
      protocol: TCP
~~~

~~~
$ kubectl -it exec my-pod -- /bin/bash
$ root@my-pod:/# curl 10-244-3-34.default.pod.cluster.local:8080
  You've hit my-pod
$ root@my-pod:/# curl my-pod:8080
  You've hit my-pod
~~~

#### Pod를 Service로 노출했을 떄
Pod를 Service로 노출할때에 Pod는 아래의 형식으로 도메인이 부여된다.  
`pod-ip-address.service-name.my-namespace.svc.cluster-domain.example`

~~~
$ root@my-pod:/# curl my-service:8080
  You've hit my-pod
$ root@my-pod:/# curl 10-244-3-34.my-service.default.svc.cluster.local:8080
  You've hit my-pod
~~~

#### Pod의 hostname과 subdomain 필드
Pod의 hostname은 `metadata.name` 값으로 설정된다.     
하지만 Pod생성시에 `hostname` 필드를 이용하면, 이를 우선적용하여 hostname을 적용한다.  
또한, Pod생성시에는  `subdomain` 필드도 이용할 수 있다. 

headless Service와 이를 잘 결합하면 Pod에 이를 활용한 FQDN을 부여할 수 있다.  
default namespace라면 아래의 형식으로 fqdn이 생성된다.

`<hostname>.<subdomain>.default.svc.cluster.local`

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: my-subdomain
spec:
  selector:
    name: subdomain
    clusterIP: None
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: my-subdomain-pod
  labels:
    name: subdomain
spec:
  hostname: foo
  subdomain: my-subdomain
  containers:
  - image: outgrow0905/hostname
    name: my-subdomain-pod
    ports:
    - containerPort: 8080
      protocol: TCP
~~~

~~~
$ root@foo:/#  curl foo.my-subdomain.default.svc.cluster.local:8080
  You've hit foo
~~~

## Reference
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config