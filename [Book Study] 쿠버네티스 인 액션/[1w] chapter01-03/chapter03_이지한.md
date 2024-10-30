# 파드: 쿠버네티스에서 컨테이너 실행

## 파드소개

### 파드
* 함께 배치된 **컨테이너 그룹**
* 쿠버네티스의 기본 빌딩 블록
* 일반적으로 파드는 하나의 컨테이너를 포함 (그룹이니까 여러개가 될 수 있음)
* 파드 안에 있는 모든 컨테이너는 **같은 노드에서 실행**
    
<img width="506" alt="image" src="https://github.com/user-attachments/assets/95648343-85eb-4ef5-94f3-5069f53b529a">   
    

### 파드가 필요한 이유
* 여러 프로세스를 실행하는 단일 컨테이너보다 다중 컨테이너가 나은 이유
  * 모든 프로세스 실행과 로그 관리를 사용자가 직접 처리해야함
  * 각 프로세스의 실패 시 자동 재시작 같은 메커니즘이 필요
  * 여러 프로세스가 동일한 표준 출력으로 로그를 기록하므로 어떤 로그가 어떤 프로세스에서 발생했는지 구분하기 어려움

### 파드 이해하기

#### 네임스페이스 공유
  * 동일한 네트워크 네임스페이스와 UTS(Unix Time-Sharing System) 네임스페이스를 공유
    * 같은 IP 주소 포트 공간을 공유
      * 동일한 파드 안 컨테이너에서 실행중인 프로세스틑 포트는 달라야함!
    * 동일한 호스트 이름을 공유
  * IPC(Inter-Process Communication) 네임스페이스를 통해 서로 통신가능 (메시지 큐, 메모리 공유...)
    <img width="801" alt="image" src="https://github.com/user-attachments/assets/d72e19f0-746b-4cdf-9a4a-c1b0856535fc">
  * 동일한 PID 네임스페이스도 공유할 수 있지만 기본 설정은 아님
#### 파일 시스템 분리
  * 컨테이너는 고유의 파일 시스템을 가지지만 k8s 볼륨을 활용해 파일 디렉터리를 공유 가능

#### 파드 간 플랫 네트워크
  * 클러스터 내의 모든 파드는 하나의 플랫한 공유 네트워크 주소 공간에 상주
    * 모든 파드는 다른 파드의 IP 주소를 사용해 접근 가능
  * NAT 없이 통신
    * 파드 간 통신에는 NAT가 필요하지 않으며, 파드가 서로 다른 워커 노드에 있더라도 NAT 없이 플랫 네트워크에서 직접 통신 가능.
     
<img width="491" alt="image" src="https://github.com/user-attachments/assets/9527b8c8-255b-4467-b4da-848c6300c35d">    
    


### 파드에서 컨테이너의 적절한 구성

#### 다계층 애플리케이션을 여러 파드로 분할
* 프론트 서버, 백엔드 서버 컨테이너 두 개가 있다면 파드를 분리하도록!
  * 반드시 같은 머신에서 실행되어야 하나? -> 아니.
  * 노드가 2개라고 가정하고, 같은 파드에 배포한다면 1개의 노드만 사용할 것임

#### 개별 확장이 가능하도록 여러 파드로 분할
* 프론트 서버만 스케일아웃 해야하는데 같은 파드에 묶으면 쓸데없이 백엔드 서버도 확장됨

#### 파드에서 여러 컨테이너는 언제 사용?
* 주요 프로세스와 하나 이상의 보완 프로세스로 구성된 경우
  * 주 컨테이너 : 파일 제공하는 웹 서버
  * 사이드카 컨테이너 : 외부 소스에서 주기적으로 데이터 받아 웹 서버에 디렉토리에 저장    

#### 여러 컨테이너를 묶어 사용할 때 참고

* 컨테이너를 함께 실행해야함? 다른 호스트에서 실행 가능?
* 여러 컨테이너가 모여 하나의 구성요소가 되는가? 개별 구성요소인가?
* 컨테이너가 함께 또는 개별로 스케일링돼야 하나?

<img width="469" alt="image" src="https://github.com/user-attachments/assets/2fdbdc7f-955e-4bfa-b529-9ab51584eb16">   
   


### YAML / JSON 디스크립터로 파드 생성
#### 기존 파드의 YAML 디스크립터

* Metadata: 이름, 네임스페이스, 레이블 및 파드에 관한 기타 정보
* Spec: 파드 컨테이너, 볼륨, 기타 데이터 등 파드 자체의 명세
* Status: 파드 상태, 각 컨테이너 설명과 상태, 파드 내부 IP 등 현재 실행중인 파드 관련 상태

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual  // 파드 이름
spec:
  containers:
  - image: ozleejihan/kubia     // 파드 이미지
    name: kubia                 // 컨테이너 이름
    ports:
    - containerPort: 8080        // 애플리케이션 수신 포트
      protocol: TCP
