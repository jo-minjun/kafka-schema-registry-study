# Kafka Producer 개발하기

## 목차

1. Kafka 및 Zookeeper 설치
2. Topic 및 정책 설정
3. Dependency 추가
4. 개발하기

---

## 1. Kafka 및 Zookeeper 설치

나는 github에서 kafka 및 zookeeper docker-compose를 다운 받아서 설치했다.

https://github.com/conduktor/kafka-stack-docker-compose

## 2. Topic 및 정책 설정

```bash
/kafka-topics.sh --bootstrap-server [server] \\
--create --topic [topic] \\
--partitions [num of partitions] \\
--replication-factor [num of replica]
```

## 3. Dependency 추가

```java
implementation 'org.springframework.kafka:spring-kafka'
```

## 4. 개발하기

```java
@Component
@RequiredArgsConstructor
public class Producer {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServer;

    @Value("${spring.kafka.topic.name}")
    private String topicName;

    private KafkaProducer<String, String> producer;

		public Producer(KafkaProducer<String, String> producer) {
				Properties properties = new Properties();

        properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServer);
        properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        producer = new KafkaProducer<>(properties);
		}

    public void send(String message) {
        ProducerRecord<String, String> record = new ProducerRecord<>(topicName, message);

        producer.send(record);
    }
```
