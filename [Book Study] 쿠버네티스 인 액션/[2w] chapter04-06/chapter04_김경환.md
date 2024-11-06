## 파드의 안정적 유지 방법

주 프로세스가 종료시 Kubelet에 의해 재시작 됨. JVM은 별도의 프로브 설정을 통해 재시작 필요.

---

### 라이브니스 프로브 종류 및 설정 방법

- 각 Probe의 주요 설정 옵션
    - **`initialDelaySeconds`**: 처음 probe를 실행하기 전 대기 시간
    - **`periodSeconds`**: probe를 주기적으로 실행하는 간격
    - **`timeoutSeconds`**: 응답을 기다리는 시간
    - **`failureThreshold`**: 연속적인 실패 횟수 (이 횟수를 넘기면 컨테이너가 재시작됨)

- **HTTP GET Probe**
    - HTTP GET 요청을 특정 엔드포인트로 보내서 응답 상태를 확인합니다.
    - 요청에 대한 응답 코드가 200에서 399 사이일 경우 정상으로 간주하고, 그 외의 응답 코드는 실패로 간주합니다.
    - 주로 HTTP API를 가진 애플리케이션에서 사용됩니다.
        
        ```yaml
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        ```
        

- **TCP Socket Probe**
    - TCP 소켓에 연결을 시도하여, 특정 포트가 열려 있는지 확인합니다.
    - 연결에 성공하면 정상으로 간주하고, 실패할 경우 비정상으로 간주합니다.
    - TCP 연결 확인이 가능한 서버에 유용합니다.
        
        ```yaml
        livenessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 5
          periodSeconds: 10
        ```
        

- **Exec Probe**
    - 파드 내에서 특정 커맨드를 실행하고, 그 결과로 나오는 종료 코드를 확인합니다.
    - 종료 코드가 0일 경우 정상으로 간주하고, 0이 아닐 경우 비정상으로 간주합니다.
    - 복잡한 논리를 수행해야 하거나 특정 파일 또는 디렉터리 상태를 확인할 때 사용됩니다.
        
        ```yaml
        livenessProbe:
          exec:
            command:
              - cat
              - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 10
        ```
        

- 제기동 횟수 확인 커멘드

```bash
kubectl get pods -n <namespace> -o wide

NAME         READY   STATUS    RESTARTS   AGE
my-pod       1/1     Running   3          10m
```

- 제기동 사유 확인 커멘드

```bash
kubectl describe pod <pod-name> -n <namespace>

Containers:
  my-container:
    Container ID:   docker://abc123
    Image:          my-image
    Image ID:       docker-pullable://my-image@sha256:12345
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 02 Nov 2024 10:00:00 UTC
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1 // 이전 종료 사유 137 강제 종료 (SIGKILL) 143(SIGTERM)
      Started:      Mon, 02 Nov 2024 09:59:00 UTC
      Finished:     Mon, 02 Nov 2024 10:00:00 UTC
    Ready:          True
    Restart Count:  3

```

- 프로브를 항상 가볍게 유지한다.
    - 너무 많은 연산 리소스를 사용하지 않아야함. 1초이내 완료
    - 스프링 부트 actuator를 활성하고 /health 이용.
    - 프로브가 요청하는 로직에 재시도 루프를 구현하지 않아야함. 프로브 설정을 통해 하도록 함

---

### 레플리케이션컨트롤러

- **파드 복제본 수 유지**:
    - 지정된 수의 파드 복제본이 항상 유지되도록 합니다. 설정된 파드 수보다 적으면 새 파드를 생성하고, 많으면 삭제합니다.
- **파드 자동 재생성**:
    - 파드가 예기치 않게 삭제되거나 장애가 발생해 종료될 경우 자동으로 파드를 생성하여 복구합니다. 이를 통해 애플리케이션의 가용성을 높일 수 있습니다.
- **스케일링**:
    - 사용자가 필요에 따라 복제본 수를 변경할 수 있습니다. 이를 통해 애플리케이션을 수동으로 스케일링할 수 있으며, 부하에 따라 수를 조정할 수 있습니다.
