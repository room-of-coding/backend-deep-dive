# ch08. 애플리케이션에서 파드 메타데이터와 그 외의 리소스 액세스

## Downward API로 메타데이터 전달

* Pod 내부에서 Kubernetes 리소스의 정보를 접근할 수 있게 해주는 기능
* 환경변수 또는 파일에 파드의 스펙 또는 상태값이 채워지도록 하는 방식 (REST API 같은 엔드포인트가 아님)
* 메타데이터
  - 파드 이름
  - 파드 IP
  - 파드가 속한 네임스페이스
  - 파드가 실행 중인 노드 이름
  - 파드가 실행 중인 서비스 어카운트 이름
  - 각 컨테이너의 CPU와 메모리 request/limit
  - 파드의 레이블, 어노테이션 (볼륨으로만 노출 가능)
<img width="562" alt="image" src="https://github.com/user-attachments/assets/7e48c233-d5c2-46d9-bf23-2bc5541b33d2">

### 환경변수로 메타데이터 전달
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 4Mi
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    ...
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:    // resourceFieldRef 사용
          resource: requests.cpu
          divisor: 1m        //  리소스 필드의 경우 필요한 단위의 값을 얻으려면 divisor을 정의
```

```shell
$ k exec downward -- env          
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=downward
CONTAINER_MEMORY_LIMIT_KIBIBYTES=4096
POD_NAME=downward
POD_NAMESPACE=default
POD_IP=10.88.1.12
NODE_NAME=gke-kubia-default-pool-a9ec92c1-pnv0
...
```

### 파일로 메타데이터 전달

* downwardAPI 볼륨을 정의해 컨테이너에 마운트
* 레이블과 어노테이션은 파드 실행중 수정되면 쿠버네티스가 이 값을 가지고 있는 파일도 업테이트 함
  * 환경변수로 노출될 경우 값이 변경되어도 수정할 방법이 없음

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    ...
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      ...
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```

```shell
$ k exec downward -- ls -lL /etc/downward         
total 24
-rw-r--r--    1 root     root          1343 Nov 10 12:23 annotations
-rw-r--r--    1 root     root             2 Nov 10 12:23 containerCpuRequestMilliCores
-rw-r--r--    1 root     root             7 Nov 10 12:23 containerMemoryLimitBytes
-rw-r--r--    1 root     root             9 Nov 10 12:23 labels
-rw-r--r--    1 root     root             8 Nov 10 12:23 podName
-rw-r--r--    1 root     root             7 Nov 10 12:23 podNamespace
```

```shell
$ k label pod downward foo=bor --overwrite
pod/downward labeled
$ k exec downward -- cat /etc/downward/labels     
foo="bor"
```
   
> Downward API의 사용법은 단순하나 사용 가능한 메타데이터가 상당히 제한적임

   
## 쿠버네티스 API 서버와 통신

* 애플리케이션에서 클러스터에 정의된 다른 파드나 리소스에 관한 더 많은 정보가 필요할 수 있음
<img width="465" alt="image" src="https://github.com/user-attachments/assets/62da928e-22aa-4e04-a2a0-268c1c998b2d">

### kubectl 프록시로 API 서버 액세스

* 프록시 서버를 실행해 로컬에서 HTTP 연결을 수신하고, 인증을 관리하여 API 서버로 전달함

```shell
$ k proxy
Starting to serve on 127.0.0.1:8001
```
   
```shell
$ curl localhost:8001
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/apps/v1",
    ...
```

```shell
$ curl localhost:8001/apis/batch/v1/jobs
{
  "kind": "JobList",
  "apiVersion": "batch/v1",
  "metadata": {
    "resourceVersion": "3321063"
  },
  "items": []
}
```

### 파드 내에서 API 서버와 통신

#### 파드 실행
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: main
    image: curlimages/curl
    command: ["sleep", "9999999"]
```

#### 서버의 아이덴티티 검증

```shell
$ k exec -it curl -- sh

$ curl https://kubernetes
curl: (60) SSL peer certificate or SSH remote key was not OK
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the webpage mentioned above.
```

* 인증서 서명
  * 7장의 시크릿에서 등장한 /var/run/secrets/kubernetes.io/serviceaccount/에 마운트 되는 default-token 시크릿 사용 (p.339)
```shell
$ curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}

$ export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

* API 서버로 인증
  * default-token 시크릿의 token 사용
