# Producer에 Schema 적용하기

## 목차

1. Maven repositoy 설정
2. Dependency 추가
3. Subject에 schema 등록
4. 개발하기

---

## 1. Maven repositoy 설정

```java
repositories {
	mavenCentral()
	maven {
		url "<https://packages.confluent.io/maven>"
	}
}
```

## 2. Dependency 추가

```java
implementation 'io.confluent:kafka-schema-registry-client:7.0.0'
implementation 'io.confluent:kafka-avro-serializer:7.0.0'
```

## 3. Subject에 schema 등록

```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \\
http://[host]/subjects/[subject_name]/versions \\
--data '{"schema": "[schema]"}'
```

## 4. 개발하기

### application.properties

```yaml
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=io.confluent.kafka.serializers.KafkaAvroSerializer

spring.kafka.topic.name=topic

spring.kafka.bootstrap-servers=[bootstrap server]
spring.kafka.schema.registry.endpoint=[endpoint]
```

### Producer class

```java
@Component
@RequiredArgsConstructor
public class Producer {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServer;
		@Value("${spring.kafka.schema.registry.endpoint}")
    private String schemaRegistryServer;

		@Value("${spring.kafka.producer.key-serializer}")
    private String keySerializer;
    @Value("${spring.kafka.producer.value-serializer}")
    private String valueSerializer;

    @Value("${spring.kafka.topic.name}")
    private String topicName;
		private final String subjectName = "subject";

		private final Parser parser = new Parser();
		private Schema schema;

    private SchemaRegistryClient schemaRegistryClient;
    private KafkaProducer<String, String> producer;

		public Producer(KafkaProducer<String, String> producer, SchemaRegistryClient schemaRegistryClient) {
				Properties properties = new Properties();

				// 하나의 토픽에 여러개의 스키마를 사용
				properties.setProperty(KafkaAvroSerializerConfig.VALUE_SUBJECT_NAME_STRATEGY, TopicRecordNameStrategy.class.getName());
        properties.setProperty(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, schemaRegistryServer);
        properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServer);
        properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, keySerializer);
        properties.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, valueSerializer);

        producer = new KafkaProducer<>(properties);

				int schemaVersion = -1;

				StringBuilder builder = new StringBuilder().append(Producer.class.getName());
				try {
            schemaVersion = schemaRegistryClient.getLatestSchemaMetadata(subjectName).getVersion();
            builder.append("\\n")
                    .append("Schema Version: ").append(schemaVersion)

            log.info(builder.toString());
        } catch (IOException | RestClientException e) {
            builder.append("\\n")
                    .append("\\nFail To Get Schema Version")

            log.error(builder.toString());
            throw new IllegalArgumentException("Failed To Get Schema Version");
				}
				schema = getSchema(subjectName, schemaVersion);
		}

		public Schema getSchema(String subject, int version) {
        String schemaString = schemaRegistryClient.getByVersion(subject, version, false).getSchema();
        return parser.parse(schemaString);
    }

    public void send(Message message) {
        GenericRecord avroRecord = new GenericData.Record(schema);

				avroRecord.put("name", message.getName());
        avroRecord.put("age", message.getAge());

        log.info("produce: " + avroRecord);
        producer.send(new ProducerRecord<>(topicName, avroRecord));
    }
```
