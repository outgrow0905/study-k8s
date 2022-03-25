## Service
실행되고 있는 Pod 들을 네트워크에 노출할 수 있는 추상화된 방식이다.  
kubernetes는 Pod에 고유한 ip주소와 dns를 부여하고, 부하분산을 할 수 있다.  

Pod는 클러스터에서 원하는 상태에 따라 언제든 생성/삭제될 수 있다.   
예를 들어, Deployment의 replica 수를 조정하여 Pod의 수를 조절할 수 있다.  
이러한 가정에서는 시간이 지나면 특정 Deployment에 소속된 Pod들의 ip들이 계속 변화한다.

이러한 환경에서, 아래의 문제를 생각해보자.  
frontend Pod 100대가 database Pod 2대를 바라보고 있다.   
만약, database Pod를 확장한다면, 100대의 fe 서버에 이를 세팅해야 하는가?  
혹은, database Pod 일부가 비정상종료되었다가 정상화되면서 ip가 바뀌었다면 이를 어떻게 추적할 것인가?  
더 나아가, database Pod의 변화를 감지할 수는 있는가?  

Service를 사용하자.  


#### example
~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports: # 여러개의 port를 노출할 수 있다.
    - protocol: TCP
      port: 80
      targetPort: 9376 # 설정하지 않는다면, port에 설정한 것과 동일하게 세팅된다.
~~~


## Services without selectors
selector를 설정하지 않는다면, Service 리소스는 Endpoint 리소스를 자동으로 생성해주지 않는다. 따라서, 수동으로 생성해야 한다.  
어떨 때 이렇게 해야할까?

- 다른 Namespace에 속한 Service를 호출해야하는 경우
- 같은 label을 가진 Pod 중에서 일부만 트레픽을 보내어 부하분산 테스트를 해야하는 경우

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376   
---
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42 # cluster ip 를 사용할 수 없다.
    ports:
      - port: 9376
~~~

## Traffic policies
#### External traffic policy / Internal traffic policy
클러스터 내부와 외부에서 접근하는 트레픽은 각각 ```spec.internalTrafficPolicy```와 ```spec.externalTrafficPolicy```에서 설정할 수 있다.  
```Cluster```와 ```Local``` 두 값으로 가능하다.  

```Cluster```가 default 설정이며 모든 트레픽을 모든 endpoint 로 전송한다.  
```Local```은 트레픽이 들어온 node 내부의 endpoint로만 트레픽을 전송한다. 만약, node 내에 endpoint가 없다면 kube-proxy는 트레픽을 Service로 전송하지 않는다.  

> FEATURE STATE: Kubernetes v1.22 [alpha]  
만약 [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)에서 ```ProxyTerminatingEndpoints``` 기능을 활성화한다면,  
kube-proxy는 node내의 로컬 endpoint(node-local)들의 정상상태여부를 확인한다. 그리고 모든 node-local endpoint가 비정상상태라면 마치 ```Cluster```로 설정한 것과 같이 트레픽을 다른 node의 정상 endpoint로 전송한다.  
이러한 설정은 ```NodePort``` Service 환경에서, 외부로부터 접근하는 트레픽을 우아하게 다른 Service로 이전할 수 있게 한다.  
하지만, 이러한 환경에서도 node가 아직 node pool에 남아있고, Pod가 종료되고 있는 상태라면 트레픽이 유실될 수 있으니 조심해야 한다.


## Discovering services
Pod 입장에서 다른 Service를 어떻게 알 수 있을까?  
kubernetes는 두 가지 방법을 제공한다.   

#### Environment variables
Pod를 실행시킬 때에 kubelet은 활성화된 모든 Service의 host와 port를 황경변수로 추가한다. ```{SVCNAME}_SERVICE_HOST```와 ```{SVCNAME}_SERVICE_PORT```이다.  
Service를 먼저 생성했다고 가정하고 Pod를 실행하면 Service에 해당하는 환경변수들이 주입된다.  
환경변수는 Pod가 생성되기 전에 Service가 먼저 생성되어 있어야 주입된다. DNS를 사용한다면 이러한 문제를 걱정하지 않아도 된다.  

