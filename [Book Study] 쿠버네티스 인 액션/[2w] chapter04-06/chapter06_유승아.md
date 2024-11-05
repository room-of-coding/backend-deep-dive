# 볼륨: 컨테이너에 디스크 스토리지 연결

- 컨테이너는 독립적인 파일 시스템을 가지므로, 컨테이너가 재시작되면 해당 컨테이너의 파일 시스템은 사라지고 새로 생성
- 동일한 파드안에 있는 컨테이너는 볼륨을 사용하여 파일을 공유할 수 있음
- 스토리지 볼륨은 파드의 일부로, 파드가 시작될 때 볼륨이 생성되고, 파드가 삭제되면 볼륨도 삭제


## 6.1  볼륨(Volumn) 소개
`볼륨(Volum)`은 파드 내의 컨테이너들이 공유할 수 있는 데이터 저장 공간

> - 파드의 구성요소로 pod spec 에서 정의가 필요
> - 다른 쿠버네티스 오브젝트처럼 자체적으로 생성, 삭제 X
> - 볼륨은 파드안의 모든 컨테이너에서 사용가능하지만, 접근하려면 컨테이너에서 각각 `마운트(Mount)` 필요

### 6.1.1 예제의 볼륨 설명

1. 공유 스토리지가 없는 동일 파드의 컨테이너 3개
   - Webserver : html 페이지를 서비스하고 액세스 로그를 저장하는 역할
   - ContentAgent : /var/html에 HTML 파일을 생성하는 에이전트를 실행하는 역할
   - LogRotator : 로그를 처리하는 역할
2. 두개의 볼륨을 공유하고 각 경로에 마운트 -> 컨테이너간 데이터 공유
   - publicHTML : Webserver, ContentAgent
   - logVol: Webserver, LogRotator

