## 카프카 프로듀서

### 카프카 프로듀서 개괄

![img.png](img.png)

* 프로듀서가 브로커에 메세지 전송 과정
    * 키와 파티션은 선택사항
    * 키와 값을 직렬화 후 바이트 배열 변환.
    * 파티션을 지정하지 않은경우 파티셔너 통해 파티션이 결정되고 레코드 배치에 추가.
    * 별도의 쓰레드가 카프카 브로커에게 전송.
    * 브로커는 메세지 응답을 돌려준다 성공의 경우 RecordMetadata 정보를 리턴한다.(토픽,파티션,레코드 오프셋) 에러인 경우 에러를 리턴한다.
    * 에러를 받은 경우 몇번 더 재전송을 시도 할수 있다.

~~~java
  Properties kafkaProps=new Properties();
  kafkaProps.put("bootstrap.servers","broker1:9092,broker2:9092");
  kafkaProps.put("key.serializer" , "org.apache.kafka.common.serialization.StringSerializer");
  kafkaProps.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
  KafkaProducer<String, String> producer.=new KafkaProducer<String, String>(kafkaProps);
~~~

## 카프카로 메세지 전달하기
* 보내고 잊어 버리기.
  * 성능 상으로 가장 우수함.
  * 에러를 어느 정도 허용하는 경우 사용가능.

~~~java
    ProducerRecord<String, String> record = newProducerRecord◇("CustomerCountry", "PrecisiDnProducts", "France"); 
    try {
        producer.send(record);
    }catch (Exception e) {
        e.printstackTrace();
    }
~~~

* 동기적으로 메세지 전송하기
  * RecordMetadata 객체를 통해 메타정보를 알수 있다.
  * 실제로는 잘 사용되지 않는 패턴 성능상으로 좋지 않음. 

~~~java
    ProducerRecord<String, String> record = newProducerRecord◇("CustomerCountry", "PrecisiDnProducts", "France"); 
    try {
        producer.send(record).get();
    }catch (Exception e) {
        e.printstackTrace();
    }
~~~

* 비동기적으로 메세지 전송하기
  * RecordMetadata 객체를 통해 메타정보를 알수 있다.
  * 동일한 메인 쓰레드에서 콜백을 처리함.
  * 콜백에 무거운 로직을 넣는 것은 좋지 않음.
  * 무거운 로직을 넣는경우 별도의 쓰레드 사용 필요.

~~~java
private class DemoProducerCallback implements Callback { 
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) { 
        if (e != null) {
            e.printstackTrace();
        }
    }
}

ProducerRecord<String, String> record = new ProducerRecord<>("CustomerCountry", "BiomedicalMaterials", "USA");
producer.send(record, new DemoProducerCallback());
~~~

## 프로듀서 설정하기
* acks 설정에 따라 성능과 안전성 트레이드 오프함.

| 파라미터               | 용도                                                                                 | msk 지원 여부 | 기본값      |                   
|--------------------|------------------------------------------------------------------------------------|-----------|----------|                   
| client.id          | 로그를 확인 확인 하기 위해 필요.                                                                | O        | 임의적으로 배정 |                   
| acks               | 0(브로커 응답을 확인하지 않음.) 1(단일 브로커 응답을 확인함.) -1(in-sync replica 설정된 만큼의 모든 브로커로 응답을 받음.) | O         | 1        |


### 메세지 전달 시간

![img_1.png](img_1.png)
