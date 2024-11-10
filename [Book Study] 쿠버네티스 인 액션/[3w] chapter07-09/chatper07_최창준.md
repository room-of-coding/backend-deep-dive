
# 7장 컨피그맵과 시크릿: 애플리케이션 설정

빌드된 애플리케이션 자체에 포함하지 말아야하는 설정(배포된 인스턴스별로 다른 세팅, 외부 시스템 액세스를 위한 자격 증명 등)을 위해 컨피그맵과 시크릿이 필요하다.

## 7.2 컨테이너에 명령줄 인자 전달

### 7.2.1 도커에서 명령어와 인자 정의

- ENTRYPOINT: 컨테이너가 시작될 때 호출될 명령어 정의
- CMD: ENTRYPOINT에 전달되는 인자 정의

ENTRYPOINT로 실행하되, 기본 인자를 정의할려는 경우에만 CMD를 지정한다.

```yaml
FROM alpine:latest
ENTRYPOINT ["curl"]          # 기본 명령을 curl로 설정
CMD ["https://example.com"]   # curl에 전달할 기본 인자를 설정
```

**shell과 exec 형식 차이**

- shell 형식: ENTRYPOINT node app.js
- exec 형식: ENTRYPOINT [”node”, “app.js”]

**exec 형식 권장**

exec 형식으로 실행하면 컨테이너 내부에서 프로세스를 직접 실행한다.