```

> kubectl explain pods로 오브젝트 필드 찾을 수 있음

* kubectl create 명령으로 파드 만들기

```shell
$ kubectl create -f kubia-manual.yaml
```
* 실행중인 파드의 전체 정의 가져오기
```shell
$ kubectl get po kubia-manual -o yaml
```

* 애플리케이션 로그 보기   
```shell
$ kubectl logs kubia-manual
```

* 로컬 네트워크 포트를 파드의 포트로 포워딩
```shell
$ k port-forward kubia-manual 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
```

```shell
$ curl localhost:8888
You've hit kubia-manual
```   

#### 레이블을 이용한 파드 구성

<img width="544" alt="image" src="https://github.com/user-attachments/assets/925d5c31-940e-479d-90d0-b7e9bb55068f">   


여러 개 레플리카를 실행하는 여러 마이크로서비스에 속해 있는 파드와 동일한 마이크로서비스의 다른 릴리즈에 속한 파드덜..    
어떤 파드가 어떤 것인지 알기 쉽게 하기 위해 **레이블** 을 통해 **오브젝트를 조직화** 함

* 레이블 소개
  * 리소스에 첨부하는 키-값 쌍
  * 레이블 셀럭터를 사용해 리소스를 선택할 때 활용됨

<img width="585" alt="image" src="https://github.com/user-attachments/assets/bf60ed71-ff59-4825-81fc-d7b2f445734c">

    
두 레이블을 추가해서 위의 난잡한 파드를 2차원으로 구성함

* app: 파드가 속한 애플리케이션
* rel: 파드의 릴리즈

##### 파드를 생성할 때 레이블 지정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: kubia-manual
    env: prod
spec:
  containers:
  - image: ozleejihan/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

* 모든 레이블 보기 

```shell
$ k get po --show-labels
NAME              READY   STATUS    RESTARTS   AGE   LABELS
kubia-manual      1/1     Running   0          24m   <none>
kubia-manual-v2   1/1     Running   0          27s   creation_method=kubia-manual,env=prod
```

* 특정 레이블만 보기
```shell
$ k get po -L creation_method,env 
NAME              READY   STATUS    RESTARTS   AGE   CREATION_METHOD   ENV
kubia-manual      1/1     Running   0          25m                     
kubia-manual-v2   1/1     Running   0          95s   kubia-manual      prod
```
   
#### 기존 파드 레이블 수정
```shell
$ k label po kubia-manual creation_method=manual
pod/kubia-manual labeled
```

```shell
$ k label po kubia-manual-v2 env=debug --overwrite
pod/kubia-manual-v2 labeled
```

```shell
$ k get po -L creation_method,env
NAME              READY   STATUS    RESTARTS   AGE     CREATION_METHOD   ENV
kubia-manual      1/1     Running   0          31m     manual
kubia-manual-v2   1/1     Running   0          7m46s   kubia-manual      debug
```


### 레이블 셀렉터를 이용한 파드 부분 집합 나열

#### 레이블 셀렉터
특정 레이블로 태그된 파드의 부분 집합을 선태해 원하는 작업을 수행

#### 레이블 셀렉터를 사용해 파드 나열
```shell
$ k get po -l creation_method=manual
NAME           READY   STATUS    RESTARTS   AGE
kubia-manual   1/1     Running   0          37m
```

```shell
$ k get po -l env
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual-v2   1/1     Running   0          13m
```

```shell
$ k get po -l '!env'
NAME           READY   STATUS    RESTARTS   AGE
kubia-manual   1/1     Running   0          37m
```

#### 레이블 셀럭터 여러 조건 사용
```shell
$ k get po -l creation_method=kubia-manual,env=debug
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual-v2   1/1     Running   0          16m
```

### 레이블 셀렉터를 이용해 파드 스케쥴링 제한

특정한 조건으로 인해 파드를 특정 노드에 스케줄링 해야할 상황도 있음 이럴 때 레이블을 사용함

#### 워커 노드 분류에 레이블 사용

파드 뿐만 아니라 노드 등 모든 쿠버네티스 오브젝트에 레이블을 부착할 수 있음

```shell
$ k label node minikube gpu=true
node/minikube labeled
```

```shell
$ k get nodes -L gpu
NAME       STATUS   ROLES           AGE   VERSION   GPU
minikube   Ready    control-plane   18h   v1.31.0   true
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:   // gpu=true인 노드에 이 파드를 배포해라!
    gpu: "true"
  containers:
  - image: ozleejihan/kubia
    name: kubia
