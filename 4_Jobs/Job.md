## Job
Job은 하나 이상의 Pod를 생성하고, 설정한 횟수만큼 정상종료 될 때까지 Pod를 실행한다.    
Pod가 성공적으로 정상종료하면, Job은 정상종료여부를 확인한다.    
Job은 여러개의 Pod를 병렬적으로 사용할 수도 있다.    
Job을 스케쥴링하고 싶다면 CronJob을 사용하면 된다.

#### exercise
~~~yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      parallelism: 2 # 병럴 처리 Pod 개수
      completions: 5 # 총 수행되어야 하는 Pod 개수
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never # Pod와 달리 Job에서는 필수값이며 Never, OnFailure 만 허용된다.
  backoffLimit: 4
~~~

~~~
$ kubectl get -w pod
~~~

![job watch 1](./img/job-watch-1.png)

![job watch 2](./img/job-watch-2.png)

#### Indexed Job
비디오 파일을 병렬로 렌더링하는 Job을 생각해보자.  
이를 병렬처리하기 위해서는 각각의 Job이 어디서부터 렌더링을 해야하는 지 알고 있어야 한다.  
예를 들어, 1번 Job은 1-10까지의 프레임을 할당하고, 2번 Job은 11-20까지의 프레임을 할당해야 할 것이다.  
Indexed Job에서는 각 Job에서 ```$JOB_COMPLETION_INDEX``` 환경변수를 할당한다.  
각 Job에 0부터 1씩 증가하는 숫자를 할당해주는 것인데, 이를 활용하여 특정 변수나 파일을 Job에 전달할 수 있다.

아래의 예시는 각 Job 시작 전에, ```$JOB_COMPLETION_INDEX``` 환경변수를 활용하여 파일을 생성하고 이를 Job에 전달하는 예시이다.

~~~yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: 'indexed-job'
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      initContainers:
      - name: 'input'
        image: 'docker.io/library/bash'
        command:
        - "bash"
        - "-c"
        - |
          items=(foo bar baz qux xyz)
          echo ${items[$JOB_COMPLETION_INDEX]} > /input/data.txt
        volumeMounts:
        - mountPath: /input
          name: input
      containers:
      - name: 'worker'
        image: 'docker.io/library/busybox'
        command:
        - "rev"
        - "/input/data.txt"
        volumeMounts:
        - mountPath: /input
          name: input
      volumes:
      - name: input
        emptyDir: {}
~~~


## References
- https://kubernetes.io/docs/concepts/workloads/controllers/job/
- https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/