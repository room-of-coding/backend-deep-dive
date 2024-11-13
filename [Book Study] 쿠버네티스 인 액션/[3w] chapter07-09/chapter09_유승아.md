# 디플로이먼트: 선언적 애플리케이션 업데이트

> - 파드를 최신 버전으로 교체
> - 관리되는 파드 업데이트
> - 디플로이먼트 리소스로 파드의 선언적 업데이트
> - 롤링 업데이트 수행
> - 잘못된 버전의 롤아웃 자동 차단
> - 롤아웃 속도 제어
> - 이전 버전으로 파드 되돌리기

## 파드에서 실행중인 애플리케이션 업데이트
모든 파드를 업데이트하는 방법
> - 기존 파드를 모두 삭제한 다음 새 파드를 시작하는 방법
> - 새로운 파드를 시작하고, 기동하면 기존파드를 삭제하는 방법
>   - 새파드를 모두 추가한 다음 한꺼번에 삭제
>   - 순차적으로 파드를 추가하고 기존 파드를 점진적으로 삭제


### 수동으로 오래된 파드를 삭제하고 새파드로 교체하는 방법
![img.png](https://github.com/user-attachments/assets/d1b28fd5-a9da-43f1-bf9e-6d5ad0e5409a)
1. **레플리케이션 컨트롤러 레이블 셀렉터 변경**: 새 버전의 파드를 생성할 수 있도록  레플리케이션 컨트롤러의 레이블 셀렉터를 변경합니다.
2. **기존 파드 삭제**: 기존 파드가 삭제되면 새로운 파드가 생성됩니다.
> 새로운 파드가 기동되는 시간이 필요하기 때문에 다운타임 발생
>

### 수동으로 새 파드를 기동하고, 이전 파드를 삭제하는 방법  
#### 한번에 이전 버전에서, 새버전으로 전환 (블루-그린 디플로이먼트)
![img.png](https://github.com/user-attachments/assets/4c3c2ccb-a513-4b3a-a208-a5541e9043c9)
1. **두 개의 레플리케이션 컨트롤러**: 새로운 버전으로 전환하기 위해 기존 애플리케이션(Blue)은 그대로 두고, 새 버전 애플리케이션(Green)을 레플리케이션 컨트롤러로 실행합니다. 즉,레플리케이션 컨트롤러는 두개가 함께 존재합니다.
2. **서비스 레이블 셀렉터 변경**: 모든 것이 정상이라면 서비스의 레이블 셀렉터를 변경하여 새로운 버전(Green)으로 트래픽을 전환합니다.

> 잠시동안 동시에 두배의 파드가 실행되므로 더 많은 하드웨어 리소스가 필요

## 레플리케이션컨트롤러로 자동 롤링 엡데이트 수행
 > 참고. kubectl rolling-update 명령어는 Kubernetes 1.14부터 더 이상 사용되지 않으며, 이 기능은 Deployment 리소스로 대체 
 
### kubectl rolling-update를 더이상 사용하지 않는 이유
- kubectl은  Kubernetes API 서버와의 실시간 통신을 통해 작업을 처리: 네트워크 문제등으로 통신이 중단되면 작업의 완전성이나 상태 관리가 불확실
- 쿠버네티스는 의도하는 시스템의 상태를 선언하는 것이 핵심 철학 : 의도하는 상태를 명시하고, 쿠버네티스가 실제 시스템을 그 상태로 자동으로 조정하도록 하는 방식이므로 rolling-update 명령은 적절하지 않음  

## 선언적 업데이트를 위한 디플로이먼트
> 디플로이먼트를 생성하면 하단의 레플리카셋 리소스가 아래에 생성됨
> 실제 파드는 디플로이먼트가 아닌 디플로이먼트의 레플리카셋에 의해 관리

- 리플리카셋은 파드의 수와 상태를 유지하는 역할
- 디플로이먼트는 애플리케이션 배포 및 관리의 전반적인 과정을 자동화
  - 선언적으로 업데이트하기 위한 높은 수준의 리소스
  
![img.png](https://github.com/user-attachments/assets/92685bd1-ebb7-4bc8-b574-d63036b917a9)

### 디플로이먼트

#### 디플로이먼트 롤아웃 상태 출력
```bash
❯ kubectl rollout status deployment kubia
deployment "kubia" successfully rolled out
```
#### 디플로이의먼트 리플리카셋
> Deployment를 생성할 때, 템플릿의 구성에 따라 ReplicaSet이 생성
> 이 때 ReplicaSet은 **고유한 해시 값**을 생성
> **고유한 해시값**을 이용하여 ReplicaSet이 관리해야 할 파드를 정확히 파악하고, 설정된 개수만큼의 파드가 항상 유지되도록 관리
```bash
❯ kubectl get replicasets
NAME               DESIRED   CURRENT   READY   AGE
kubia-58cd55b576   3         3         3       44m

❯ kubectl get pods
NAME                     READY   STATUS              RESTARTS   AGE
kubia-58cd55b576-7kftn   1/1     Running             0          82m
kubia-58cd55b576-kjc24   1/1     Running             0          82m
kubia-58cd55b576-pbvjj   1/1     Running             0          82m
```
### 디플로이먼트 업데이트
디플로이먼트 리소스에 정의된 파드 템플릿을 수정하면, 쿠버네티스가 실제 시스템 상태를 리소스에 정의된 상태로 만드는데 모든 단계를 수행합니다.
1. Rolling Update(Default) : 이전 파드를 하나씩 제거하고 동시에 새파드를 추가 (다운타임 X)
2. Recreate : 이전 버전을 모두 삭제하고 새로운 파드를 만듦 (다운타임 O)

![img_1.png](https://github.com/user-attachments/assets/e1b3e1a4-ed0e-4650-8d0a-1ed23300b143)

### 디플로이먼트 롤백

#### 마지막 롤아웃 취소
```bash
kubectl rollout undo deployment <deployeement-name>
```
#### 디플로이먼트 롤아웃 이력표시
```bash
kubectl rollout history deployment <deployeement-name>
```

#### 특정 디플로이먼트 개정으로 롤백
```bash
kubectl rollout undo deployment <deployeement-name> --to-revision=<revision-value>
```
레플리카셋을 수동으로 삭제하는 것에 주의하세요. 
![img_2.png](https://github.com/user-attachments/assets/dfe0e496-1be9-4095-8ec0-f4738de8a58f)

### 롤라웃 속도 제어 
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```
#### maxSurge
 - 새로운 Pod의 최대 수를 지정. 새로 배포될 Pod가 기존 Pod 수를 초과할 수 있는 최대 개수
 - 즉, 새로운 버전의 Pod가 배포되는 동안, 기존 Pod 수를 초과하지 않도록 보장하는 역할
#### maxUnavailable
  - 업데이트 중에 사용할 수 없는 Pod의 최대 수를 지정
  - 즉, 새로운 버전으로 배포되는 동안, 기존 버전의 Pod가 최대로 사용 불가능한 수를 정의
  - 예시: 1인 경우, 새 버전의 Pod가 배포될 때마다, 기존 Pod 중 한 개는 종료되지만, 그 이상의 Pod는 동시에 종료되지 않도록 보장

#### 롤아웃 일시 정지
```bash
kubectl rollout pause deployement <deployement-name>
```

#### 롤아웃 재개
```bash
kubectl rollout resume deployement <deployement-name>
```

### 잘못된 버전의 롤아웃 방지
- minReadySeconds와 레디니스 프로브를 통해 잘못된 버전의 롤아웃을 방지할 수 있음
- minReadySeconds 동안 레디니스 프로브가 성공적으로 동작해야만 해당 파드를 준비 상태로 인정
- 이 시간 내에 레디니스 프로브가 실패하면, Kubernetes는 해당 버전의 롤아웃을 차단
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
        - image: luksa/kubia:v3
          name: nodejs
          readinessProbe:
            periodSeconds: 1
            httpGet:
              path: /
              port: 8080
```

### 롤아웃 데드라인
- `progressDeadlineSeconds` 속성을 이용하여 롤아웃 데드라인 설정 가능
- 해당 시간 동안 롤아웃이 진행되지않으면 실패한 것으로 간주
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  progressDeadlineSeconds: 600  # 배포가 10분 내에 완료되지 않으면 실패로 간주
  ...
```