- **셀렉터 기반으로 파드 관리**:
    - ReplicationController는 특정 라벨 셀렉터를 기반으로 파드를 관리합니다. 셀렉터에 맞는 파드를 확인하여 제어 대상 파드와 일치하지 않는 파드는 관리하지 않습니다.

<img width="735" alt="image" src="https://github.com/user-attachments/assets/8b9e4787-debb-4e69-899c-3153ef628f77">

- 레플리케이션 컨트롤러 정의

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: my-app-image
        ports:
        - containerPort: 8080
```

- 상태 확인

<img width="759" alt="image" src="https://github.com/user-attachments/assets/39a9b432-b30c-4e19-8e65-987d01c2d431">

- **노드 장애시 가능한 노드로 이동 되어 배포됨. 가용 노드가 없는경우에는 Pending 상태로 남아 있음.**
- 레이블을 변경을 통해 관리되는 파드 또는 비관리 파드로 변경가능
    - 파드를 제거하는 실제 사례
        - 시스템 장애시 디버깅 용도로
- 리플리케이션컨트롤러 레이블 변경시
    - 레이블 셀랙터가 같지 않다면 의도한 파드가 새로 생성
    - 같다면 의도한 파드 숫자만큼 만듬

<img width="900" alt="image" src="https://github.com/user-attachments/assets/dfd4bea9-3497-41b1-8025-5550d0f61509">

- 레플리케이션컨트롤러의 피드 템플릿을 변경하면 변경 이후에 생성된 파드만 영향을 미치며 기존 파드는 영향을 받지 않는다.
- 스케일 업 방법 커멘드 ( 에디터 방식으로 가능 )

```yaml
//up
kubectl scale rc kubia --replicas=10

//down
kubectl scale rc kubia --replicas=10

```

- 레플리케이션컨트롤러 삭제시 파드도 삭제됨
    - —cascade=false 관리되지 않는 파드로 만드는건 가능
    
    ```yaml
    kubectl delete rc kubia --ca5cade=Fal5e
    ```
    

### ReplicationController와 ReplicaSet의 차이점

- **ReplicaSet**은 ReplicationController의 개선된 버전으로, 동일한 기능을 제공하면서도 더 강력한 라벨 셀렉터와 Deployment와의 호환성을 가지고 있습니다.
- 최신 Kubernetes에서는 보통 **ReplicaSet**과 **Deployment**를 사용하는 것을 권장하며, ReplicationController는 구형 리소스입니다.

### matchExpressions

- **`In`**: 특정 값들 중 하나와 일치하는 경우
- **`NotIn`**: 특정 값들과 일치하지 않는 경우
- **`Exists`**: 특정 키가 존재하는 경우
- **`DoesNotExist`**: 특정 키가 존재하지 않는 경우

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: advanced-replicaset
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app
        operator: In
        values: ["my-app", "your-app"]
      - key: environment
        operator: NotIn
        values: ["dev"]
      - key: tier
        operator: Exists
  template:
    metadata:
      labels:
        app: my-app
        environment: prod
        tier: frontend
    spec:
      containers:
      - name: my-container
        image: nginx

```

---

### DaemonSet의 주요 기능

1. **모든 노드에 파드 배포**:
    - DaemonSet은 클러스터의 각 노드에 지정된 파드가 하나씩 배포되도록 보장합니다.
    - 예를 들어, 클러스터 내의 모든 노드에서 로그 수집, 모니터링, 또는 시스템 에이전트 등의 작업을 수행하는 파드를 실행할 때 유용합니다.
2. **노드 추가 및 삭제 시 자동 조정**:
    - 새로운 노드가 클러스터에 추가되면, DaemonSet은 자동으로 해당 노드에 파드를 생성합니다.
    - 반대로, 노드가 삭제되거나 DaemonSet이 관리하는 파드가 중지되면, 이를 자동으로 조정하여 모든 노드에서 파드가 일관되게 배포되도록 유지합니다.
