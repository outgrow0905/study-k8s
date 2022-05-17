## Affinity and anti-affinity
Pod를 스케줄링하는 또 다른 방법으로는 affinity and anti-affinity 가 있다.
affinity의 사전적 정의는 _'a liking or sympathy for someone or something'_ 으로 쓰여져 있다.
무언가를 '선호','좋아하는' 것이다.

affinity는 Pod에 설정하는데, 노드와 Pod에 대하여 설정할 수 있다.  
'좋아하는/싫어하는' 노드나 Pod를 설정하여 스케줄링되도록 하는 것이다. 노드나 Pod에 설정하는 방법이 조금 다른데, 노드부터 살펴보자.

## Node affinity
~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
~~~
`requiredDuringSchedulingIgnoredDuringExecution`을 나누어서 살펴보자. 
`requiredDuringScheduling`은 스케줄링 되는 동안 필수로 충족되어야 하는 조건이라는 의미이다.
`IgnoredDuringExecution`은 스케줄링 된 이후에는 무시된다는 의미이다. 이미 스케줄링 되었는데, 노드에 다른 라벨이 설정되어도 evict 되지 않는다는 의미겠다.

`preferredDuringSchedulingIgnoredDuringExecution`을 보면, 위와 똑같은데 표현이 `preferred`이다. 
`required`의 소프트 버전이라고 생각하면 좋을 것 같다.

`weight`는 중요도이다. 노드 affinity는 여러 조건을 설정할 수 있고, 만약 여러개의 노드가 이 조건에 부합한다면, 중요도가 높은 노드가 선택된다.


## Pod affinity
~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
~~~

노드 affinity와 비슷한데, `topologyKey`가 새로 생겼다. Pod affinity는 이렇게 작동한다.  

1. `matchExpressions`에 부합하는 Pod를 찾는다. (노드 affinity에서는 `matchExpressions`에 부합하는 '노드'를 찾았다.)
2. 해당 Pod가 스케줄링된 노드의 라벨에서 `topologyKey`에 설정된 키-값을 조회한다.
3. 찾은 키-값에 부합하는 노드 리스트에 Pod를 스케줄링한다.

위의 예시에서 `podAffinity`만 다시 해석해보자.

1. Pod 라벨이 `security=S1`인 Pod를 조회한다.
2. 1.에서 찾은 Pod가 소속된 노드에서, 노드 라벨의 키가 `topology.kubernetes.io/zone`인 정보들을 조회한다.
   예를 들어 Pod가 2개라면, 2개의 Pod에서 각각 `topology.kubernetes.io/zone=KR`, `topology.kubernetes.io/zone=US` 키-값이 조회될 수 있다.
3. 2.에서 조회된 노드 라벨 조건에 부합하는 노드에 Pod를 스케줄링한다.
   예를 들면, `topology.kubernetes.io/zone=KR`, `topology.kubernetes.io/zone=US` 조건에 부합하는 노드들 중 하나에 스케줄링 될 것이다.
   

## Reference
- https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity
- https://kubernetes.io/docs/reference/labels-annotations-taints/