```

> 이전회사에서 airflow job을 pod로 배포하는데, 이 때 medium-node, haevy-node로 구분하여 해당 노드에 배포되도록 작업

### 파드에 어노테이션 달기

* 어노테이션은 레이블과 비슷하지만 식별정보를 갖지 않음
* 파드나 다른 API 오브젝트에 설명을 추가해둘 때 사용됨

#### 어노테이션 추가 및 수정

```shell
$ k annotate pod kubia-manual jihan.com/annotation="foo bar"
pod/kubia-manual annotated
```

```shell
$ k describe pod kubia-manual
Name:             kubia-manual
Labels:           creation_method=manual
...
Annotations:      jihan.com/annotation: foo bar
```

### 네임스페이스를 사용한 리소스 그룹화

레이블을 사용했을 때 오브젝트 그룹이 서로 겹칠 수 있음
한 번에 하나의 그룹 안에서만 작업하고 싶을 때, 오브젝트를 **네임스페이스로 그룹화** 함

#### 다른 네임스페이스와 파드 살펴보기

```shell
$ k get ns
NAME                   STATUS   AGE
default                Active   19h
kube-node-lease        Active   19h
kube-public            Active   19h
kube-system            Active   19h
kubernetes-dashboard   Active   18h
```
기본은 default 네임스페이스에 속해있는 오브젝트만 표시한다.

* 특정 네임스페이스의 파드 조회

```shell
$ k get po -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-6f6b679f8f-lrwxf           1/1     Running   0          19h
etcd-minikube                      1/1     Running   0          19h
kube-apiserver-minikube            1/1     Running   0          19h
kube-controller-manager-minikube   1/1     Running   0          19h
kube-proxy-r4z7r                   1/1     Running   0          19h
kube-scheduler-minikube            1/1     Running   0          19h
storage-provisioner                1/1     Running   0          19h
```

네임스페이스는 리소스 격리 외에도 특정 사용자가 지정된 리소스에 접근할 수 있도록 허용
개별 사용자가 사용할 수 있는 컴퓨팅 리소스 제한하는 데에도 사용됨

#### 네임스페이스 생성

##### yaml 파일로 생성

```shell
$ k create -f custom-namespace.yaml
namespace/oz-namespace created
```

##### kubectl create namespace 명령으로 생성

```shell
$ k create namespace jihan-namespace
namespace/jihan-namespace created
```

#### 다른 네임스페이스의 오브젝트 관리

```shell
$ k create -f kubia-manual.yaml -n oz-namespace
pod/kubia-manual created
```

```shell
$ k get po -n oz-namespace
NAME           READY   STATUS    RESTARTS   AGE
kubia-manual   1/1     Running   0          119s
```

#### 네임스페이스가 제공하는 격리 이해

네임스페이스가 다르면 서로 격리돼 통신할 수 없다?
-> 반드시 그렇지 않음. 네트워킹 솔루션에 따라 다르지만 서로 IP를 알고있는 상태면 통신 가능

### 파드 중지와 제거

#### 이름으로 파드 삭제
```shell
$ k delete po kubia-manual
pod "kubia-manual" deleted
```

#### 레이블 셀렉터로 파드 삭제
```shell
$ k delete po -l env=debug
pod "kubia-manual-v2" deleted
```
<img width="593" alt="image" src="https://github.com/user-attachments/assets/c450d9cf-2e2a-4400-94f2-ec794a92fb9f">

#### 네임스페이스 삭제로 파드 제거

```shell
$ k delete ns oz-namespace
namespace "oz-namespace" deleted
```

#### 네임스페이스는 유지하면서 안에 있는 모든 파드 삭제
```shell
$ k delete po --all
pod "kubia-btfb5" deleted
pod "kubia-bz5rp" deleted
```

#### 네임스페이스에서 (거의) 모든 리소스 삭제
```shell
$ k delete all --all
pod "kubia-8p5c6" deleted
pod "kubia-fwps5" deleted
pod "kubia-s27m8" deleted
replicationcontroller "kubia" deleted
service "kubernetes" deleted
service "kubia-http" deleted
```

```shell
$ k get pods
No resources found in default namespace.
```

파드의 파일 시스템은 노드의 파일 시스템과 격리됨
물리적 용량 자체는 노드에서 따옴

- 파드의 볼륨 종류는 ephemeral([에퍼머럴]), persistent volume이 있으며 ephemeral은 파드가 죽으면 날라가는 일시적인 것이고 이것이 디폴트임. persisten volume으로 설정 시 파드가 죽어도 볼륨이 유지됨
<img width="972" alt="image" src="https://github.com/user-attachments/assets/0f98f2a6-2ad8-490e-a22e-e7d7aa0753be">


## Ref.
* 마르코 룩샤 저/강인호, 황주필, 이원기, 임찬식 역 (2020). Kubernetes IN ACTION