3. **노드 셀렉터를 통한 특정 노드 대상 설정**:
    - 특정 라벨을 가진 노드에만 DaemonSet 파드를 배포할 수도 있습니다. 이를 통해 모든 노드가 아닌 특정 용도의 노드에만 DaemonSet을 적용할 수 있습니다.
4. **시스템 파드 배포에 적합**:
    - DaemonSet은 주로 시스템 수준의 작업을 수행하는 파드에 적합합니다. 예를 들어, 로그 수집기, 모니터링 에이전트, 네트워크 플러그인, kube-proxy 등과 같은 작업에 사용됩니다.

### DaemonSet 관리

- **롤링 업데이트**: DaemonSet은 Kubernetes의 롤링 업데이트 기능을 지원하여, 점진적으로 모든 노드의 파드를 업데이트할 수 있습니다.
- **스케일 조정 불가**: DaemonSet은 기본적으로 각 노드에 하나씩 파드를 배포하도록 설계되어 있어, ReplicaSet처럼 복제본 수를 조정할 수 없습니다.

데몬셋 생성

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:latest
        resources:
          limits:
            memory: "200Mi"
            cpu: "0.5"

```

데몬셋 배포 예시

<img width="705" alt="image" src="https://github.com/user-attachments/assets/cb7ea9a1-82a7-45fc-be2d-4f7e2f5557a7">

노드 셀럭터를 통한 데몬셋 배포

<img width="709" alt="image" src="https://github.com/user-attachments/assets/cfb7091f-feb7-4c46-8e3b-26d65ab78bca">

---

### Job 리소스의 주요 특징

1. **작업 완료 보장**:
    - Job은 특정 작업이 성공적으로 완료될 때까지 파드를 관리합니다. 작업이 완료되면 Job 리소스는 파드를 종료하고 완료 상태를 유지합니다.
    - 실패한 작업은 자동으로 다시 시도하며, 설정에 따라 일정 횟수 이상 실패 시 Job이 종료될 수도 있습니다.
2. **재시도 정책**:
    - Job이 실패할 경우, Kubernetes는 설정된 재시도 정책에 따라 작업을 자동으로 재시도합니다.
    - `backoffLimit` 필드를 통해 최대 재시도 횟수를 설정할 수 있으며, 이 값을 초과할 경우 Job은 실패 상태로 종료됩니다.
3. **병렬 처리 지원**:
    - Job은 병렬 처리를 지원하며, `parallelism` 필드를 통해 동시에 실행할 파드의 개수를 설정할 수 있습니다.
    - 이를 통해 대규모 데이터 처리 작업이나 대량의 작업을 병렬로 수행할 수 있습니다.
4. **완료 후 파드 자동 정리**:
    - 기본적으로 Job이 완료되면 Kubernetes는 파드를 자동으로 정리하여 리소스를 확보합니다.
    - `ttlSecondsAfterFinished` 필드를 사용하여 Job이 완료된 후 일정 시간이 지나면 자동으로 Job 리소스를 삭제할 수도 있습니다.

### Job의 활용 사례

- **데이터 처리**: 데이터베이스 백업, ETL 작업 등과 같은 데이터 처리 작업에 활용
- **배치 작업**: 일회성으로 실행해야 하는 배치 작업
- **알림 및 이메일 전송**: 특정 이벤트가 발생했을 때 알림 또는 이메일을 전송하는 작업
- **애드혹 스크립트 실행**: 필요할 때만 실행해야 하는 스크립트나 유지 보수 작업 수행

잡 리소스 생성

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  parallelism: 3
  completions: 5
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo Processing..."]
      restartPolicy: Never

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "0 3 * * *"  # 매일 오전 3시에 실행
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo", "Hello, World!"]
          restartPolicy: Never
  successfulJobsHistoryLimit: 3  # 성공한 Job 기록 3개 유지
  failedJobsHistoryLimit: 1      # 실패한 Job 기록 1개 유지
```
