## 카프카 설치

### 주키퍼 카프카 설치

~~~yaml
version: '3'
services:
  zookeeper-1:
    image: wurstmeister/zookeeper:3.4.6
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_ID: 1
      ZOOKEEPER_SERVER_1: zookeeper-1:2888:3888
      ZOOKEEPER_SERVER_2: zookeeper-2:2888:3888
      ZOOKEEPER_SERVER_3: zookeeper-3:2888:3888
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - kafka-network

  zookeeper-2:
    image: wurstmeister/zookeeper:3.4.6
    ports:
      - "2182:2181"
    environment:
      ZOOKEEPER_ID: 2
      ZOOKEEPER_SERVER_1: zookeeper-1:2888:3888
      ZOOKEEPER_SERVER_2: zookeeper-2:2888:3888
      ZOOKEEPER_SERVER_3: zookeeper-3:2888:3888
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - kafka-network

  zookeeper-3:
    image: wurstmeister/zookeeper:3.4.6
    ports:
      - "2183:2181"
    environment:
      ZOOKEEPER_ID: 3
      ZOOKEEPER_SERVER_1: zookeeper-1:2888:3888
      ZOOKEEPER_SERVER_2: zookeeper-2:2888:3888
      ZOOKEEPER_SERVER_3: zookeeper-3:2888:3888
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - kafka-network

  kafka-1:
    image: wurstmeister/kafka:2.12-2.5.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
    networks:
      - kafka-network

  kafka-2:
    image: wurstmeister/kafka:2.12-2.5.0
    ports:
      - "9093:9092"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
    networks:
      - kafka-network

  kafka-3:
    image: wurstmeister/kafka:2.12-2.5.0
    ports:
      - "9094:9092"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9094
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
    networks:
      - kafka-network

networks:
  kafka-network:
    driver: bridge
~~~

* 도커 컴포즈 런

~~~shell
docker-compose up -d
~~~

### 카프카 쉘

* [카프카 쉘 다운로드](https://kafka.apache.org/downloads)
* 명령어 날려보기
    * 토픽 만들기
      ~~~shell
      /usr/local/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --replication-factor 1 --partitions 1 --topic test
      ~~~
    * 메세지 보내기
      ~~~shell
      /usr/1ocal/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
      test message 1
      test message 2
      ~~~
    * 메시지 받기
      ~~~shell
      /usr/1ocal/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
      test message 1
      test message 2
      ~~~

### 주요 브로커 옵션

| 파라미터                              | 용도                                                                                               | msk 지원 여부 | 기본값   |
|-----------------------------------|--------------------------------------------------------------------------------------------------|-----------|-------|
| log.dirs                          | 별도의 로그 세그먼트 저장 디렉토리 지정                                                                           | X         | false |
| num.recovery.threads.per.data.dir | 브로커가 시작과 종료시에만 활성화 되기 때문에 최대한 많이 할당하는게 좋음 (num.recovery.threads.per.data.dir * 1og.dirs) 만큼 할당됨 | X         | empty |
| auto.create.topics.enable         | 프로듀서가 토픽 쓸때, 컨슈머가 토픽을 읽을 때, 클라이언트가 토픽을 메타를 요청할때 생성                                               | O         | false |
| auto.leader.rebalance.enable      | 백그라운드 프로세스가 토픽에 리더 균형을 유지해줌 (leader.imbalance.check.interval.seconds 설정을 통해 주기를 설정 가능)          | O         | true  |
| delete.topic.enable               | 토픽 삭제 방지                                                                                         | O         | false |

### 파티션 수 결정학기
* 쓰기 처리량을 고려 ( MSK 경우 인스턴스 별 쓰기 처리량이 정해져 있음 )
* 읽기 처리량을 고려 ( MSK 경우 인스턴스 별 읽기 처리량이 정해져 있음 )
* 키값을 이용한다면 파티션 추가시 많은 고민이 필요 처음부터 어느정도 예측값을 통해 처리량을 계산이 필요.
* [MSK IOPS 문서](https://docs.aws.amazon.com/msk/latest/developerguide/msk-provision-throughput.html)

### 주요 토픽 옵션

| 파라미터                       | 용도                                                                            | msk 지원 여부 | 기본값 |
|----------------------------|-------------------------------------------------------------------------------|-----------|-----|
| num.partitions             | 파티션 수 - 처리량에 따라 설정이 필요 너무 많은 경우( IOPS, 파티션 크기 ) 너무 적은 경우( 처리량 )를 고려해 작업이 필요   | O        | 1   |
| default.replication.factor | min.insync.replicas 값보다 최소 1이상 크게 잡아줄것을 권장                                    | O         | 2   |
| log.retention.ms           | 로그 보전 기간 세그먼트의 마지막 시간 이후 7일                                                   | O         | 7일  |
| log.retention.bytes        | 로그 보전 용량 (1og.retention.ms && 1og.retention.bytes 둘다 사용하는 경우 하나의 조건만 성립되면 삭제) | O         | -1  |
| log.segment.bytes          | 로그 세그먼트 용량 설정한 용량에 따라 세그먼트 파일이 생성 된다                                          | O         | 1G  |
| in.sync.replicas           | 최소 복제 단위 복제 단위가 클수록 성능에 영향을 미친다.                                              | O         | 2   |
| message.max.bytes          | 최소 메세지 크기 fetch.message.max.bytes 같은 값을 설정해야함.                                | O         | 1MB |

### 하드웨어
* 디스크 용량
  * 저장공간을 고려 최소 10프로 이상 디스를 확보해야 오버해드를 줄일 수 있음.
* 메모리
  * 카프카를 독립적으로 운영이 필수 페이지 케시를 공유하면 성능상 떨어질수 있음.
* 네트워크
  * 10GB 이상 NIC 선택 권장
* CPU
  * 디스크와 메모리 만큼 중요하지 않지만 상황에 맞게 선택이 필요.

클루우드 제품을 사용하므로 설정할수 있는 부분이 제한적이라 패스 ~~~         

### 클라우드
* 상환에 맞는 사양을 선택할 필요가 있음.

### 클러스터 설정하기
* 디스크 용량
* 브로커당 레플리카 용량
* CPU 용량
* 네트워크 용량
                        
### 운영체제 튜닝하기
* 가상 메모리
* 디스크
* 네트워킹

### 프로덕션 환경에서의 고려사항
* 가비지 수집기 옵션

### 데이터 센터 레이아웃

### 주키퍼 공유하기
* 브로커, 토픽, 파티션, 메타데이터를 저장하기 위해 사용됨
* 3.3.0 버전이 부터는 Kraft (래프트 기반에 컨트롤러 사용)

클루우드 제품을 사용하므로 설정할수 있는 부분이 제한적이라 패스 ~~~
