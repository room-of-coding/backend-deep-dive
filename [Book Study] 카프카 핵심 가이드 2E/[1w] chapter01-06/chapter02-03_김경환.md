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
    /usr/local/kafka/bin/kafka-topics.sh --bootstrap-serverlocalhost:9092 --create --replication-factor 1 --partitions 1 --topic test
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

파라미터 | 용도 | msk 지원 여부 | 기본값
--------- | --------- | --------- | ---------
log.dirs | 별도의 로그 세그먼트 저장 디렉토리 지정 | X | false
num.recovery.threads.per.data.dir | 브로커가 시작과 종료시에만 활성화 되기 때문에 최대한 많이 할당하는게 좋음 (num.recovery.threads.per.data.dir * 1og.dirs) 만큼 할당됨 | X | empty
auto.create.topics.enable | 프로듀서가 토픽 쓸때, 컨슈머가 토픽을 읽을 때, 클라이언트가 토픽을 메타를 요청할때 생성 | O | false
auto.leader.rebalance.enable | 백그라운드 프로세스가 토픽에 리더 균형을 유지해줌 (leader.imbalance.check.interval.seconds 설정을 통해 주기를 설정 가능)| O | true
delete.topic.enable | 토픽 삭제 방지 | O | false

### 주요 토픽 옵션

파라미터 | 용도 | msk 지원 여부 | 기본값
--------- | --------- | --------- | ---------
num.partitions | 파티션 수 - 처리량에 따라 설정이 필요 너무 많은 경우( IOPS, 크기 ) 너무 적은 경우(처리량)를 고려해 작업이 필요 | O | 1 

