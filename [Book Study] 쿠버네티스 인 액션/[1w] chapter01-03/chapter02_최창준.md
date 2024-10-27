## 2.1 도커를 사용한 컨테이너 이미지 생성, 실행, 공유하기

### 2.1.1 도커 설치와 Hello World 컨테이너 실행하기

**리눅스 머신**

도커는 리눅스 머신에 설치되고 실행된다. 맥이나 윈도우를 사용하는 환경이라면 도커를 안내에 따라 설치하는 것은 가상 머신을 생성하고 그 가상 머신 안에 도커 데몬을 구동하게 되는 것이다. 호스트 OS에서 도커 클라이언트 파일을 실행하면 가상 머신에 구동된 도커 데몬과 통신하는 방식이다.

**busybox**

busybox 이미지는 echo, ls, gzip 등과 같은 표준 UNIX 명령줄 도구들을 합쳐놓은 단일 실행 파일이다.

![image](https://github.com/user-attachments/assets/d8548e35-af44-4452-84ad-6f5e0fd8b50d)


**호스트 이름** ⭐

컨테이너는 내부의 호스트 이름을 갖는다. 호스트 머신에서 실행되지만 호스트 머신의 이름이 아닌 각자의 호스트 이름을 갖게 되는 것이다. 나중에 쿠버네티스에서 애플리케이션들을 컨테이너 형태로 배포하고, 스케일 아웃 등을 할 때 이런 호스트 이름이 유용하게 사용된다.

**이미지 빌드 원리**

도커 클라이언트가 디렉터리의 컨텐츠를 데몬에 업로드한다. 리눅스가 아닌 OS에서는 도커 클라이언트는 호스트 OS에 위치하고 데몬은 가상머신 내부에서 실행된다.

![image](https://github.com/user-attachments/assets/aac479a1-801a-4d57-a594-bfd9f3e6d3a1)


쿠버네티스 기초 다지기 책에서 발췌한 도커 이미지 레이어 생성 방식. 레이어들을 기반으로 쌓이는 구조이고, 쓰기 가능한 레이어가 마치 케이크의 프로스팅처럼 위에 쌓인다고 기억하면 기억이 잘 된다.

![image](https://github.com/user-attachments/assets/19e4f216-f21b-4c33-9ef5-506e22320eb9)


## 2.2 설치

쿠버네티스 클라이언트, kubectl 설치 내용은 생략

## 2.3 쿠버네티스에서 애플리케이션 실행

### 여러가지 쿠버네티스 명령어

**별칭 설정**

bash shell 이나 zshell에 아래 내용을 추가하면 매번 명령어를 길게 입력할 필요가 없다.

```jsx
alias k=kubectl

$ sudo vi ~/.zshrc  # zshell 내용에 추가
$ sudo vi ~/.bashrc # bash shell 내용에 추가
```

- 클러스터 정보 표시

> $ kubectl cluster-info
> 

- 클러스터 노드 조회

> $ kubectl get nodes
> 

- 오브젝트 세부 정보 조회(describe까지)

> $ kubectl describe node {노드 이름}
> 

- docker와 비슷하게 kubectl로 바로 이미지 실행 가능

> $ kubectl run {pod의 이름 설정} —image={이미지명} —port={포트번호} —generator={레플리케이션 컨트롤러를 사용하는 경우 입력}
> 

**파드**

- 쿠버네티스는 개별 컨테이너들을 직접 다루지 않음
- 같은 워커 노드에서 같은 리눅스 네임스페이스로 실행되는 컨테이너 그룹
- 자체 IP, 호스트 이름, 프로세스 등이 있음

![image](https://github.com/user-attachments/assets/fed0d727-e18c-4040-a5ee-0208107d26a8)


**아래 명령어를 통해 pod 정보 조회 가능**

> $ kubectl get pods
> 

**백그라운드 동작 원리**

1. 쿠버네티스 API 서버로 kubectl 명령 전달
2. 클러스터에 새로운 레플리케이션 컨트롤러(RC) 오브젝트를 생성
    1. rc는 마스터 노드의 etcd(상태 저장소)에 클러스터 리소스로 저장됨
3. rc가 새 파드를 생성하고, 스케줄러가 이를 워커 노드 중 하나에 스케줄링(할당)함
4. 해당 워커 노드의 kubelet은 스케줄링 신호를 받고, 이미지가 로컬에 없는 경우 레지스트리에서 풀
5. 도커가 이미지를 통해 컨테이너 실행

![image](https://github.com/user-attachments/assets/5f8cbdd4-9217-4da9-b254-eaf121525c31)


### 2.3.2 웹 애플리케이션에 접근하기

외부에서 파드에 접근할려면 서비스 오브젝트를 만들고 노출해야함. LoadBalancer 유형의 서비스를 생성 시 로드밸런서의 퍼블릭 IP를 통해 내부 파드에 연결 가능

> $ kubectl expose rc {rc 이름} —type=LoadBalancer —name {로드밸런서로 지정할 이름}
> 

서비스 조회

> $ kubectl get svc(services)
> 

![image](https://github.com/user-attachments/assets/52a4cf5b-f241-4ef2-b6f0-cd28045028f7)


external ip와 port 정보를 통해 curl 요청을 보낼 수 있음

### 2.3.3 시스템의 논리적인 부분

사용자가 pod를 직접 생성하는게 아니고 rc가 pod를 생성. rc가 파드를 복제하고 항상 실행 상태로 만드는 주체이다. 만약 파드가 사라졌다면 rc는 사라진 파드를 대체하기 위해 새로운 파드를 생성함

![image](https://github.com/user-attachments/assets/6bee3ab8-94ba-4d56-ad3a-da2375696fe9)



**서비스 필요성**

- 파드는 고유한 사설 IP와 호스트 이름을 가짐
- 파드는 언제든지 사라질 수 있고, rc에 의해 다른 파드로 교체될 수 있음
- 교체된 파드의 IP 주소 문제, 그리고 여러 개의 파드를 단일 IP로 묶되 여러 포트로 노출시키는 문제를 해결하기 위해 서비스가 필요
- 서비스는 정적 IP를 갖고 변경되지 않음

### 2.3.4 애플리케이션 수평 확장

**파드 수 확인**

DESIRED, CURRENT로 표기됨

> kubectl get replicationcontrollers
> 

![image](https://github.com/user-attachments/assets/6278cb8a-1808-4198-b4a4-4d89cb832a70)


레플리카 수 늘리기

> kubectl scale rc kubia —replicas=3
> 

→ 쿠버네티스의 기본 원칙: 쿠버네티스에게 ‘액션'을 알려주는게 아니라 ‘원하는 상태(desired state)’를 알려줌

확장 후 여러 번 curl 요청 시, 서비스가 다수의 파드 앞에서 로드밸런서 역할을 하므로 무작위로 요청이 전달됨

### 2.3.5 애플리케이션이 실행 중인 노드 검사하기

- 쿠버네티스에서는 파드가 어떤 노드에서 실행 중인지는 중요하지 않음
    - kubectl get pods 명령어에서 노드에 대한 정보가 없는 것은 이 때문임
    - -o wide 옵션을 추가하면 파드 IP와 실행 중인 노드도 표시됨
    - kubectl describe pod {파드 이름}으로 더 상세한 정보 확인 가능
- 파드의 노드 위치 여부와 상관없이 파드끼리 통신 가능

### 2.3.6 쿠버네티스 대시보드 소개

GKE, Minikube에서의 dashboard를 소개함. 경환님이 알려준 k9s도 있음

![image](https://github.com/user-attachments/assets/94ad0c67-2e1a-4c3e-840f-7abb38584205)