~~~
$ kubectl exec -it ${pod-name} -- env
~~~ 

~~~
MY_SERVICE_PORT_80_TCP_PROTO=tcp
MY_SERVICE_PORT_80_TCP_PORT=80
MY_SERVICE_PORT_80_TCP_ADDR=10.99.155.51
MY_SERVICE_PORT_80_TCP=tcp://10.99.155.51:80
MY_SERVICE_SERVICE_PORT=80
MY_SERVICE_PORT=tcp://10.99.155.51:80
MY_SERVICE_SERVICE_HOST=10.99.155.51
~~~


#### DNS
kube-system namespace로 생성된 Pod를 조회하면 codedns를 확인할 수 있다.   
이 Pod는 kubernetes API를 활용하여 새로 생성되는 Service를 모니터링하고 DNS record에 저장한다.

```<service-name>.<namespace>.svc.cluster.local```의 형식으로 도메인이 생성된다.  
같은 namespace 내의 Service에서는 service 명 만으로 접근이 가능하지만, 다른 namespace에서는 FQDN을 모두 명시하여야만 접근이 가능하다.


## ServiceTypes
Service는 클러스터 내부에서만 사용될 수도 있고, 외부로의 공개가 필요할 수도 있다.  
혹은, 외부의 사이트 (ex. www.naver.com) 를 마치 kubernetes 의 서비스처럼 이용하고 싶을 수도 있다.  

```ClusterIP```, ```NodePort```, ```LoadBalancer```, ```ExternalName``` 으로 구성이 가능하다.

- ClusterIP: default 이다. 클러스터 내부에서만 사용할 수 있다.  
- NodePort: 클러스터에 소속된 모든 node에 같은 port를 오픈한다. 이를 통해, <NodeIP>:<NodePort> 의 경로로 외부에서 Service로 접근할 수 있다.  
- LoadBalancer: 클라우드 서비스(ex. aws)에서 제공하는 기능이다. 각 클라우드 서비스에서 자동으로 load balancer를 생성하여, ```NodePort```로 라우팅한다.
- ExternalName: 외부의 사이트를 마치 kubernetes의 서비스처럼 이용할 수 있다. 예를 들어 www.naver.com을 마지 클러스터 내부에서 naver 라는 이름의 Service로 등록된 것처럼 이용할 수 있다. 클러스터 내부에서 ```CNAME``` 을 등록하기 때문에 가능하다.


#### NodePort
NodePort가 할당하는 port의 기본 범위는 30000-32767 이며, 이는 ```--service-node-port-range``` 으로 조절이 가능하다.  
혹은, Service 생성 시 ```spec.ports[*].nodePort``` 필드로 할당할 수도 있다.
또한, 지정한 port-range 내에서 특정 port를 할당하고 싶다면 Service 생성 스크립트의 nodePort 필트를 사용하면 된다.

특정 ip 범위를 할당하고 싶다면, kube-proxy 의 ```--nodeport-addresses```를 통해 ip 범위를 지정할 수 있다. ```(ex. --nodeport-addresses=127.0.0.0/8)```

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
      # targetPort를 지정하지 않는다면, port 와 동일한 값을 자동으로 세팅한다.
    - port: 80
      targetPort: 80
      # Optional field
      # 지정하지 않는다면 default: 30000-32767 범위 내에서 자동으로 할당한다.
      nodePort: 30007
~~~

#### LoadBalancer
클라우드 사업자가 제공하는 기능이다.  
외부의 트레픽은 Pod로 바로 전송된다. 따라서, 외부에서만 접근되는 Service라면 사실상 NodePort는 필요가 없다.  

그래서 이러한 경우 NodePort를 자동으로 할당하는 기능을 제어할 수 있다. ```spec.allocateLoadBalancerNodePorts``` 를 ```false``` 로 설정하면 된다.  
default는 ```true```이다. 주의할 점은 이미 생성된 Service의 스펙을 ```false```로 설정하더라도 기생성된 NodePort를 제거하지는 않는다는 점이다.


## References
- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/