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