![image](https://github.com/user-attachments/assets/3155d464-0299-4d2d-9243-ea74c16bcc63)


shell 형식으로 실행하면 컨테이너의 메인 프로세스(PID 1번)이 shell 프로세스가 되고 이를 통해 원하는 프로세스를 실행한다.

![image](https://github.com/user-attachments/assets/7918b8d3-1d90-499d-9757-197a4911f065)


shell이 불필요한 경우이므로 exec 형식을 권장한다.

### 7.2.2 쿠버네티스에서 명령과 인자 재정의

ENTRYPOINT, CMD는 쿠버네티스에서 각각 command, args 속성에 해당한다.

```yaml
kind: Pod
spec:
  containers:
    - image: some/image
      command: ["/bin/command"]
      args: ["arg1", "arg2", "arg3"]
```

| 도커 | 쿠버네티스 | 설명 |
| --- | --- | --- |
| ENTRYPOINT | command | 컨테이너 안에서 실행되는 실행파일 |
| CMD | args | 실행파일에 전달되는 인자 |

! 수정 코드

책에서의 내용은 amd64 이미지만 제공하여 arm에서 실행 불가

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
    - image: "changjun0823/fortune-pod-args"
      args: ["2"]
      name: html-generator
      volumeMounts:
        - name: html
          mountPath: /var/htdocs
  volumes:
    - name: html
      emptyDir: {}
```

내부 파일로 작동 확인, args 값 정상 반영 확인

: 2초 마다 메시지 나오는 것 확인함

![image](https://github.com/user-attachments/assets/5a389210-2392-4208-bf20-02cbc89eaa12)


![image](https://github.com/user-attachments/assets/8969531f-fc7c-4e15-b81b-d094002a3165)


---

## 7.3 컨테이너의 환경변수 설정

행운의 메시지를 출력하는 forutneloop.sh에서 $INTERVAL 변수의 초기화 부분을 제거하여 env로 받을 수 있도록 수정함. bash 이기 때문에 $INTERVAL만 써주면 되고, 애플리케이션 종류마다 다른 형태로 작성해줘야 환경변수로 인식함

- 자바 애플리케이션: System.getenv(”INTERVAL”)
- NodeJS: process.env.INTERVAL
- 파이썬: os.environ[’INTERVAL’]

이런 식으로 env로 작성해준다.

```yaml
spec:
  containers:
  - image: changjun0823/fortune-pod-env
    env:
    - name: INTERVAL
      value: "30"
```

container inspect 결과

![image](https://github.com/user-attachments/assets/1e0f061f-240f-49c4-b053-b3abfff10cb6)


먼저 선언한 env값 상대참조 가능

```yaml
env：
- name: FIRST_VAR
	value： "foo"
- name: SECOND_VAR
	value： "$（FIRST_VAR）bar"
```

## 7.4 컨피그맵으로 설정 분리

### 7.4.1 컨피그맵 소개

- 키:밸류 로 구성된 맵
- ConfigMap의 데이터는 k8s의 오브젝트이며, etcd 데이터베이스에 저장되어 관리된다. Pod에서는 ConfigMap을 환경 변수로 불러오거나, 파일로 마운트하여 사용할 수 있다

### 7.4.2 컨피그맵 생성(CRD)

- 생성 방법: literal

```jsx
$ kubectl create configmap {컨피그맵 이름} --from-literal=foo=bar --from-literal=bar=baz
```

여러 개의 값을 정의할 때 —from-literal을 여러 번 사용하여 정의 가능

create를 하면 쿠버네티스 API에 게시하고 etcd 데이터베이스에 저장

- 생성 방법: 파일

```jsx
$ kubectl create configmap {컨피그맵 이름} --from-file={컨피그맵 파일 이름.conf}
```

—from-file 이라는 인수도 여러 번 사용하여 여러 파일 추가 가능

파일 이름 대신 디렉토리로 지정하여 모든 파일을 가져와서 적용가능

- 조회

```jsx
$ kubectl get configmaps
```

- yaml로 된 configmap 출력

```jsx
$ kubectl get configmap {컨피그맵 이름} -o yaml
```

- 삭제

```jsx
$ kubectl delete configmap {컨피그맵 이름}
```

### 7.4.3 컨피그맵 항목을 환경변수로 컨테이너에 전달

valueFrom.configMapKeyRef를 사용하여 파일명 및 key를 env값에 전달함

![image](https://github.com/user-attachments/assets/13a92344-b350-4394-aa0f-e0265386bf50)


### 7.4.4 컨피그맵의 모든 항목을 한 번에 환경변수로 전달

모든 환경 변수를 일일이 지정하기가 너무 번거롭기 때문에 env 대신 **envFrom** **을 통해 제공 가능**

my-config-map 파일 안에 CONFIG_FOO-BAR, CONFIG_SOMETHING-GOOD 등으로 prefix가 CONFIG_인 환경변수들을 지정해놓고 configMapRef로 참조하도록하면 이 환경 변수 값들을 참고로 해서 컨테이너들에 적용된다는 의미

![image](https://github.com/user-attachments/assets/eddefa2d-7d9c-46c0-9c69-123d50be26d7)



### 7.4.5 컨피그맵 항목을 명령줄 인자로 전달

$(환경변수이름) 문법을 사용하여 args(명령줄 인자)로 전달 가능

![image](https://github.com/user-attachments/assets/6219919c-a35c-4ab7-9035-39f8cc15a77c)


### 7.4.6 컨피그맵 볼륨을 사용해 컨피그맵 항목을 파일로 노출

대형 설정 파일들을 컨테이너에 전달하기 위해 특수 볼륨 유형 중 하나인 컨피그맵 볼륨을 사용할 수 있음

* ConfigMap 볼륨은 기본적으로 emptyDir 볼륨과 비슷한 방식으로 작동하지만, 노드의 디스크에 저장되지 않고 임시로 메모리에 유지된다. Pod가 종료되면 사라지는 임시 성격의 볼륨

예제에서 사용한 nginx 서버에 gzip 기능을 추가하기 위해 nginx 서버를 위한 아래와 같은 설정 파일을 추가해야하는 상황을 가정함

```yaml
server｛
  listen 80;
  Server_name www.kubia-example.com；

  gzip on；
  gzip_types text/plain application/xml；
  location / ｛
    root /usr/share/nginx/html；
    index index.html index.htm；
  ｝
}
```

그리고 메인 fortune 컨테이너를 위한 설정 내용을 sleep-interval.txt라는 파일에 넣고 위 설정 파일과 같은 디렉토리 안에 넣어둔다.

![image](https://github.com/user-attachments/assets/a9a786aa-52b8-4d13-a529-15e93770f70e)


그리고 이를 “configmap-files”라는 이름의 configmap으로 생성한다.

```jsx
$ kubectl create configmap fortune-config --from-file=configmap-files
```

이제 아래처럼 매핑하면 nginx의 설정용 디렉토리인 /etc/nginx/conf.d 디렉터리에서 위에서 생성한 configmap-files를 참조해서 설정을 넣어준다.

![image](https://github.com/user-attachments/assets/525d1757-678e-4d2a-993c-76c00d77b220)


**볼륨에 특정 컨피그맵 항목 노출**

nginx용 설정 파일만 볼륨에 노출되도록 설정할 수 있음

![image](https://github.com/user-attachments/assets/d76b799f-44e9-4b49-b8a8-79b38b8e0c23)



items 항목으로 특정 파일만 노출

**subPath: 파일 숨김 처리 방지**

mountPath를 /etc로 지정하는 경우 볼륨에 있는 /etc 경로 전체가 컨테이너의 /etc에 마운트 됨. 이럴 경우 기존 컨테이너에 있는 /etc 경로의 파일들이 숨김처리됨

아래와 같이 mountPath를 일부 파일로만 지정하고, subPath를 주면 myvolume에 포함된 myconfig.conf라는 파일만을 컨테이너의 /etc/someconfig.conf 파일로써 마운트하게 된다. 이러면 기존 컨테이너에 있던 파일들은 그대로 유지된다.

```yaml
spec：
containers：
  - image: some/image
    volumeMounts：
    - name: myvolume
      mountPath:/etc/someconfig.conf
      subPath:myconfig.conf
```

**컨피그맵 볼륨 안에 있는 파일 권한 설정**

기본 컨피그맵 볼륨의 파일 권한들은 644(-rw-r-r—: owner는 read/write 가능, group과 other는 read만 가능)이다. defaultMode 값을 주어서 권한을 변경할 수 있다.

```yaml
volumes:
  - name: config
    configMap:
      name: fortune-config
      defaultMode: "6600"
```

### 7.4.7 애플리케이션을 재시작하지 않고 애플리케이션 설정 업데이트

컨피그맵 업데이트 시, 볼륨의 파일들이 업데이트되며 컨테이너에 신호를 보내 값을 업데이트할 수 있다. 다만 실시간 반영은 안되고 1~2분 정도 걸린다는 점을 기억

**심볼릭 링크 참조**

심볼릭 링크란 특정 파일이나 디렉터리를 가리키는 가상 파일이다. 심볼릭 링크를 통해 원본 파일이 변경되면 이 링크를 접근하는 모든 경로에서도 변경이 반영되는 방식이다.

쿠버네티스는 ConfigMap을 마운트 할 때, 직접 파일을 Pod에 복사하는 것이 아니라 심볼릭 링크 형태로 참조한다. 따라서 ConfigMap이 변경되면 k8s는 새로운 버전의 ConfigMap을 생성하고 심볼릭 링크를 새로운 ConfigMap을 가리키도록 업데이트 한다. k8s는 주기적으로 심볼릭 링크를 참조하기 때문에 pod을 재시작하지 않아도 변경사항을 읽을 수 있는 것이다. 또한 즉각적이지 않고 1~2분의 지연이 있을 수 있다.

→ 유의점: k8s의 설정 파일은 그대로인데, ConfigMap을 변경하여 특정 컨테이너들만 업데이트 된다면 여러 컨테이너들을 실행 중일 때 지연 발생으로 인해 최대 1분 정도 동안 파드의 파일들이 동기화되지 않은 상태로 존재할 수 있음을 이해해야함

## 7.5 시크릿으로 민감한 데이터를 컨테이너에 전달

### 7.5.1 시크릿 소개

보안이 유지되어야하는 자격증명, 개인 암호화 키와 같은 민감한 정보를 전달하기 위한 목적으로 컨피그맵과 구분되어 제공됨

- k8s는 시크릿에 접근해야하는 파드가 실행되고 있는 노드에만 개별 시크릿을 배포함
- 노드 자체적으로 메모리에만 시크릿을 저장하고 물리 저장소에 기록되지 않도록 함
    - 물리저장소는 시크릿을 삭제한 후에도 디스크를 완전히 삭제하는 작업이 필요하기 때문

### 7.5.2 기본 토큰 시크릿 소개

모든 파드에는 secret 볼륨이 자동으로 연결돼 있음. default-token-xxxx 같은 이름의 시크릿을 참조함.

시크릿은 리소스이기 때문에 아래와 같이 조회 가능

```jsx
$ kubectl get secrets
```

```jsx
$ kubectl describe secrets
```

![image](https://github.com/user-attachments/assets/5ecd79c3-c083-4746-88e2-2418180af2a2)


기본 시크릿이 갖고 있는 항목들

![image](https://github.com/user-attachments/assets/99fe0b15-2131-4096-8a70-f41cd86475a3)


각 파드에 자동으로 마운트되는 시크릿

### 7.5.3 시크릿 생성

nginx 컨테이너가 HTTPS 트래픽을 제공하는 예시로 학습해봄. 인증서와 개인키를 생성하고 secret으로 등록

```jsx
$ openssl genrsa -out https.key 2048
```

```jsx
$ openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.kubia-example.com
```

추가로, 더미 파일인 foo를 만들고 bar라는 문자열을 저장

```jsx
$ echo bar > foo
```

```jsx
$ kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo
```

- generic은 시크릿의 타입이며, 도커 레지스트리를 사용하기 위한 docker-registry, TLS 통신을 위한 tls, generic 타입이 있음
- 시크릿용 볼륨도 인메모리 파일시스템(tmpfs)를 사용함

### 7.5.4 컨피그맵과 시크릿 비교

컨피그맵은 일반 텍스트인 반면, 시크릿은 Base64 인코딩 문자열로 표시되어 설정하고 읽을 때마다 인코딩과 디코딩을 해야함

![image](https://github.com/user-attachments/assets/cd913748-96de-44ac-b632-a0e59da251b4)


Base64로 인코딩된 시크릿 내용

- 또한 시크릿은 최대 1MB로 크기가 제한됨

StringData 필드

stringData 필드로 쓰면 일반 텍스트 형식으로 지정가능. 다만 쓰기 전용이라 설정할 때만 사용되며 kubectl get -o yaml 명령어로 조회 시에는 data 항목 아래 Base64로 인코딩되어 표시됨

![image](https://github.com/user-attachments/assets/094cc8ed-b15a-4555-b1b4-cb44b98bcb49)


### ⭐ 7.5.5 파드에서 시크릿 사용

위에서 설정한 시크릿의 내용을 이미 컨피그맵으로 설정해놓은 설정 파일과 일치시킴(https.cert, https.key)

```jsx
$ kubectl edit configmap fortune-config
```

![image](https://github.com/user-attachments/assets/7f3ae82f-1b20-4de0-be35-d2147fb63b08)


nginx의 설정이 /etc/nginx/ 경로에서 설정 파일들을 읽도록 되어있음

![image](https://github.com/user-attachments/assets/b0c4635d-17bb-4743-b365-9d9a3c6f75b7)


fortune-https 시크릿을 파드에 마운트

fortune-pod-https.yaml 파일의 설정을 살펴본다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
  - image: changjun0823/fortune-pod-args
    name: html-generator
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: https.conf
  - name: certs
    secret:
      secretName: fortune-https
```

1. volumes: 정의 부분에서 html, config, certs라는 3개의 볼륨을 지정하고 있음
    1. html 볼륨은 emptyDir 타입. volumeMounts 항목을 보면 html-generator와 web-server라는 두 컨테이너가 해당 볼륨을 공유하도록 되어있음
    2. config는 configMap으로 fortune-config라는 이름의 ConfigMap에서 데이터를 가져옴
        1. items 필드를 사용하여 ConfigMap 내의 my-nginx-config.conf라는 key만을 선택적으로 마운트
        2. path를 https.conf로 지정했고 html 컨테이너의 volumeMounts 항목에서 mountPath를 지정했으므로 my-nginx-config.conf라는 key에 해당하는 값이 컨테이너의 /etc/nginx/conf.d/https.conf 파일로 마운트됨
        3. fortune-config의 my-nginx-config.conf 키값에 해당하는 데이터가 업데이트되면 k8s에서 동기화하여 pod 내의 파일에도 반영함
    3. certs 볼륨은 fortune-https라는 Secert을 참조
        1. fortune-https라는 이름의 secret을 참조하고, /etc/nginx/certs/ 경로에 마운트됨
2. 1-b.에 따라 fortune-config.yaml 파일을 아래와 같이 수정

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortune-config
data:
  my-nginx-config.conf: |
    server {
        listen  80;
        listen  443 ssl;
        server_name www.kubia-example.com;
        ssl_certificate certs/https.cert;
        ssl_certifiacte certs/https.key;
        ssl_protocoals  TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers HIGH:!aNULL:!MD5;
        location / {
          root /usr/share/nginx/html;
          index index.html index.htm;
        }
    }
    sleep-interval: |
      25
```

1. c에 따라 아래 명령어를 실행(위에서 실행하지 않았었다면)

```jsx
$ kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo
```

[https.cert](https://prod-files-secure.s3.us-west-2.amazonaws.com/fbf82d43-89c8-4d71-97d9-bd3af84b398d/cc0639a5-309b-48a9-9a64-aaf95ac7ae75/https.cert)

[https.key](https://prod-files-secure.s3.us-west-2.amazonaws.com/fbf82d43-89c8-4d71-97d9-bd3af84b398d/a2c23075-0233-4578-889e-0a74d07b75eb/https.key)

!! 주의사항

아래와 같은 방식으로 생성해버리면 fortune-config라는 key를 갖는 1개의 data만 생성됨

```jsx
$ kubectl create configmap fortune-config --from-file=fortune-config.yaml
```

![image](https://github.com/user-attachments/assets/4d17435b-2da6-456e-97c5-7e6dbdbb3cdf)


파일 내부에 my-nginx-config.conf, sleep-interval 이라는 2개의 key를 갖는 configmap을 생성하고 pod에서 참조하는게 목적이기 때문에, 아래처럼 apply를 적용해줘야함

```jsx
$ kubectl apply -f ./fortune-config.yaml
```

![image](https://github.com/user-attachments/assets/c9afc9a1-565a-4445-9060-2bd317ee86f9)


**결과 확인**

```yaml
$ kubectl port-forward fortune-https 8443:443

(다른 터미널 탭에서)
$ curl https://localhost:8443 -k -v
```

![image](https://github.com/user-attachments/assets/2b2243d7-9ba6-4be8-980c-f6a917b67a14)


### 7.5.6 이미지를 가져올 때 사용하는 시크릿 이해

퍼블릭 레지스트리가 아니라 프라이빗 레지스트리에서 이미지를 가져오는 경우 자격증명이 필요하다. 이를 시크릿으로 처리할 수 있다.

명령어 방식(도커 허브 기준)

```bash
$ kubectl create secret docker-registry {시크릿 이름 지정} \
--docker-username={사용자ID} --docker-password={사용자 패스워드} \
--docker-email={사용자 email}
```

파드에 추가하는 방식

imagePullSecrets 항목을 추가하면 된다.

![image](https://github.com/user-attachments/assets/4b59dfef-1368-4e40-90a4-4022bc308ff5)
