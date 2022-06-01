## Network Policy
kubernetes에서는 Network Policy도 객체로 관리할 수 있다.  
예를 들어, 특정 Pod로부터의 ingress 트레픽만 허용하거나, 특정 Pod를 호출하는 egress 트레픽만 허용할 수 있다.  
혹은, 특정 namespace 로부터의 트레픽만 허용할 수 있다.  
마지막으로, 특정 ip 대역으로도 제어가 가능하다.

Pod는 기본적으로 `All Allow`정책을 적용한다. 기본적으로는 들어오고 나가는 트레픽에 대해서 전혀 제한을 하지 않는 것이다.    
하지만, ingress, egress 등 Network Policy을 적용하면 해당 정책에 설정한 트레픽만 허용하게 된다.

기본적으로 정책은 OR 조건으로 적용된다. 하나의 정책에라도 허용이 된다면 통신이 가능하다.  
하지만, 통신하는 두 Pod에 모두 Network Policy가 적용되어 있다면, 통신하는 두 Pod의 ingress, egress 정책을 모두 통과해야 한다.

## example
~~~yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
~~~

## Reference
- https://kubernetes.io/docs/concepts/services-networking/network-policies/