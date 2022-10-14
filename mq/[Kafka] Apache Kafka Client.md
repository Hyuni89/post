# Consumer

### Properties

카프카 설정을 properties로 설정한다.

```kotlin
val properties = Properties().apply {
	put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID)
	put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, KAFKA_CLUSTER)
	put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer::class.java)
	put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer::class.java)
	put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
	put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false")
	put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "1000")
}
```

##### `KEY_DESERIALIZER_CLASS_CONFIG` (key.deserializer, class)

kafka에서 받아온 message의 key를 역직렬화하는 클래스. `org.apache.kafka.common.serialization.Deserializer` interface의 하위 클래스

##### `VALUE_DESERIALIZER_CLASS_CONFIG` (value.deserializer, class)

value를 역직렬화하는 클래스. key와 동일한 interface의 하위 클래스

##### `BOOTSTRAP_SERVERS_CONFIG` (bootstrap.servers, list)

kafka cluster 리스트. 초기 connection을 맺는데 사용되며 host:port로 구성

##### `GROUP_ID_CONFIG` (group.id, string)

consuming group을 설정

##### `AUTO_OFFSET_RESET_CONFIG` (auto.offset.reset, string)

offset 값이 없거나 설정되어 있지 않으면 적용되는 설정. 가장 초기 offset으로 이동하는 `earliest`, 가장 마지막 offset으로 이동하는 `latest`, 예외를 던지는 `none` 옵션이 있다.  
default `latest`

##### `MAX_POLL_RECORDS_CONFIG` (max.poll.records, int)

한번에 `poll`로 가져올 수 있는 최대 record 값. default 500

##### `ENABLE_AUTO_COMMIT_CONFIG` (enable.auto.commit, boolean)

offset을 background에서 주기적으로 commit 하는 설정. default true

### Consumer

```kotlin
val consumer = KafkaConsumer<String, String>(properties)
consumer.subscribe(listOf("kafka_topic"))

while (true) {
	val record = consumer.poll(Duration.ofMillis(500))
	for (r in record) {
		println("topic: ${r.topic()}, partition: ${r.partition()}, offset: ${r.offset()}, key: ${r.key()}, value: ${r.value()}")
	}
}

consumer.close()
```

`subscribe`을 사용해서 kafka topic을 구독할 수 있다. 만약 kafka partition이 여러개라면 생성한 `consumer`는 알아서 partition을 돌아가며 record를 가져온다. 각 partition마다 offset은 다를 수 있다.

```kotlin
val topicPartition = TopicPartition("topic", 0) // partition number
consumer.assign(listOf(topicPartition))
```

`assign`을 사용하면 원하는 partition에만 붙어서 record를 가져올 수 있다.

```kotlin
val offset = consumer.offsetsForTimes(mapOf(topicPartition to 1656167334L)) // timestamp
offset[topicPartition]?.offset()?.let { consumer.seek(topicPartition, it) }
```

`offsetsForTimes`를 이용해 특정 날짜에 대한 offset 값을 가져올 수 있고, `seek`을 통해 해당 offset으로 이동할 수 있다.

# Producer

### Properties

consumer와 동일하게 properties로 설정한다.

```kotlin
val properties = Properties().apply {
	put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka_cluster")
	put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer::class.java)
	put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer::class.java)
	put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy")
}
```

##### `KEY_SERIALIZER_CLASS_CONFIG` (key.serializer, class)

key를 직렬화 하는 class. `org.apache.kafka.common.serialization.Serializer` interface의 하위 클래스

##### `VALUE_SERIALIZER_CLASS_CONFIG` (value.serializer, class)

value를 직렬화 하는 class. key와 동일한 interface의 하위 클래스

##### `BOOTSTRAP_SERVERS_CONFIG` (bootstrap.servers, list)

kafka cluster 리스트. 초기 connection을 맺는데 사용되며 host:port로 구성

##### `COMPRESSION_TYPE_CONFIG` (comperssion.type, string)

압축 옵션. default none

### Producer

```kotlin
val producer = KafkaProducer<String, String>(properties)

while (true) {
	val record = ProducerRecord<String, String>("kafka_topic", "message")
	producer.send(record) { meta: RecordMetadata, e: Exception ->
		println("partition: ${meta.partition()}, offset: ${meta.offset()}, meta: ${meta.toString()}")
	}
}

producer.close()
```