```shell
$ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ curl -H "Authorization: Bearer $TOKEN" https://kubernetes
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    ...
  ]
}
```

> RBAC이 활성화된 쿠버네티스 클러스터를 사용할 경우, 서비스 어카운트가 API 서버에 액세스할 권한이 없기 때문에    
> RBAC을 우회하는 명령어 실행해야 위의 실습 가능 (p. 76)    
> 프로덕션에서는 금지!    

* 동일 네임스페이스의 모든 파드 조회하기
  * default-token 시크릿의 namespace 사용
```shell
$ NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
$ curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "3359769"
  },
  "items": [
    {
      "metadata": {
        "name": "curl",
        "namespace": "default",
...
```

* 정리
<img width="731" alt="image" src="https://github.com/user-attachments/assets/e0bc9d89-66fa-45a3-a4f7-4acaead4c1b1">


#### 앰배서더 컨테이너를 이용한 API 통신 간소화

<img width="475" alt="image" src="https://github.com/user-attachments/assets/c87540a4-74aa-4785-a21d-4224e372a68a">

* 메인 컨테이너 애플리케이션 -> HTTP로 앰배서더 연결 -> 앰배서더 프록시 -> HTTPS로 API 서버 연결

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-ambassador
spec:
  containers:
  - name: main
    image: curlimages/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

* 인증없이 잘 되는지 확인
```shell
$ k exec -it curl-with-ambassador -c main -- sh
$ curl localhost:8001
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    ...
```

<img width="547" alt="image" src="https://github.com/user-attachments/assets/1a4af38b-7dd7-4b34-8e11-6fda0663e975">    
    
* curl -> http -> 프록시 -> https -> api 서버
* 메인 애플리케이션 언어에 상관없이 여러 애플리케이션에서 재사용이 가능
* 단점은 추가 프로세스가 실행되며, 추가 리소스 소비

#### 클라이언트 라이브러리 사용해 API 서버 통신

https://kubernetes.io/docs/reference/using-api/client-libraries/

```shell
$ pip install kubernetes
```

* main.py 예제
```python
from kubernetes import client, config
from time import sleep

# Kubernetes 클라이언트 구성 로드 (kubeconfig 사용)
config.load_kube_config()

# Kubernetes 클라이언트 생성
v1 = client.CoreV1Api()

# 기본(default) 네임스페이스의 모든 Pod 목록 가져오기
print("Listing pods in namespace 'default'")
pods = v1.list_namespaced_pod(namespace="default")
for pod in pods.items:
    print(f"Found pod: {pod.metadata.name}")

# 새로운 Pod 생성
print("Creating a pod")
pod_body = client.V1Pod(
    metadata=client.V1ObjectMeta(name="programmatically-created-pod"),
    spec=client.V1PodSpec(
        containers=[
            client.V1Container(
                name="main",
                image="busybox",
                command=["sleep", "99999"]
            )
        ]
    )
)
created_pod = v1.create_namespaced_pod(namespace="default", body=pod_body)
print(f"Created pod: {created_pod.metadata.name}")

# Pod 편집 (라벨 추가)
print("Editing pod to add label foo=bar")
patch_body = {
    "metadata": {
        "labels": {
            "foo": "bar"
        }
    }
}
v1.patch_namespaced_pod(name="programmatically-created-pod", namespace="default", body=patch_body)
print("Added label foo=bar to pod")

# 1분 대기 후 Pod 삭제
print("Waiting 1 minute before deleting pod...")
sleep(60)
print("Deleting the pod")
v1.delete_namespaced_pod(name="programmatically-created-pod", namespace="default")
print("Deleted the pod")
```

* main.py 실행
```shell
$ python main.py
Listing pods in namespace 'default'
Found pod: kubia-fcbrx
Found pod: kubia-mdhv4
Found pod: kubia-sk8hb
Creating a pod
Created pod: programmatically-created-pod
Editing pod to add label foo=bar
Added label foo=bar to pod
Waiting 1 minute before deleting pod...
Deleting the pod
Deleted the pod
```   
   
* po 조회 결과
```shell
$ k get po
NAME                           READY   STATUS        RESTARTS   AGE
kubia-fcbrx                    1/1     Running       0          11m
kubia-mdhv4                    1/1     Running       0          11m
kubia-sk8hb                    1/1     Running       0          11m
programmatically-created-pod   1/1     Terminating   0          79s
```
