# Kafka / Event Streaming

Entur uses **Apache Kafka** hosted on **Aiven** for event streaming and asynchronous messaging between services. The standard library is **`entur-kafka-spring-starter`** (`org.entur.data:entur-kafka-spring-starter`), which provides Spring Boot autoconfiguration with sensible defaults.

For advanced Kafka topics (error handling strategies, operational patterns), see the [Data Handbook](https://enturas.atlassian.net/wiki/spaces/TD/pages/4962451517/2+-+Data+Streaming+Kafka).

## When to Use Kafka vs REST

| Pattern | Use |
|---------|-----|
| **Kafka** | Event-driven communication, fan-out to multiple consumers, audit trails, high-throughput data pipelines, decoupled producers/consumers |
| **REST** | Synchronous request-response, CRUD operations, client-facing APIs |

## Infrastructure

### Aiven Clusters

Entur operates separate Kafka clusters per environment, each with internal (VPC-peered) and public endpoints:

| Cluster Enum | Use When | Environment |
|-------------|----------|-------------|
| `AIVEN_TEST_INT` | App runs in `kub-ent-tst` (inside VPC) | Test |
| `AIVEN_PUBLIC_TEST_INT` | App runs locally or outside VPC | Test |
| `AIVEN_TEST_EXT` | External partner access | Test |
| `AIVEN_PROD_INT` | App runs in `kub-ent-prd` (inside VPC) | Production |
| `AIVEN_PUBLIC_PROD_INT` | App runs locally or outside VPC | Production |
| `AIVEN_PROD_EXT` | External partner access | Production |

**Rule**: Use `*_INT` clusters for apps running in the corresponding Kubernetes environment. Use `*_PUBLIC_*_INT` for local development or apps running outside the VPC. Use `*_EXT` clusters only for external partner integrations.

### Authentication

All clusters use **SASL/SCRAM-SHA-512** authentication over TLS (`SASL_SSL` security protocol). Credentials (username/password) are provisioned per service and must be stored in **Google Secret Manager**, referenced via ExternalSecrets in Helm -- never hardcode them.

### Schema Registry

Each cluster has an associated **Confluent Schema Registry** for Avro and Protobuf schema management. The schema registry URL is automatically resolved from the cluster enum. Authentication uses the same SASL credentials as basic auth.

## Dependency Setup

### Gradle

```kotlin
plugins {
    // Required if using Avro schemas (generates Java/Kotlin classes from .avsc files)
    id("com.github.davidmc24.gradle.plugin.avro") version "1.9.1"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.entur.data:entur-kafka-spring-starter:<version>")  // check Artifactory for latest
}
```

The starter is published to Entur's JFrog Artifactory. Check [Artifactory](https://entur2.jfrog.io) for the latest version. See [java.md](java.md#artifactory-jfrog) for repository configuration.

## Configuration

All configuration is under the `entur.kafka` prefix. The starter autoconfigures both producer and consumer beans.

### Minimal Configuration

```yaml
entur:
  kafka:
    kafkaCluster: "AIVEN_TEST_INT"
    sasl:
      username: ${KAFKA_USERNAME}
      password: ${KAFKA_PASSWORD}
    consumer:
      group: "my-application"
```

### Per-Environment Configuration

Use Spring profiles to select the correct cluster per environment:

```yaml
# application.yml (shared)
entur:
  kafka:
    sasl:
      username: ${KAFKA_USERNAME}
      password: ${KAFKA_PASSWORD}
    consumer:
      group: "my-application"

---
# application-dev.yml / application-tst.yml
entur:
  kafka:
    kafkaCluster: "AIVEN_TEST_INT"

---
# application-prd.yml
entur:
  kafka:
    kafkaCluster: "AIVEN_PROD_INT"

---
# application-local.yml (local development)
entur:
  kafka:
    kafkaCluster: "AIVEN_PUBLIC_TEST_INT"
```

### Serialization Defaults

| Setting | Default | Alternatives |
|---------|---------|-------------|
| Key serializer | `StringSerializer` | `KafkaAvroSerializer` for Avro-typed keys |
| Key deserializer | `StringDeserializer` | `KafkaAvroDeserializer` for Avro-typed keys |
| Value serializer | `KafkaAvroSerializer` | `KafkaProtobufSerializer` for Protobuf |
| Value deserializer | `KafkaAvroDeserializer` | `KafkaProtobufDeserializer` for Protobuf |
| Specific Avro reader | `true` | Set `false` for `GenericRecord` |

### Separate Producer/Consumer Clusters

If you need to produce to a different cluster than you consume from (e.g., cross-environment replication):

```yaml
entur:
  kafka:
    kafkaCluster: "AIVEN_TEST_INT"          # consumer cluster
    producerCluster: "AIVEN_PROD_INT"       # producer cluster (overrides for producers)
    sasl:
      username: ${KAFKA_CONSUMER_USERNAME}
      password: ${KAFKA_CONSUMER_PASSWORD}
      producerUsername: ${KAFKA_PRODUCER_USERNAME}   # separate producer credentials
      producerPassword: ${KAFKA_PRODUCER_PASSWORD}
```

### Toggling Producer/Consumer Beans

```yaml
entur:
  kafka:
    consumer:
      enabled: false   # disable consumer beans (e.g., producer-only service)
    producer:
      enabled: false   # disable producer beans (e.g., consumer-only service)
```

Note: DLT functionality requires the producer to be enabled (it produces to DLT topics).

## Producing Messages

### Standard Producer (Avro, String Keys)

The most common pattern -- Avro-serialized values with string keys:

```kotlin
@Component
class OrderEventProducer(
    private val producer: EnturKafkaProducer<OrderEvent>
) {
    fun publishOrderCreated(order: OrderEvent) {
        producer.send(
            "order-events",              // topic
            order.orderId.toString(),    // key (determines partition)
            order,                       // value (Avro SpecificRecord)
            correlationId(),             // correlation ID (auto-added as header)
            { result -> log.info("Sent order event: {}", result.recordMetadata) },
            { error -> log.error("Failed to send order event", error) },
            listOf(RecordHeader("event-type", "ORDER_CREATED".toByteArray()))  // optional custom headers
        )
    }
}
```

The `correlationId` parameter is added as an `X-Correlation-Id` header automatically.

### Avro-Keyed Producer

For Avro-typed keys (both key and value are `SpecificRecordBase`):

```kotlin
@Component
class MyProducer(
    private val producer: EnturAvroKeyKafkaProducer<MyKeyType, MyEventType>
) {
    fun send(key: MyKeyType, event: MyEventType) {
        producer.send("my-topic", key, event, correlationId())
    }
}
```

Remember to configure the key serializer/deserializer:

```yaml
entur:
  kafka:
    keySerializer: "io.confluent.kafka.serializers.KafkaAvroSerializer"
    keyDeserializer: "io.confluent.kafka.serializers.KafkaAvroDeserializer"
```

### Protobuf Producer

For Protocol Buffers values:

```kotlin
@Component
class ProtobufProducer(
    private val producer: EnturProtobufKafkaProducer<String, MyProtoMessage>
) {
    fun send(message: MyProtoMessage) {
        producer.send("proto-topic", "my-key", message, correlationId())
    }
}
```

Configure the value serializer:

```yaml
entur:
  kafka:
    valueSerializer: "io.confluent.kafka.serializers.protobuf.KafkaProtobufSerializer"
```

### Generic Producer

For schemaless topics (legacy use only):

```kotlin
@Component
class GenericProducer(
    private val producer: EnturGenericKafkaProducer<String, String>
) {
    fun send(key: String, value: String) {
        producer.send("legacy-topic", key, value, correlationId())
    }
}
```

### Producer Tuning Options

```yaml
entur:
  kafka:
    producer:
      acks: "all"                    # all replicas must acknowledge (required for idempotence)
      retries: 2147483647            # max int -- use timeout to control retry duration
      enableIdempotence: true        # exactly-once per partition
      maxInFlightRequests: 5         # max unacknowledged requests (<=5 required for idempotence)
      deliveryTimeoutMs: 120000      # max time for send() to complete
      compressionType: "lz4"         # none, gzip, snappy, lz4, zstd
```

### Transactional Producer

Enable Kafka transactions (exactly-once semantics across partitions):

```yaml
entur:
  kafka:
    producer:
      transactionIdPrefix: "my-app-tx-"   # enables transactions, implicitly sets idempotence
      allowNonTransactional: true          # allow producing outside @Transactional (default true)
```

Use `@Transactional` on methods that produce messages:

```kotlin
@Transactional
fun processAndPublish(event: MyEvent) {
    producer.send("topic-a", event.id, event, correlationId())
    producer.send("topic-b", event.id, event.summary, correlationId())
}
```

## Consuming Messages

### Standard Consumer (Avro)

```kotlin
@Component
class OrderEventListener {

    @KafkaListener(topics = ["order-events"], containerFactory = "enturListenerFactory")
    fun onOrderEvent(
        @Header(KafkaHeaders.RECEIVED_KEY) key: String,
        @Payload event: OrderEvent
    ) {
        processOrder(event)
    }
}
```

**Important**: Always use `containerFactory = "enturListenerFactory"` -- this is the factory bean provided by the starter with all Entur defaults applied.

### Avro-Keyed Consumer

```kotlin
@KafkaListener(topics = ["my-topic"], containerFactory = "enturListenerFactory")
fun onEvent(
    @Header(KafkaHeaders.RECEIVED_KEY) key: MyKeyType,
    @Payload event: MyEventType
) {
    process(key, event)
}
```

### Protobuf Consumer

```kotlin
@KafkaListener(topics = ["proto-topic"], containerFactory = "enturSpecificProtobufConsumerFactory")
fun onProtoEvent(
    @Header(KafkaHeaders.RECEIVED_KEY) key: String,
    @Payload message: MyProtoMessage
) {
    process(message)
}
```

Note the different container factory: `enturSpecificProtobufConsumerFactory`.

Configure the value deserializer and specific message type:

```yaml
entur:
  kafka:
    valueDeserializer: "io.confluent.kafka.serializers.protobuf.KafkaProtobufDeserializer"
    consumer:
      specificProtobufMessageValue: "com.example.MyProtoMessage"
```

### GenericRecord Consumer

For schemaless or dynamically typed topics:

```kotlin
@KafkaListener(topics = ["generic-topic"], containerFactory = "enturListenerFactory")
fun onGenericEvent(@Payload message: ConsumerRecord<Any, Any>) {
    val value = message.value()
    process(value)
}
```

Set `useSpecificAvro: false` for GenericRecord consumption:

```yaml
entur:
  kafka:
    consumer:
      useSpecificAvro: false
```

### Consumer Tuning Options

```yaml
entur:
  kafka:
    consumer:
      group: "my-application"         # consumer group ID (required for group coordination)
      offsetReset: "latest"           # "latest" or "earliest" for new consumer groups
      sessionTimeoutMs: 15000         # heartbeat timeout (recommended: 15000-300000)
      maxPollIntervalMs: 300000       # max time between polls before considered failed
      maxPollRecords: 500             # records per poll batch
      enableAutoCommit: true          # automatic offset commits
```

### Standalone Consumer (No Consumer Group)

For consumers that must read all partitions independently (no group coordination):

```kotlin
@KafkaListener(
    topicPartitions = [TopicPartition(
        topic = "my-topic",
        partitions = ["#{@partitionFinder.allPartitions(\"my-topic\")}"]
    )]
)
fun onEvent(
    @Header(KafkaHeaders.RECEIVED_KEY) key: String,
    @Payload message: MyEvent
) {
    process(message)
}
```

Required configuration for standalone consumers:

```yaml
entur:
  kafka:
    consumer:
      enabled: true
      group:                          # leave null -- no consumer group
      offsetReset: "earliest"         # or "latest"
      enableAutoCommit: false         # no offsets to commit without a group

spring:
  kafka:
    listener:
      ack-mode: "manual"             # disable Spring Kafka's built-in ack handling
```

## Error Handling and Retry

### Non-Blocking Retry with DLT

The starter supports **non-blocking retry** using separate retry topics with exponential backoff. Failed messages are moved to retry topics, allowing the consumer to continue processing other messages.

```yaml
entur:
  kafka:
    retry:
      enabled: true
      maxAttempts: 3                       # total attempts including original
      initialInterval: 5000                # 5s initial delay
      intervalMultiplier: 5.0              # exponential backoff multiplier
      maxInterval: 125000                  # max 125s between retries
      retryTopics:                         # empty = all topics; or list specific topics
        - "order-events"
      retryTopicsPrefix: ""                # prefix for retry/DLT topic names
      useSamePartition: false              # let Kafka choose partition on retry topics
```

**Retry topic naming**: For a topic `order-events`, retry topics are named `order-events-retry-0`, `order-events-retry-1`, etc., with a final DLT topic `order-events-dlt`.

### Blocking Retry for Transient Errors

For errors where all messages would fail (e.g., a downstream service being down), use blocking retries to pause consumption rather than flooding retry topics:

```yaml
entur:
  kafka:
    retry:
      enabled: true
      blockingRetryExceptions:
        - "org.springframework.web.client.ResourceAccessException"
      blockingInterval: 1000               # 1s initial blocking delay
      blockingIntervalMultiplier: 5.0
      maxBlockingInterval: 125000
```

### Fatal Exceptions (Skip to DLT)

For exceptions that should never be retried:

```yaml
entur:
  kafka:
    retry:
      fatalExceptions:
        - "com.fasterxml.jackson.core.JsonParseException"
        - "org.entur.myapp.InvalidOrderException"
```

### DLT Handler

To process messages that land in the DLT:

```kotlin
@Component
class OrderDltHandler {
    fun handleDlt(message: OrderEvent) {
        log.error("Order event failed all retries: {}", message.orderId)
        alertOpsTeam(message)
    }
}
```

Configure in YAML:

```yaml
entur:
  kafka:
    retry:
      dltHandlerBean: "orderDltHandler"
      dltHandlerMethod: "handleDlt"
```

### Manual Retry/DLT with Annotations

If the starter's retry configuration is insufficient, use Spring Kafka annotations directly (set `entur.kafka.retry.enabled: false`):

```kotlin
@RetryableTopic(kafkaTemplate = "enturKafkaTemplate")
@KafkaListener(topics = ["my-topic"], containerFactory = "enturListenerFactory")
fun onEvent(
    @Header(KafkaHeaders.RECEIVED_KEY) key: String,
    @Payload event: MyEvent
) {
    processEvent(event)
}

@DltHandler
fun onDlt(@Payload event: MyEvent) {
    handleDeadLetter(event)
}
```

### Custom Retry Exception Logging

Override the default retry exhaustion logging:

```kotlin
@Bean
fun customRetryExceptionLogger() = CustomRetryExceptionLogger { exception, consumerRecord, nextDestination ->
    if (nextDestination.isDltTopic) {
        log.error("Message processing failed after all retries, sending to DLT", exception)
    }
}
```

### Handling Deserialization Errors

Messages that fail to deserialize never reach the listener. To route these to the DLT, configure the list of expected Avro classes:

```yaml
entur:
  kafka:
    avroSerializableClasses:
      - "org.entur.myapp.OrderEvent"
      - "org.entur.myapp.PaymentEvent"
```

This enables a `DelegatingByTypeSerializer` that can publish the raw bytes to the DLT. **Keep this list up to date** -- missing types will cause the error handler to fail.

### Custom Error Handler

For fully custom error handling, provide a bean named `enturCustomErrorHandler`:

```kotlin
@Bean(name = ["enturCustomErrorHandler"])
fun enturCustomErrorHandler(): CommonErrorHandler =
    DefaultErrorHandler(
        DeadLetterPublishingRecoverer(enturKafkaTemplate()) { record, _ ->
            TopicPartition("${record.topic()}-dlt", -1)
        },
        FixedBackOff(1000L, 2L)
    )
```

Note: This overrides all default error handling including retry topic naming.

## Avro Schema Management

### Generating Classes from Schemas

Use the Gradle Avro plugin to generate Java/Kotlin classes from `.avsc` files:

```kotlin
plugins {
    id("com.github.davidmc24.gradle.plugin.avro") version "1.9.1"
}
```

Place `.avsc` schema files in `src/main/avro/`. The plugin generates classes during compilation.

### Schema Registry

The starter automatically configures the Confluent Schema Registry client based on the selected cluster. Schema compatibility is enforced at the registry level -- schemas must be backward-compatible by default.

## Testing

### Unit/Integration Test Configuration

For tests using embedded Kafka or Testcontainers:

```yaml
# application-test.yml
entur:
  kafka:
    bootstrapServer: "localhost:9092"
    schemaRegistryUrl: "mock://testing"
    securityProtocol: "PLAINTEXT"
    sasl:
      mechanism: "PLAIN"
      username: "test"
      password: "test"
    consumer:
      group: "test-group"
```

Override `bootstrapServer` and `schemaRegistryUrl` directly for test environments -- this takes precedence over `kafkaCluster`.

### Testcontainers with Kafka

```kotlin
@SpringBootTest
@Testcontainers
class KafkaIntegrationTest {

    companion object {
        @Container
        val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))

        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("entur.kafka.bootstrapServer") { kafka.bootstrapServers }
            registry.add("entur.kafka.schemaRegistryUrl") { "mock://testing" }
            registry.add("entur.kafka.securityProtocol") { "PLAINTEXT" }
        }
    }
}
```

## Redis as Kafka State Store

Redis (Memorystore) is commonly paired with Kafka consumers for deduplication, state caching, and idempotent processing. For Redis infrastructure setup, see [terraform/modules.md](terraform/modules.md#memorystore-redis). For general Redis usage patterns, see [java.md](java.md#redis-memorystore) or [go.md](go.md#redis-memorystore).

### Idempotent Consumer (Deduplication)

Kafka provides at-least-once delivery, meaning consumers may receive the same message more than once. Use Redis `SET NX EX` to track processed message IDs and skip duplicates:

```kotlin
@Component
class OrderEventListener(
    private val redis: StringRedisTemplate,
    private val orderService: OrderService,
) {
    @KafkaListener(topics = ["order-events"], containerFactory = "enturListenerFactory")
    fun onOrderEvent(
        @Header(KafkaHeaders.RECEIVED_KEY) key: String,
        @Header("event-id") eventId: String,
        @Payload event: OrderEvent,
    ) {
        val dedupKey = "myapp:dedup:$eventId"

        // SET NX -- returns true only if key was newly created
        val isNew = redis.opsForValue()
            .setIfAbsent(dedupKey, "1", Duration.ofHours(24)) ?: false

        if (!isNew) {
            log.info("Duplicate event skipped: {}", eventId)
            return
        }

        orderService.processOrder(event)
    }
}
```

**TTL guidance**: Set the deduplication key TTL to at least the maximum expected redelivery window. 24 hours is a safe default. If your retry/DLT configuration retries for at most 2 hours, a 4-hour TTL is sufficient.

### Consumer State Cache

When a consumer needs to enrich events with reference data, use Redis to cache lookups and avoid repeated database queries:

```kotlin
@Component
class EnrichmentListener(
    private val redis: StringRedisTemplate,
    private val productRepository: ProductRepository,
    private val objectMapper: ObjectMapper,
) {
    @KafkaListener(topics = ["raw-events"], containerFactory = "enturListenerFactory")
    fun onEvent(@Payload event: RawEvent) {
        val product = getCachedProduct(event.productId)
        val enriched = event.enrich(product)
        // ... produce enriched event or persist
    }

    private fun getCachedProduct(productId: String): Product {
        val key = "myapp:product:$productId"
        val cached = redis.opsForValue().get(key)
        if (cached != null) {
            return objectMapper.readValue(cached, Product::class.java)
        }

        val product = productRepository.findById(productId)
            ?: throw IllegalStateException("Product not found: $productId")

        redis.opsForValue().set(key, objectMapper.writeValueAsString(product), Duration.ofMinutes(30))
        return product
    }
}
```

### Best Practices for Redis + Kafka

- **Always use TTLs** on deduplication and cache keys -- Kafka consumer patterns can generate large volumes of keys.
- **Handle Redis failures gracefully** -- if Redis is down, either process the message (risking duplicates) or throw to trigger Kafka retry. Choose based on your idempotency requirements.
- **Use the event ID or Kafka offset as the dedup key** -- `{topic}:{partition}:{offset}` is naturally unique.
- **Do not use Redis as a Kafka replacement** -- Redis Pub/Sub has no persistence, no consumer groups, and no delivery guarantees.
- **Namespace keys** with the application name to avoid collisions: `myapp:dedup:`, `myapp:cache:`.

## Observability

The starter automatically registers **Micrometer/Prometheus** listeners on both producer and consumer factories when a `MeterRegistry` bean is present. This exposes standard Kafka client metrics (e.g., `kafka_producer_*`, `kafka_consumer_*`) to your Prometheus endpoint without additional configuration.

For additional Kafka consumer metrics (processing time via `@Timed` and consumption delay tracking), use the constants from the Entur metrics starter. See [observability.md](observability.md#kafka-consumer-metrics) for patterns and standard metric names.

## Configuration Reference

### Full Configuration Tree

```yaml
entur:
  kafka:
    kafkaCluster: "AIVEN_TEST_INT"              # cluster enum (sets bootstrap + schema registry)
    producerCluster:                             # override cluster for producers only
    bootstrapServer:                             # override bootstrap server (takes precedence over cluster)
    producerBootstrapServer:                     # override bootstrap server for producers only
    schemaRegistryUrl:                           # override schema registry URL
    producerSchemaRegistryUrl:                   # override schema registry URL for producers only
    schemaRegistryBasicAuth:                     # custom schema registry basic auth (username:password)
    securityProtocol: "SASL_SSL"                 # SASL_SSL (default) or PLAINTEXT (testing)
    keySerializer: "...StringSerializer"         # key serializer class
    keyDeserializer: "...StringDeserializer"     # key deserializer class
    valueSerializer: "...KafkaAvroSerializer"    # value serializer class
    valueDeserializer: "...KafkaAvroDeserializer" # value deserializer class
    avroSerializableClasses: []                  # explicit class list for deserialization error handling

    sasl:
      mechanism: "SCRAM-SHA-512"                 # SASL mechanism
      module: "...ScramLoginModule"              # JAAS login module
      username:                                  # SASL username
      password:                                  # SASL password
      producerUsername:                           # separate producer username
      producerPassword:                           # separate producer password

    consumer:
      enabled: true
      group:                                     # consumer group ID (null for standalone)
      useSpecificAvro: true                      # true for SpecificRecord, false for GenericRecord
      offsetReset:                               # "latest" or "earliest"
      sessionTimeoutMs:                          # heartbeat timeout (ms)
      maxPollIntervalMs:                         # max time between polls (ms)
      maxPollRecords:                            # records per poll batch
      enableAutoCommit:                          # automatic offset commits
      specificProtobufMessageValue:              # protobuf message class name

    producer:
      enabled: true
      acks:                                      # "0", "1", or "all"
      retries:                                   # number of retries
      enableIdempotence:                         # exactly-once per partition
      transactionIdPrefix:                       # enables transactions
      allowNonTransactional: true                # allow non-transactional sends when transactions enabled
      maxInFlightRequests:                       # max unacknowledged requests
      deliveryTimeoutMs:                         # max send timeout (ms)
      compressionType:                           # none, gzip, snappy, lz4, zstd

    retry:
      enabled: false
      retryTopics: []                            # empty = all topics
      retryTopicsPrefix:                         # prefix for retry topic names
      useSamePartition: false                    # maintain partition on retry
      initialInterval: 5000                      # initial retry delay (ms)
      intervalMultiplier: 5.0                    # exponential backoff multiplier
      maxInterval: 125000                        # max retry delay (ms)
      maxAttempts: 3                             # total attempts including original
      blockingRetryExceptions: []                # exceptions for blocking retry
      blockingInterval: 1000                     # blocking retry initial delay (ms)
      blockingIntervalMultiplier: 5.0            # blocking retry backoff multiplier
      maxBlockingInterval: 125000                # max blocking retry delay (ms)
      fatalExceptions: []                        # exceptions that skip to DLT
      dltHandlerBean:                            # bean name for DLT handler
      dltHandlerMethod:                          # method name for DLT handler
```

## Key Beans Provided

| Bean Name | Type | Condition |
|-----------|------|-----------|
| `enturKafkaTemplate` | `KafkaTemplate<K, V>` | `producer.enabled=true` (default) |
| `kafkaProducer` | `EnturKafkaProducer<T>` | `producer.enabled=true` |
| `kafkaAvroKeyProducer` | `EnturAvroKeyKafkaProducer<K, V>` | `producer.enabled=true` |
| `kafkaGenericProducer` | `EnturGenericKafkaProducer<K, V>` | `producer.enabled=true` |
| `kafkaProtobufProducer` | `EnturProtobufKafkaProducer<K, V>` | `producer.enabled=true` |
| `enturListenerFactory` | `ConcurrentKafkaListenerContainerFactory` | `consumer.enabled=true` (default) |
| `enturSpecificProtobufConsumerFactory` | `ConcurrentKafkaListenerContainerFactory` | `consumer.enabled=true` |
| `enturConsumerFactory` | `ConsumerFactory<K, V>` | `consumer.enabled=true` |
| `partitionFinder` | `PartitionFinder<K, V>` | `consumer.enabled=true` |
| `kafkaTransactionManager` | `KafkaTransactionManager<K, V>` | `transactionIdPrefix` is set |
