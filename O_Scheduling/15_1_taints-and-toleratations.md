## Taints and Tolerations
이전에는 Pod에 `nodeSelector`를 이용하여 어떤 노드에 Pod를 스케줄링할 지 설정을 하였다. 하지만 기능이 많이 추가되었다.
먼저 Taints and Tolerations를 살펴보자.  

Taint의 뜻은 _'오염', '더러움'_ 이다.    
Toleration의 뜻은 _'인내', '참음'_ 이다.

Taint는 노드에 설정하고, Toleratation은 Pod에 설정한다. 어떻게 작동할까?  
*오염*된 노드에는 기본적으로 어떤 Pod도 스케줄될 수 없고, 이를 *인내*할 수 있는 Pod만 해당 노드에 스케줄링될 수 있다.  

##### example
~~~
# set
$ kubectl taint nodes node1 key1=value1:NoSchedule
# unset
$ kubectl taint nodes node1 key1=value1:NoSchedule-
~~~
기본 형식은 kubernetes에서 익숙한 `<key>=<value>:<effect>`의 형식이다. 
이제 node1에는 기본적으로 스케줄링될 수 없고, 아래의 Toleratation을 설정한 Pod만 설정될 수 있다.

~~~yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
~~~  
operator가 `Equal` 이라면, key, value, effect 가 전부 일치하는 Taint가 설정된 노드에 스케줄 될 수 있다. 

~~~yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
~~~
operator가 `Exists` 라면, value는 설정되어서는 안된다. key, effect 가 일치하는 Taint가 설정된 노드에 스케줄 될 수 있다.
`Exists` 에서 key, effect를 설정하지 않으면 전부 tolerate 한다는 의미이다. 예를 들어, operator를 `Exists`로 설정하고 key, effect를 설정하지 않는다면,
모든 노드의 Taint를 무시하는 슈퍼 설정이 된다. 

##### effect
`NoSchedule`: 스케줄링하지 않는다. 이미 생성된 Pod를 evict 하지는 않는다.
`PreferNoSchedule`: `NoSchedule`의 소프트한 버전이다. 
`NoExecute`: 스케줄링 하지 않을 뿐더러, 이미 있는 Pod도 evict 한다. 


##### tolerationSeconds
kubernetes는 다양한 이유로 노드에 이상이 생겼다고 판단되면 노드에 Taint를 추가한다.
예를 들어, 갑자기 노드의 네트워크 이상이 생기면 kubernetes는 `node.kubernetes.io/network-unavailable` taint를 노드에 추가한다.
혹은, 디스크가 가득 찼다면, `node.kubernetes.io/disk-pressure` taint를 노드에 추가한다.

그런데 이 현상이 일시적이거나, 빈번하게 발생하고 복구되는 현상이라면?  
  
모든 Pod가 즉시 evict 될 것이다. 노드는 곧 복구될 예정인데 말이다. 이를 위한 설정이 `tolerationSeconds` 이다.
혹시 노드가 `NoExecute` 상태로 변경되더라도 잠깐 비빌수 있는 시간이라고 생각하면 된다.  
아래와 같이 설정한다.

~~~yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
~~~


## Reference 
- https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/