![image1](https://github.com/user-attachments/assets/f63b121e-9ccd-47a3-b877-43ca343f8832)
![image2](https://github.com/user-attachments/assets/bf927c96-31b6-40ec-8ef6-b5446739a0be)


### 6.1.2 사용 가능한 볼륨 소개
쿠버네티스는 다양한 볼륨 유형을 지원함
> - **emptyDir** : 일시적인 데이터를 저장하는 데 사용되는 간단한 빈 디렉터리 \
> - **hostPath**: 워커 노드의 파일시스템을 파드의 디렉터리로 마운트하는데 사용 \
> - **gitRepo** : 깃 리포지터리의 콘텐츠를 체크아웃해 초기화한 볼륨 \
> - nfs : NFS 공유를 파드에 마운트 \
> - **gcePersistentDisk, awsElasticBloakStore, azureDisk** : 클라우드 제공자의 전용 스토리지 마운트하는데 사용 \
> - cinder, cephfs, iscsi, flocker, g;isterfs, qubyte, rbd, flexVolum, vsphere Volumn.. : 다른 유형의 네트워크 스토리지를 마운트 하는데 사용 \
> - configMap, secret, downwardAPI : 쿠버네티스 리소스나 클러스터 정보를 파드에 노출하는 데 사용되는 특별한 유형의 볼륨 → 7장과 8장에서 설명 \
> - **persistentVolumClaim** : 사전에 혹은 동적으로 프로비저닝된 퍼시스턴스 스토리지를 사용하는 방법

## 6.2 볼륨을 사용한 컨테이너 간 데이터 공유

### 6.2.1 `emptyDir` 볼륨 사용하기

> - `emptyDir` 볼륨은 쿠버네티스에서 가장 기본적인 볼륨 유형
> - 이름 그대로 빈 디렉터리로 시작하며, 파드가 생성되면 비어 있는 상태로 초기화
> - `emptyDir`의 라이프사이클은 파드와 연결되어 있어, 파드가 삭제되거나 재스케줄링되면 그 안의 모든 데이터는 사라짐
> - 이 볼륨은 동일 파드 내 여러 컨테이너가 파일을 공유해야 할 때 유용
> - 또한, 메모리에 `emptyDir` 볼륨을 생성하여 대용량 데이터를 임시로 저장할 수 있음
> - 컨테이너는 자신의 파일 시스템에도 데이터를 쓸 수 있지만, 일부 환경에서는 쓰기 제한이 있을 수 있어 `emptyDir` 볼륨에 쓰는 것이 유일한 옵션인 경우도 있음


> ⚠️ `emptyDir` 볼륨을 사용하면 파드 내에서 데이터를 임시로 공유하거나 저장하는 데 매우 편리하지만, 휘발성 데이터에 적합하므로 영구적인 데이터 저장에는 적합하지 X

1. **파드 생성하기**
> 컨테이너 두개와 각 컨테이너에 각기 다른 경로로 마운트된 단일 볼륨을 갖도록 생성

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

2. 실행중인 파드 보기

    ```bash
    # 로컬 머신의 포트를 파드로 포워딩
    ❯ kubectl port-forward fortune 8080:80
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    
    # curl 요청
    ❯ curl http://localhost:8080
    Good news from afar can bring you a welcome visitor.
    ```


> ⚠️`emptyDir` 볼륨의 저장 위치 \
> 기본적으로는 파드가 배포된 노드의 디스크에 데이터를 저장 \
>  `medium`을 "Memory"로 설정하면 디스크 대신 노드의 RAM을 사용할 수 있음 \
>  이 경우 I/O 속도가 빨라지지만, RAM 용량을 많이 사용하게 되어 대용랑 데이터에는 적합하지 않음
> ```bash
>    volumn: 
>    	emptyDir:
>    		medium: Memory
>    ```


### 6.2.2 깃 리포지터리를 볼륨으로 사용하기

Kubernetes의 공식 문서에서도 사실상 **사용이 권장되지 않고**, Kubernetes 프로젝트에서는 유지보수가 중단된 상태
> - `gitRepo` 볼륨은  Git 리포지토리를 활용하는 방법
> - 컨테이너가 시작될 때, 특정 Git 저장소에서 데이터를 자동으로 클론해 사용할 수 있도록 설계된 볼륨
> - 클론된 데이터는 읽기 전용으로 마운트되어, 애플리케이션이 해당 데이터를 참조할 수 있지만 변경할 수는 없음
> - 깃 동기화를 위해선 추가 프로세스 필요 → 깃 동기화 컨테이너

![image.png](https://github.com/user-attachments/assets/743573ba-b2d7-4708-800b-cd39d514de18)

> ⚠️ `gitRepo` 볼륨 대안책
>
> - **Init 컨테이너** : 컨테이너 단위의 초기화 작업과 인증 설정, 오류 디버깅 등을 유연하게 제어할 수 있어 `gitRepo` 볼륨보다 안정적
> - **GitOps 도구** :  자동화된 동기화와 운영 효율성을 통해 Kubernetes 환경을 최신 상태로 유지

## 6.3 워커 노드 파일 시스템의 파일 접근

### 6.3.1 hostPath 볼륨 소개

> hostPath 볼륨은 **노드 파일시스템의 특정 파일이나 디렉터리**를 가리킴
> - 똑같은 노드에서 실행 중인 파드가 hostPath 볼륨의 동일 경로를 사용한다면 동일한 파일 표시
> - 노드의 파일시스템을 활용하여 `empty Dir`, `gitRepo`와 다르게 파드가 종료되도 삭제되지 않음.

![image.png](https://github.com/user-attachments/assets/39299d4d-eb31-439f-b707-e02face210ce)

> ⚠️ 데이터베이스의 데이터 디렉터리 같은 경우는 hostPath 볼륨을 사용하지 않아야함
>    - 노드 장애시 데이터 손실 위험
>    - 파드가 다른 노드로 스케줄링되면 이전 데이터 접근 불가

### 6.3.2 hostPath 볼륨을 사용하는 시스템 파드 검사하기
> 노드의 로그파일, 쿠버네티스 구성파일(kubeconfig), CA 인증서를 접근하기 위해서 hostPath 볼륨을 사용 \
> -> 7장, 8장 참고

```bash

# 시스템 서비스를 실행하는 Pod 조회
❯ kubectl get pods --namespace kube-system

NAME                                                      READY   STATUS    RESTARTS   AGE
coredns-76f75df574-mlpsw                                  1/1     Running   0          7d
coredns-76f75df574-nz46w                                  1/1     Running   0          7d
etcd-docker-desktop                                       1/1     Running   0          7d
kube-apiserver-docker-desktop                             1/1     Running   0          7d
kube-controller-manager-docker-desktop                    1/1     Running   0          7d
kube-proxy-5m2zg                                          1/1     Running   0          7d
kube-scheduler-docker-desktop                             1/1     Running   0          7d
nginx-ingress-ingress-nginx-controller-64c8ffc67f-rkzjs   1/1     Running   0          20h
storage-provisioner                                       1/1     Running   0          7d
vpnkit-controller       

# hostPath 확인
Volumes:
  ca-certs:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/ssl/certs
    HostPathType:  DirectoryOrCreate
  etc-ca-certificates:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/ca-certificates
    HostPathType:  DirectoryOrCreate
  flexvolume-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /usr/libexec/kubernetes/kubelet-plugins/volume/exec
    HostPathType:  DirectoryOrCreate
  k8s-certs:
    Type:          HostPath (bare host directory volume)
    Path:          /run/config/pki
    HostPathType:  DirectoryOrCreate
  kubeconfig:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/kubernetes/controller-manager.conf
    HostPathType:  FileOrCreate                                  1/1     Running   0          7d
```


## 6.4 퍼시스턴트 스토리지 사용

여러 노드에서 실행중인 파드가 동일한 데이터를 사용해야 하는 경우, `NAS(Network-Attached Storage)` 유형의 볼륨에 저장 필요

### 6.4.1 `GCE 퍼시스턴트 디스크`를 파드 볼륨으로 사용하기

1. `GCE 퍼시스턴트 디스크` 생성

```bash
❯ gcloud compute disks create mongodb \
    --size=1GiB \
    --zone=asia-northeast3-a
    
WARNING: You have selected a disk size of under [200GB]. This may result in poor I/O performance. For more information, see: https://developers.google.com/compute/docs/disks#performance.
Created [https://www.googleapis.com/compute/v1/projects/cybernetic-song-440602-c5/zones/asia-northeast3-a/disks/mongodb].
NAME     ZONE               SIZE_GB  TYPE         STATUS
mongodb  asia-northeast3-a  1        pd-standard  READY

New disks are unformatted. You must format and mount a disk before it
can be used. You can find instructions on how to do this at:

https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting
```

2. `GCE 퍼시스턴트 디스크 볼륨`을 사용하는 `파드` 생성

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  volumes:
  - name: mongodb-data
    gcePersistentDisk:
      pdName: mongodb
      fsType: ext4
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```
3. 파드 확인
```bash

❯ kubectl get pods
NAME      READY   STATUS              RESTARTS   AGE
fortune   2/2     Running             0          13h
mongodb   0/1     ContainerCreating   0          34s
```

![image.png](https://github.com/user-attachments/assets/598586b8-4592-4ab6-a237-4b96688ca052)

> mongodb 파드는 외부 GCE 퍼시스턴트 디스크를 참조하는 볼륨을 마운트

4. MongoDB 데이터베이스에 도큐먼트를 추가해 `퍼시스턴트 스토리지`에 데이터 쓰기

```bash
❯ kubectl exec -it mongodb -- mongosh
Current Mongosh Log ID: 6726fa2b12197c20c9fe6910
Connecting to:          mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.3.2
Using MongoDB:          8.0.3
Using Mongosh:          2.3.2

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/

To help improve our products, anonymous usage data is collected and sent to MongoDB periodically (https://www.mongodb.com/legal/privacy-policy).
You can opt-out by running the disableTelemetry() command.

------
   The server generated these startup warnings when booting
   2024-11-03T04:19:52.882+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
   2024-11-03T04:19:54.067+00:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
   2024-11-03T04:19:54.068+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2024-11-03T04:19:54.068+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2024-11-03T04:19:54.068+00:00: We suggest setting the contents of sysfsFile to 0.
   2024-11-03T04:19:54.068+00:00: Your system has glibc support for rseq built in, which is not yet supported by tcmalloc-google and has critical performance implications. Please set the environment variable GLIBC_TUNABLES=glibc.pthread.rseq=0
   2024-11-03T04:19:54.068+00:00: vm.max_map_count is too low
------
```

5. 데이터 추가
```bash

# json Document 추가

test> use mystore
switched to db mystore

## foo라는 이름의 컬렉션에 {name: 'foo'}라는 데이터를 삽입
mystore> db.foo.insert({name:'foo'})
DeprecationWarning: Collection.insert() is deprecated. Use insertOne, insertMany, or bulkWrite.
{
  acknowledged: true,
  insertedIds: { '0': ObjectId('67270b6512197c20c9fe6911') }
}

# json Documnet 확인
mystore> db.foo.find()
[ { _id: ObjectId('67270b6512197c20c9fe6911'), name: 'foo' } ]

```

6. 기존 파드 삭제 후, 파드를 다시 생성하고 이전 파드가 저장한 데이터를 읽을 수 있는지 확인

```bash
# 기존 파드 삭제
❯ kubectl delete pod mongodb
pod "mongodb" deleted

# 파드 재생성
❯ kubectl apply -f mongodb-pod-gcepd.yaml
pod/mongodb created

# 데이터 있는지 확인
❯ kubectl exec -it mongodb -- mongosh
Current Mongosh Log ID: 67270c4e8ce02bbc87fe6910

test> use mystore
switched to db mystore

mystore> db.foo.find()
[ { _id: ObjectId('67270b6512197c20c9fe6911'), name: 'foo' } ]
```

### 6.4.2 기반 퍼시스턴트 스토리지로 다른 유형의 볼륨 사용하기

사용하는 Kubernetes 클러스터의 클라우드 제공업체에 따라 적절한 퍼시스턴트 스토리지 옵션을 선택

> - **Google Kubernetes Engine (GKE)**: GCE Persistent Disk 사용
> - **Amazon EKS (Elastic Kubernetes Service)**: AWS Elastic Block Store (EBS) 사용
> - **Azure Kubernetes Service (AKS)**: Azure Disk 또는 Azure File 사용

> ⚠️ 파드의 볼륨이 실제 기반 인프라스트럭처(여기서는 GCE Disk)를 참조한다는 것은 쿠버네티스가 추구하는 바가 아님 \
> 인프라 스트럭처 관련 정보를 파드에 정의한다는 것은 파드 정의가 특정 쿠버네티스 클러스터에 밀접하게 연결됨을 의미

## 6.5 기반 스토리지 기술과 파드 분리

> - 앞서 살펴본 퍼시스턴트 볼륨유형은 파드개발자가 실제 넽워크 스토리지 인프라스트럭처에 관한 지식을 갖춰야함
> - 이는 인프라스트럭처를 숨긴다는 쿠버네티스의 기본 아이디어에 반함
> - 쿠버네트에서 배포하는 개발자는 기저에 어떤 종류에 스토리지 기술이 사용되는지 알 필요가 없음
> - 인프라 스트럭처 관련 처리는 클러스터 관리자의 영역

### 6.5.1 퍼시스턴트볼륨과 퍼시스턴트볼륨 클레임
> - 퍼시스턴트 볼륨(PV)과 퍼시스턴트 볼륨 클레임(PVC)을 통해 애플리케이션이 스토리지를 요청할 수 있으며, 클러스터 관리자가 인프라스트럭처를 관리하는 것이 이상적
> - 하단의 그림은 파드, 퍼시스턴트 볼륨, 퍼시스턴트 볼륨 클레임과 실제 기반 스토리지의 관계를 나타내는 그림

![image.png](https://github.com/user-attachments/assets/3af1c1a0-be8c-4b19-904f-d2957a7c4f01)


### 6.5.2 퍼시스턴트 볼륨 생성
1. GCE 퍼시스턴트 볼륨을 기반으로 한 `PV`를 생성
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
	# PV 용량
  capacity: 
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  # 클레임이 해제된 후에도 PV는 유지되야함  
  persistentVolumeReclaimPolicy: Retain
  # 이전에 생성한 GCE 퍼시스턴트 디스크, 다른걸 사용한 경우 옵션값이 달라짐
  gcePersistentDisk:
    pdName: mongodb
    fsType: ext4
```

2. PV 조회

```yaml
❯ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
mongodb-pv   1Gi        RWO,ROX        Retain           Available                          <unset>                          6s
```

3. 구조 살펴 보기
   ![image.png](https://github.com/user-attachments/assets/623d87c5-9abe-4b2c-b0c1-d787b744c926)

> - `퍼시스턴트 볼륨 (PV)`는 클러스터의 스토리지 자원으로, 클러스터 노드(마스터 노드)에 존재하며 네임스페이스의 제약을 받지 않음
> - `퍼시스턴트 볼륨 클레임 (PVC)`는 네임스페이스에 속함
    >   - 여러 네임스페이스에 있는 파드들이 동일한 PV에 접근할 수는 없음
>   - 한 개의 PV는 한 개의 PVC와 바인딩되기 때문

### 6.5.3`퍼시스턴트 볼륨 클레임 (PVC)` 생성을 통한 `퍼시스턴트 볼륨 (PV)` 요청

> 생성된 `퍼시스턴트 볼륨 (PV)`를 사용하려면 `퍼시스턴트 볼륨 클레임 (PVC)`생성이 필요 \
> `PVC`는 파드와 독립적으로 생성되어야 하며, 파드가 죽더라도 `PVC`는 지속됨 \
> 파드가 PV를 이용하는 방법은 `PVC`를 생성한 후, `파드`와 `PVC`를 연결

1. `PVC` 생성
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc 
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  # 동적 프로비저닝 참고
  storageClassName: ""
```

2. `PVC` 조회 → `PV` 와 바인딩 된 것을 확인 가능

```yaml
❯ kubectl get pvc
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mongodb-pvc   Bound    mongodb-pv   1Gi        RWO,ROX                       <unset>                 81s
```

> ⚠️ mongo-pv PV는 이제 mongodb-pvc에 바인딩 돼 다른 Claim에 바인딩 될 수 없다.

```yaml
❯ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
mongodb-pv   1Gi        RWO,ROX        Retain           Bound    default/mongodb-pvc                  <unset>                          20m
```

### 6.5.4 파드에서 PVC(퍼시스턴트볼륨클레임) 사용하기

1. 파드 생성시,  볼륨에 persistentVolumeClaim를 참조

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

2. `GCE 퍼시스턴트 디스크` 에 저장했던 데이터 확인

```yaml
❯ kubectl apply -f mongodb-pod-pvc.yaml
pod/mongodb created
❯ kubectl exec -it mongodb -- mongosh
Current Mongosh Log ID: 672719d77755789a86fe6910
Connecting to:          mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.3.2
Using MongoDB:          8.0.3
Using Mongosh:          2.3.2
mongosh 2.3.3 is available for download: https://www.mongodb.com/try/download/shell

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/

------
   The server generated these startup warnings when booting
   2024-11-03T06:36:03.406+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
   2024-11-03T06:36:04.704+00:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
   2024-11-03T06:36:04.705+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2024-11-03T06:36:04.705+00:00: For customers running the current memory allocator, we suggest changing the contents of the following sysfsFile
   2024-11-03T06:36:04.705+00:00: We suggest setting the contents of sysfsFile to 0.
   2024-11-03T06:36:04.705+00:00: Your system has glibc support for rseq built in, which is not yet supported by tcmalloc-google and has critical performance implications. Please set the environment variable GLIBC_TUNABLES=glibc.pthread.rseq=0
   2024-11-03T06:36:04.705+00:00: vm.max_map_count is too low
------

test> use mystore
switched to db mystore
mystore> db.foo.find()
[ { _id: ObjectId('67270b6512197c20c9fe6911'), name: 'foo' } ]
```

### 6.5.5 PV(퍼시스턴트 볼륨) 과 PVC 사용의 장점

> **개발자는 스토리지 기술을 몰라도 됨 →** 파드가 어떤 스토리지에 연결되는지 신경 쓰지 않아도 됨

![image.png](%E1%84%8C%E1%85%A6%E1%84%86%E1%85%A9%E1%86%A8%20%E1%84%8B%E1%85%A5%E1%86%B9%E1%84%8B%E1%85%B3%E1%86%B7%201322a0bd5e158059ad30c9ce165595fa/image%207.png)

### 6.5.6 퍼시스턴트 볼륨 재사용

> PVC를 삭제하고 나서 새로운 PVC를 만들면, 삭제된 PVC가 바인딩되던 PV를 사용할 수 있을까? \
> → X. 자동으로 바인딩은 어려움.

> ⚠️PVC가 삭제되더라도, 기존 PVC와 바인딩된 PV는 자동으로 삭제되지 않으며 해당 PVC와 바인딩 상태를 계속 유지 \
> 새 PVC를 같은 PV에 연결하려면 PV를 **수동으로 해제하거나 재설정 필요** \
> 그렇지 않으면 기존 PVC의 잔여 바인딩 정보 때문에 새 PVC가 같은 PV를 사용하지 못함

> ⚠️️ PV의 `reclaimPolicy` 설정에 따라 동작이 달라짐
> - **Retain** : 삭제된 PVC와의 바인딩 정보를 유지하며, 수동으로 데이터를 삭제하고 PV를 해제
> - **Delete**: PVC 삭제 시 PV와 해당 데이터를 모두 삭제 (**기반 스토리지**에서도 PV와 연관된 데이터가 삭제)
>

![image.png](https://github.com/user-attachments/assets/df0a7f93-4a23-4f52-a07c-12b87ada4507)


## 6.6 퍼시스턴트볼륨(PV)의 동적 프로비저닝

> - PV를 프로비저닝 하는 것 대신, `스토리지 클래스`를 정의하여 해결하는 방법 소개
> - PVC를 생성하면 결과적으로 새로운 퍼시스턴트볼륨이 프로비저닝 되는 방식
    >     - 스토리지 클래스를 이용

### 6.6.2 PVC에서 StorageClass 요청하기

1. 기존 PV, PVC,파드 삭제

```bash
❯ kubectl delete pod mongodb
pod "mongodb" deleted

❯ kubectl delete pvc mongodb-pvc
persistentvolumeclaim "mongodb-pvc" deleted

❯ kubectl delete pv mongodb-pv
persistentvolume "mongodb-pv" deleted
```

2. StorageClass 생성

```yaml
# StorageClass -> fast 생성

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

3. 특정 StorageClass를 요청하도록 `PVC` 생성

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc 
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce

# PVC 조회    
❯ kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mongodb-pvc   Bound    pvc-b2face38-cffd-40ca-8012-76b6fc874d73   1Gi        RWO            fast           <unset>                 9s
    
# PV 생성됐는지 확인
❯ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-b2face38-cffd-40ca-8012-76b6fc874d73   1Gi        RWO            Delete           Bound    default/mongodb-pvc   fast           <unset>                          49s    
```
>  - 클레임을 지정하면 `fast` 스토리지 클래스 리소스에 참조된 프로비저너가 작동
     >     - **GCE Persistent Disk 생성**: PVC 요청에 따라 Google Cloud에서 실제 GCE Persistent Disk를 생성
>     - **PersistentVolume (PV) 리소스 생성**: 생성된 GCE Persistent Disk를 참조하는 PV 리소스를 자동으로 Kubernetes에 생성하여 등록
>  - RECLAIM POLICY 정책이 사용되어, PVC가 삭제되면 PV도 삭제

### 6.6.3 StorageClass를 지정하지 않는 동적 프로비저닝

- StorageClass 지정하지 않으면 StorageClass 기본값으로 작동
- 스토리지 클래스 조회 →  standard가 default

```bash
❯ kubectl get sc
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
fast                 kubernetes.io/gce-pd    Delete          Immediate              false                  23m
premium-rwo          pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   4h33m
standard (default)   kubernetes.io/gce-pd    Delete          Immediate              true                   4h33m
standard-rwo         pd.csi.storage.gke.io   Delete          WaitForFirstConsumer   true                   4h33m
```

- StorageClass를 지정하지 않고, PVC 생성 하면, standard StorageClass가 지정될까? → O

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc2 
spec:
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
    
```

- 생성한 PVC 확인

```bash
❯ kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mongodb-pvc    Bound    pvc-b2face38-cffd-40ca-8012-76b6fc874d73   1Gi        RWO            fast           <unset>                 18m
mongodb-pvc2   Bound    pvc-c72e951f-72a3-4c89-82cd-c05b9995e385   1Gi        RWO            standard       <unset>                 4s                                                                   standard-rwo   <unset>                 48s
```

> - PVC를 미리 프로비저닝된 PV로 바인딩을 강제하려면? (새로운 PV 생성X)
>    - storageClassName을 빈문자열로 지정
>    - storageClassName : “”
>    - 바인딩될 PV가 없다면? → PVC 생성 안됨

### PV 동적 프로지버닝의 전체 그림

![image.png](https://github.com/user-attachments/assets/7abb9de9-df8d-429f-afef-92c5b0ab9ca8)
1. **클러스터 관리자는 PV 프로비저너를 지정**
   - 클러스터 관리자는 사용할 스토리지 기술에 맞는 PV 프로비저너를 설정
2. **클러스터 관리자는 PV 프로비저너를 참조하는 StorageClass를 생성**
   - 관리자는 특정 프로비저너를 사용하여 PV를 동적으로 생성하기 위한 StorageClass를 생성
3. **PVC 생성 시 StorageClass를 참조해서 생성**
   - 사용자는 PVC를 생성할 때 원하는 스토리지 클래스(StorageClass)를 지정
4. **쿠버네티스는 StorageClass를 보고 지정된 프로비저닝에게 PV를 프로비저닝하도록 요청**
   - PVC가 생성되면, Kubernetes는 해당 PVC에 지정된 StorageClass를 확인하고, 적절한 PV 프로비저너에 PV 생성을 요청
5. **PV 프로비저너는 실제 스토리지(여기서는 GCE Disk) 및 PV를 생성하고 PVC에 바인딩**
   - 프로비저너는 클라우드 제공자의 API를 사용하여 실제 스토리지(예: GCE Persistent Disk)를 생성하고, 그에 따라 PV를 생성하며, 이 PV를 PVC에 바인딩
6. **사용자는 PVC를 참조하는 파드를 생성**
   - 사용자는 생성된 PVC를 참조하여 파드를 생성하고, 이 파드는 PVC를 통해 PV에 접근하여 데이터 I/O