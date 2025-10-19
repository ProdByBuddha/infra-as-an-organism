# Chapter 7: The Digestive System (Data Processing)

## Introduction: From Raw Data to Actionable Insights

Just as an organism's digestive system transforms food into energy and nutrients, your infrastructure's data processing layer transforms raw data into actionable insights. This chapter explores modern data pipeline architecture, from ingestion to transformation to storage, with production-ready implementations.

**What You'll Learn:**
- Data pipeline architecture patterns
- Stream processing with Apache Kafka
- Batch processing with Apache Airflow
- Data transformation and validation
- Real-time analytics implementation
- Data quality and monitoring
- Performance optimization strategies

**Real-World Scenario:**

Your e-commerce platform generates data from multiple sources:
- User clickstream events (millions per day)
- Transaction records
- Inventory updates
- Customer support tickets
- Application logs
- Third-party API responses

You need to:
1. Ingest data from all sources reliably
2. Transform and enrich the data
3. Validate data quality
4. Store processed data efficiently
5. Generate real-time analytics
6. Support batch analytics jobs

This chapter provides production-ready solutions for all these requirements.

---

## The Evolution of Data Processing

### Phase 1: Monolithic Databases (2000s)

```
┌─────────────────────────────────────┐
│         Application                 │
│  ┌────────────────────────────┐    │
│  │  Business Logic            │    │
│  │  + Data Processing         │    │
│  │  + Analytics               │    │
│  └──────────┬─────────────────┘    │
│             │                       │
│  ┌──────────▼─────────────────┐    │
│  │  Single Database           │    │
│  │  (PostgreSQL/MySQL)        │    │
│  └────────────────────────────┘    │
└─────────────────────────────────────┘

Problems:
❌ Scaling bottlenecks
❌ Processing blocks transactions
❌ No separation of concerns
❌ Difficult to optimize
```

### Phase 2: Data Warehouses (2010s)

```
┌─────────────┐      ETL      ┌──────────────┐
│   OLTP DB   ├──────────────►│  Data        │
│ (Transact)  │   Nightly     │  Warehouse   │
└─────────────┘               │  (Analytics) │
                              └──────────────┘

Improvements:
✅ Separated transactional and analytical workloads
✅ Optimized for different query patterns

Limitations:
❌ Batch processing only (hours of delay)
❌ Complex ETL pipelines
❌ Expensive to scale
```

### Phase 3: Big Data Era (2015+)

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Hadoop   │    │  Spark   │    │  Kafka   │
│ (Batch)  │    │ (Batch)  │    │ (Stream) │
└──────────┘    └──────────┘    └──────────┘
      │              │                │
      └──────────────┴────────────────┘
                     │
              ┌──────▼───────┐
              │  Data Lake   │
              │    (HDFS)    │
              └──────────────┘

Improvements:
✅ Massive scale (petabytes)
✅ Both batch and stream processing
✅ Schema-on-read flexibility

Challenges:
⚠️ Complex infrastructure
⚠️ Requires specialized skills
⚠️ High operational overhead
```

### Phase 4: Modern Data Stack (2020+)

```
┌────────────────────────────────────────────┐
│           Data Sources                     │
│  Apps │ APIs │ Databases │ Events │ Logs  │
└───┬────────────────────────────────────┬───┘
    │                                    │
    ▼                                    ▼
┌─────────────┐                  ┌─────────────┐
│   Kafka     │                  │  Debezium   │
│  (Stream)   │◄─────────────────│    (CDC)    │
└──────┬──────┘                  └─────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│         Stream Processing                    │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐    │
│  │  Flink  │  │ Kafka   │  │  Custom  │    │
│  │ Streams │  │ Streams │  │  Go Apps │    │
│  └─────────┘  └─────────┘  └──────────┘    │
└────────┬────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│         Batch Orchestration                  │
│           Apache Airflow                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  ETL DAG │  │  ML DAG  │  │ Report   │  │
│  │          │  │          │  │   DAG    │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└────────┬────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│           Data Storage Layer                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │PostgreSQL│  │ClickHouse│  │   S3/    │  │
│  │  (OLTP)  │  │(Analytics│  │  MinIO   │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘

Modern Advantages:
✅ Real-time and batch processing
✅ Modular, best-of-breed tools
✅ Container-native (easy deployment)
✅ Language flexibility (Go, Python, Java)
✅ Lower operational overhead
```

This is the architecture we'll implement in this chapter.

---

## Core Data Processing Patterns

### Pattern 1: Lambda Architecture

Combines batch and stream processing for complete data view.

```
┌──────────────┐
│ Data Source  │
└──────┬───────┘
       │
       ├─────────────────┬──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
┌─────────────┐   ┌────────────┐   ┌──────────────┐
│  Batch      │   │  Speed     │   │  Raw Data    │
│  Layer      │   │  Layer     │   │  Storage     │
│  (Airflow)  │   │  (Kafka)   │   │  (S3/MinIO)  │
└──────┬──────┘   └──────┬─────┘   └──────────────┘
       │                 │
       └────────┬────────┘
                │
                ▼
        ┌───────────────┐
        │  Serving      │
        │  Layer        │
        │  (PostgreSQL/ │
        │  ClickHouse)  │
        └───────────────┘

Use When:
- Need both real-time and batch processing
- Historical reprocessing required
- High accuracy demands
- Complex analytics

Trade-offs:
+ Complete data coverage
+ Fault tolerant
- More complex to maintain
- Duplicate processing logic
```

### Pattern 2: Kappa Architecture

Stream-first architecture, simpler than Lambda.

```
┌──────────────┐
│ Data Source  │
└──────┬───────┘
       │
       ▼
┌─────────────────┐
│  Kafka          │
│  (Event Log)    │
│  Infinite       │
│  Retention      │
└────────┬────────┘
         │
         ├────────────────┬──────────────────┐
         │                │                  │
         ▼                ▼                  ▼
  ┌────────────┐   ┌────────────┐   ┌────────────┐
  │ Stream     │   │ Stream     │   │ Batch      │
  │ Processor  │   │ Processor  │   │ Reprocess  │
  │ (Real-time)│   │ (Real-time)│   │ (History)  │
  └──────┬─────┘   └──────┬─────┘   └──────┬─────┘
         │                │                │
         └────────────────┴────────────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Data Store   │
                  └───────────────┘

Use When:
- Stream processing primary workload
- Replay capability needed
- Simpler operations preferred
- Event sourcing architecture

Trade-offs:
+ Simpler than Lambda
+ Single processing logic
+ Replayable
- Requires more storage
- Stream processing must handle all cases
```

### Pattern 3: Event Sourcing

Store events, derive state.

```
┌────────────────────────────────────────┐
│         Application Events             │
│  OrderCreated │ PaymentReceived │ ...  │
└────────────┬───────────────────────────┘
             │
             ▼
      ┌──────────────┐
      │  Event Store │  ◄── Immutable, Append-Only
      │  (Kafka)     │      Full History
      └──────┬───────┘
             │
             ├────────────────┬──────────────┐
             │                │              │
             ▼                ▼              ▼
    ┌────────────┐   ┌────────────┐  ┌───────────┐
    │  Read      │   │  Read      │  │  Read     │
    │  Model 1   │   │  Model 2   │  │  Model 3  │
    │ (Current)  │   │(Analytics) │  │  (Audit)  │
    └────────────┘   └────────────┘  └───────────┘

Benefits:
✅ Complete audit trail
✅ Time travel (replay to any point)
✅ Multiple read models
✅ Event-driven architecture

When to Use:
- Audit requirements
- Complex domain logic
- Need to reconstruct state
- Event-driven systems
```

### Pattern 4: Change Data Capture (CDC)

Capture database changes as events.

```
┌──────────────────┐
│   PostgreSQL     │
│                  │
│  ┌────────────┐  │
│  │   Orders   │  │
│  │   Table    │  │
│  └─────┬──────┘  │
│        │         │
│        │ WAL     │
└────────┼─────────┘
         │
         ▼
  ┌──────────────┐
  │  Debezium    │  ◄── Reads WAL, Emits Events
  │  Connector   │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │    Kafka     │
  │  orders.     │
  │  changes     │
  └──────┬───────┘
         │
         ├────────────────┬──────────────┐
         │                │              │
         ▼                ▼              ▼
  ┌───────────┐   ┌───────────┐  ┌──────────┐
  │ Elastic   │   │ Data Lake │  │Analytics │
  │  Search   │   │  (S3)     │  │   DB     │
  └───────────┘   └───────────┘  └──────────┘

Use Cases:
- Real-time data sync
- Event-driven architecture
- Audit logs
- Cache invalidation
- Search index updates

Tools:
- Debezium (open source, best for PostgreSQL/MySQL)
- AWS DMS
- Kafka Connect
```

### Pattern 5: Stream Processing

Real-time data transformation.

```
┌────────────────────────────────────────┐
│          Input Streams                 │
│  Clicks │ Orders │ Inventory │ Events  │
└────────┬───────────────────────────────┘
         │
         ▼
  ┌──────────────────┐
  │  Apache Kafka    │
  └────────┬─────────┘
           │
           ▼
  ┌────────────────────────────────────┐
  │    Stream Processor                │
  │                                    │
  │  ┌──────────────────────────────┐ │
  │  │  Window Functions            │ │
  │  │  - Tumbling (5 min windows)  │ │
  │  │  - Sliding (overlap)         │ │
  │  │  - Session (user activity)   │ │
  │  └──────────────────────────────┘ │
  │                                    │
  │  ┌──────────────────────────────┐ │
  │  │  Aggregations                │ │
  │  │  - Count, Sum, Avg           │ │
  │  │  - TopN, Distinct            │ │
  │  └──────────────────────────────┘ │
  │                                    │
  │  ┌──────────────────────────────┐ │
  │  │  Joins                       │ │
  │  │  - Stream-Stream             │ │
  │  │  - Stream-Table              │ │
  │  └──────────────────────────────┘ │
  └────────┬───────────────────────────┘
           │
           ▼
  ┌────────────────────┐
  │  Output Streams    │
  │  or Databases      │
  └────────────────────┘

Common Operations:
1. Filter: Drop irrelevant events
2. Map: Transform events
3. FlatMap: One-to-many transformation
4. Aggregate: Combine events (count, sum)
5. Window: Time-based grouping
6. Join: Combine streams
7. Deduplicate: Remove duplicates
```

---

## Building Block 1: Apache Kafka (Event Streaming)

Kafka is the backbone of modern data pipelines.

### Why Kafka?

**Traditional Message Queue:**
```
Producer → Queue → Consumer
           (Message deleted after read)
```

**Kafka:**
```
Producer → Topic (Retained for days/weeks) → Consumer Group 1
                                           → Consumer Group 2
                                           → Consumer Group 3
                (Multiple consumers can read same data)
```

### Kafka Architecture

```
┌─────────────────────────────────────────────────┐
│            Kafka Cluster                        │
│                                                 │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
│  │  Broker 1 │  │  Broker 2 │  │  Broker 3 │  │
│  │           │  │           │  │           │  │
│  │  Topic: orders (Partitions 0,1,2)       │  │
│  │  ┌────┐   │  │  ┌────┐   │  │  ┌────┐  │  │
│  │  │ P0 │   │  │  │ P1 │   │  │  │ P2 │  │  │
│  │  └────┘   │  │  └────┘   │  │  └────┘  │  │
│  │  replica  │  │  replica  │  │  replica  │  │
│  └───────────┘  └───────────┘  └───────────┘  │
└─────────────────────────────────────────────────┘
         ▲                             │
         │                             │
    ┌────┴────┐                   ┌────▼────┐
    │Producer │                   │Consumer │
    │         │                   │  Group  │
    └─────────┘                   └─────────┘
```

### Complete Kafka Setup

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    restart: unless-stopped
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_log:/var/lib/zookeeper/log
    networks:
      - kafka_network

  kafka-1:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-1
    restart: unless-stopped
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "19092:19092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:9092,PLAINTEXT_HOST://localhost:19092
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_RETENTION_BYTES: 1073741824
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
    volumes:
      - kafka_1_data:/var/lib/kafka/data
    networks:
      - kafka_network

  kafka-2:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-2
    restart: unless-stopped
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
      - "19093:19093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:9093,PLAINTEXT_HOST://localhost:19093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
    volumes:
      - kafka_2_data:/var/lib/kafka/data
    networks:
      - kafka_network

  kafka-3:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-3
    restart: unless-stopped
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
      - "19094:19094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:9094,PLAINTEXT_HOST://localhost:19094
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
    volumes:
      - kafka_3_data:/var/lib/kafka/data
    networks:
      - kafka_network

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    restart: unless-stopped
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092,kafka-2:9093,kafka-3:9094
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    networks:
      - kafka_network

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    container_name: schema-registry
    restart: unless-stopped
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9093,kafka-3:9094
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    networks:
      - kafka_network

volumes:
  zookeeper_data:
  zookeeper_log:
  kafka_1_data:
  kafka_2_data:
  kafka_3_data:

networks:
  kafka_network:
    driver: bridge
```

### Kafka Producer (Go)

```go
// producer/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

type OrderEvent struct {
    OrderID    string    `json:"order_id"`
    UserID     string    `json:"user_id"`
    Amount     float64   `json:"amount"`
    Status     string    `json:"status"`
    Timestamp  time.Time `json:"timestamp"`
}

type KafkaProducer struct {
    writer *kafka.Writer
}

func NewKafkaProducer(brokers []string, topic string) *KafkaProducer {
    writer := &kafka.Writer{
        Addr:         kafka.TCP(brokers...),
        Topic:        topic,
        Balancer:     &kafka.Hash{}, // Partition by key
        RequiredAcks: kafka.RequireAll, // Wait for all replicas
        Async:        false, // Synchronous for reliability
        Compression:  kafka.Snappy,
        BatchSize:    100,
        BatchTimeout: 10 * time.Millisecond,
    }

    return &KafkaProducer{writer: writer}
}

func (p *KafkaProducer) ProduceOrder(ctx context.Context, order OrderEvent) error {
    // Serialize to JSON
    value, err := json.Marshal(order)
    if err != nil {
        return fmt.Errorf("failed to marshal order: %w", err)
    }

    // Create message
    msg := kafka.Message{
        Key:   []byte(order.OrderID), // Partition by order ID
        Value: value,
        Headers: []kafka.Header{
            {Key: "event-type", Value: []byte("order")},
            {Key: "version", Value: []byte("v1")},
        },
        Time: time.Now(),
    }

    // Write to Kafka
    err = p.writer.WriteMessages(ctx, msg)
    if err != nil {
        return fmt.Errorf("failed to write message: %w", err)
    }

    log.Printf("✅ Produced order: %s to Kafka\n", order.OrderID)
    return nil
}

func (p *KafkaProducer) ProduceBatch(ctx context.Context, orders []OrderEvent) error {
    messages := make([]kafka.Message, len(orders))

    for i, order := range orders {
        value, err := json.Marshal(order)
        if err != nil {
            return fmt.Errorf("failed to marshal order %s: %w", order.OrderID, err)
        }

        messages[i] = kafka.Message{
            Key:   []byte(order.OrderID),
            Value: value,
            Time:  time.Now(),
        }
    }

    err := p.writer.WriteMessages(ctx, messages...)
    if err != nil {
        return fmt.Errorf("failed to write batch: %w", err)
    }

    log.Printf("✅ Produced %d orders to Kafka\n", len(orders))
    return nil
}

func (p *KafkaProducer) Close() error {
    return p.writer.Close()
}

func main() {
    brokers := []string{"localhost:19092", "localhost:19093", "localhost:19094"}
    producer := NewKafkaProducer(brokers, "orders")
    defer producer.Close()

    ctx := context.Background()

    // Produce single order
    order := OrderEvent{
        OrderID:   "ORD-12345",
        UserID:    "USER-789",
        Amount:    99.99,
        Status:    "created",
        Timestamp: time.Now(),
    }

    err := producer.ProduceOrder(ctx, order)
    if err != nil {
        log.Fatalf("❌ Failed to produce order: %v", err)
    }

    // Produce batch
    orders := []OrderEvent{
        {OrderID: "ORD-100", UserID: "USER-1", Amount: 50.00, Status: "created", Timestamp: time.Now()},
        {OrderID: "ORD-101", UserID: "USER-2", Amount: 75.50, Status: "created", Timestamp: time.Now()},
        {OrderID: "ORD-102", UserID: "USER-3", Amount: 120.00, Status: "created", Timestamp: time.Now()},
    }

    err = producer.ProduceBatch(ctx, orders)
    if err != nil {
        log.Fatalf("❌ Failed to produce batch: %v", err)
    }
}
```

### Kafka Consumer (Go)

```go
// consumer/main.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/segmentio/kafka-go"
)

type OrderEvent struct {
    OrderID   string    `json:"order_id"`
    UserID    string    `json:"user_id"`
    Amount    float64   `json:"amount"`
    Status    string    `json:"status"`
    Timestamp time.Time `json:"timestamp"`
}

type KafkaConsumer struct {
    reader *kafka.Reader
}

func NewKafkaConsumer(brokers []string, topic, groupID string) *KafkaConsumer {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:        brokers,
        GroupID:        groupID,
        Topic:          topic,
        MinBytes:       1,         // 1 byte
        MaxBytes:       10e6,      // 10MB
        CommitInterval: time.Second, // Auto-commit every second
        StartOffset:    kafka.LastOffset, // Start from latest
        MaxWait:        500 * time.Millisecond,
        ErrorLogger:    kafka.LoggerFunc(log.Printf),
    })

    return &KafkaConsumer{reader: reader}
}

func (c *KafkaConsumer) Consume(ctx context.Context, handler func(OrderEvent) error) error {
    for {
        select {
        case <-ctx.Done():
            log.Println("🛑 Consumer shutting down...")
            return ctx.Err()
        default:
            // Read message
            msg, err := c.reader.ReadMessage(ctx)
            if err != nil {
                log.Printf("❌ Error reading message: %v\n", err)
                continue
            }

            // Deserialize
            var order OrderEvent
            err = json.Unmarshal(msg.Value, &order)
            if err != nil {
                log.Printf("❌ Error unmarshaling message: %v\n", err)
                continue
            }

            // Process
            log.Printf("📨 Received order: %s (offset: %d, partition: %d)\n",
                order.OrderID, msg.Offset, msg.Partition)

            err = handler(order)
            if err != nil {
                log.Printf("❌ Error processing order %s: %v\n", order.OrderID, err)
                // In production, you might send to DLQ (Dead Letter Queue)
                continue
            }

            log.Printf("✅ Processed order: %s\n", order.OrderID)
        }
    }
}

func (c *KafkaConsumer) Close() error {
    return c.reader.Close()
}

func main() {
    brokers := []string{"localhost:19092", "localhost:19093", "localhost:19094"}
    consumer := NewKafkaConsumer(brokers, "orders", "order-processor-group")
    defer consumer.Close()

    // Create context with cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle graceful shutdown
    sigterm := make(chan os.Signal, 1)
    signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        <-sigterm
        log.Println("🛑 Received shutdown signal")
        cancel()
    }()

    // Define handler
    handler := func(order OrderEvent) error {
        // Business logic here
        // Examples:
        // - Update database
        // - Send notification
        // - Calculate metrics
        // - Trigger downstream services

        // Simulate processing
        time.Sleep(100 * time.Millisecond)

        if order.Amount > 1000 {
            log.Printf("⚠️  High-value order detected: %s ($%.2f)\n", order.OrderID, order.Amount)
            // Send alert
        }

        return nil
    }

    // Start consuming
    log.Println("🚀 Consumer started, waiting for messages...")
    err := consumer.Consume(ctx, handler)
    if err != nil && err != context.Canceled {
        log.Fatalf("❌ Consumer error: %v", err)
    }

    log.Println("✅ Consumer shutdown complete")
}
```

### Kafka Stream Processing (Go)

```go
// stream-processor/main.go
package main

import (
    "context"
    "encoding/json"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

type OrderEvent struct {
    OrderID   string    `json:"order_id"`
    Amount    float64   `json:"amount"`
    Timestamp time.Time `json:"timestamp"`
}

type OrderStats struct {
    WindowStart time.Time `json:"window_start"`
    WindowEnd   time.Time `json:"window_end"`
    OrderCount  int       `json:"order_count"`
    TotalAmount float64   `json:"total_amount"`
    AvgAmount   float64   `json:"avg_amount"`
}

type StreamProcessor struct {
    reader *kafka.Reader
    writer *kafka.Writer
}

func NewStreamProcessor(brokers []string, inputTopic, outputTopic string) *StreamProcessor {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:  brokers,
        GroupID:  "stream-processor-group",
        Topic:    inputTopic,
        MinBytes: 1,
        MaxBytes: 10e6,
    })

    writer := &kafka.Writer{
        Addr:         kafka.TCP(brokers...),
        Topic:        outputTopic,
        Balancer:     &kafka.Hash{},
        RequiredAcks: kafka.RequireAll,
    }

    return &StreamProcessor{reader: reader, writer: writer}
}

func (sp *StreamProcessor) ProcessWindowed(ctx context.Context, windowDuration time.Duration) {
    window := make([]OrderEvent, 0)
    windowStart := time.Now()

    ticker := time.NewTicker(windowDuration)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            // Window completed, calculate stats
            if len(window) > 0 {
                stats := sp.calculateStats(window, windowStart, time.Now())
                sp.emitStats(ctx, stats)

                // Reset window
                window = make([]OrderEvent, 0)
                windowStart = time.Now()
            }
        default:
            // Read message (non-blocking)
            msg, err := sp.reader.ReadMessage(ctx)
            if err != nil {
                continue
            }

            var order OrderEvent
            err = json.Unmarshal(msg.Value, &order)
            if err != nil {
                log.Printf("❌ Error unmarshaling: %v\n", err)
                continue
            }

            window = append(window, order)
        }
    }
}

func (sp *StreamProcessor) calculateStats(orders []OrderEvent, start, end time.Time) OrderStats {
    var totalAmount float64
    for _, order := range orders {
        totalAmount += order.Amount
    }

    avgAmount := totalAmount / float64(len(orders))

    return OrderStats{
        WindowStart: start,
        WindowEnd:   end,
        OrderCount:  len(orders),
        TotalAmount: totalAmount,
        AvgAmount:   avgAmount,
    }
}

func (sp *StreamProcessor) emitStats(ctx context.Context, stats OrderStats) {
    value, err := json.Marshal(stats)
    if err != nil {
        log.Printf("❌ Error marshaling stats: %v\n", err)
        return
    }

    msg := kafka.Message{
        Key:   []byte(stats.WindowStart.Format(time.RFC3339)),
        Value: value,
        Time:  time.Now(),
    }

    err = sp.writer.WriteMessages(ctx, msg)
    if err != nil {
        log.Printf("❌ Error writing stats: %v\n", err)
        return
    }

    log.Printf("📊 Window stats: %d orders, $%.2f total, $%.2f avg\n",
        stats.OrderCount, stats.TotalAmount, stats.AvgAmount)
}

func (sp *StreamProcessor) Close() {
    sp.reader.Close()
    sp.writer.Close()
}

func main() {
    brokers := []string{"localhost:19092", "localhost:19093", "localhost:19094"}
    processor := NewStreamProcessor(brokers, "orders", "order-stats")
    defer processor.Close()

    ctx := context.Background()

    log.Println("🚀 Stream processor started (5-minute windows)")
    processor.ProcessWindowed(ctx, 5*time.Minute)
}
```

---

## Building Block 2: Apache Airflow (Batch Orchestration)

Airflow orchestrates complex data workflows.

### Airflow Architecture

```
┌───────────────────────────────────────────────┐
│           Airflow Components                  │
│                                               │
│  ┌─────────────┐         ┌─────────────┐    │
│  │   Web UI    │         │  Scheduler  │    │
│  │  (Flask)    │         │  (DAG exec) │    │
│  └─────────────┘         └──────┬──────┘    │
│         │                       │            │
│         │                       │            │
│  ┌──────▼───────────────────────▼────────┐  │
│  │       Metadata Database               │  │
│  │       (PostgreSQL)                    │  │
│  └───────────────────────────────────────┘  │
│                       │                      │
│                       │                      │
│  ┌────────────────────▼──────────────────┐  │
│  │          Executor                     │  │
│  │  ┌──────────┐  ┌──────────┐         │  │
│  │  │ Worker 1 │  │ Worker 2 │  ...    │  │
│  │  └──────────┘  └──────────┘         │  │
│  └────────────────────────────────────────┘ │
└───────────────────────────────────────────────┘
```

### Complete Airflow Setup

**docker-compose.yml:**

```yaml
version: '3.8'

x-airflow-common:
  &airflow-common
  image: apache/airflow:2.7.1
  environment: &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth'
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins
    - ./config:/opt/airflow/config
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on: &airflow-common-depends-on
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:15
    container_name: airflow-postgres
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - airflow_network

  redis:
    image: redis:7-alpine
    container_name: airflow-redis
    expose:
      - 6379
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 30s
      retries: 50
    restart: unless-stopped
    networks:
      - airflow_network

  airflow-webserver:
    <<: *airflow-common
    container_name: airflow-webserver
    command: webserver
    ports:
      - "8082:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 10s
      timeout: 10s
      retries: 5
    restart: unless-stopped
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully
    networks:
      - airflow_network

  airflow-scheduler:
    <<: *airflow-common
    container_name: airflow-scheduler
    command: scheduler
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type SchedulerJob --hostname "$${HOSTNAME}"']
      interval: 10s
      timeout: 10s
      retries: 5
    restart: unless-stopped
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully
    networks:
      - airflow_network

  airflow-worker:
    <<: *airflow-common
    container_name: airflow-worker
    command: celery worker
    healthcheck:
      test:
        - "CMD-SHELL"
        - 'celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
      interval: 10s
      timeout: 10s
      retries: 5
    environment:
      <<: *airflow-common-env
      DUMB_INIT_SETSID: "0"
    restart: unless-stopped
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully
    networks:
      - airflow_network

  airflow-triggerer:
    <<: *airflow-common
    container_name: airflow-triggerer
    command: triggerer
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"']
      interval: 10s
      timeout: 10s
      retries: 5
    restart: unless-stopped
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully
    networks:
      - airflow_network

  airflow-init:
    <<: *airflow-common
    container_name: airflow-init
    entrypoint: /bin/bash
    command:
      - -c
      - |
        mkdir -p /sources/logs /sources/dags /sources/plugins
        chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
        exec /entrypoint airflow version
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_UPGRADE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
    user: "0:0"
    volumes:
      - .:/sources
    networks:
      - airflow_network

  airflow-cli:
    <<: *airflow-common
    container_name: airflow-cli
    profiles:
      - debug
    environment:
      <<: *airflow-common-env
      CONNECTION_CHECK_MAX_COUNT: "0"
    command:
      - bash
      - -c
      - airflow
    networks:
      - airflow_network

volumes:
  postgres_data:

networks:
  airflow_network:
    driver: bridge
```

### Example DAG: ETL Pipeline

```python
# dags/etl_orders.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
import pandas as pd
import logging

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email': ['alerts@example.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=1),
}

dag = DAG(
    'etl_orders_daily',
    default_args=default_args,
    description='Daily ETL for order data',
    schedule_interval='0 2 * * *',  # 2 AM daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['etl', 'orders', 'daily'],
)

def extract_orders(**context):
    """Extract orders from source database"""
    execution_date = context['execution_date']
    logging.info(f"Extracting orders for {execution_date}")

    # Connect to source database
    hook = PostgresHook(postgres_conn_id='source_db')
    conn = hook.get_conn()

    # Extract data
    query = """
        SELECT
            order_id,
            user_id,
            order_date,
            total_amount,
            status,
            created_at
        FROM orders
        WHERE DATE(created_at) = %(date)s
    """

    df = pd.read_sql(query, conn, params={'date': execution_date.date()})

    # Save to temp location
    output_path = f'/tmp/orders_{execution_date.strftime("%Y%m%d")}.csv'
    df.to_csv(output_path, index=False)

    logging.info(f"Extracted {len(df)} orders to {output_path}")

    # Push to XCom for next task
    context['task_instance'].xcom_push(key='extract_path', value=output_path)
    context['task_instance'].xcom_push(key='row_count', value=len(df))

    return output_path

def transform_orders(**context):
    """Transform and enrich order data"""
    ti = context['task_instance']
    input_path = ti.xcom_pull(task_ids='extract_orders', key='extract_path')

    logging.info(f"Transforming orders from {input_path}")

    # Load data
    df = pd.read_csv(input_path)

    # Transformations
    df['order_date'] = pd.to_datetime(df['order_date'])
    df['order_year'] = df['order_date'].dt.year
    df['order_month'] = df['order_date'].dt.month
    df['order_day'] = df['order_date'].dt.day
    df['order_dayofweek'] = df['order_date'].dt.dayofweek

    # Calculate order value category
    df['value_category'] = pd.cut(
        df['total_amount'],
        bins=[0, 50, 100, 500, float('inf')],
        labels=['low', 'medium', 'high', 'premium']
    )

    # Add processing timestamp
    df['processed_at'] = datetime.now()

    # Save transformed data
    execution_date = context['execution_date']
    output_path = f'/tmp/orders_transformed_{execution_date.strftime("%Y%m%d")}.csv'
    df.to_csv(output_path, index=False)

    logging.info(f"Transformed {len(df)} orders to {output_path}")

    ti.xcom_push(key='transform_path', value=output_path)

    return output_path

def load_orders(**context):
    """Load transformed data to data warehouse"""
    ti = context['task_instance']
    input_path = ti.xcom_pull(task_ids='transform_orders', key='transform_path')

    logging.info(f"Loading orders from {input_path}")

    # Load data
    df = pd.read_csv(input_path)

    # Connect to warehouse
    hook = PostgresHook(postgres_conn_id='warehouse_db')
    engine = hook.get_sqlalchemy_engine()

    # Load to staging table first
    df.to_sql(
        'orders_staging',
        engine,
        if_exists='replace',
        index=False,
        method='multi',
        chunksize=1000
    )

    logging.info(f"Loaded {len(df)} orders to staging table")

    # Merge to main table (handled by SQL task)
    ti.xcom_push(key='loaded_count', value=len(df))

    return len(df)

def data_quality_check(**context):
    """Validate data quality"""
    ti = context['task_instance']
    loaded_count = ti.xcom_pull(task_ids='load_orders', key='loaded_count')
    extracted_count = ti.xcom_pull(task_ids='extract_orders', key='row_count')

    logging.info(f"Quality check: Extracted {extracted_count}, Loaded {loaded_count}")

    # Check counts match
    if loaded_count != extracted_count:
        raise ValueError(f"Row count mismatch: {extracted_count} extracted vs {loaded_count} loaded")

    # Check for duplicates
    hook = PostgresHook(postgres_conn_id='warehouse_db')
    conn = hook.get_conn()
    cursor = conn.cursor()

    cursor.execute("""
        SELECT COUNT(*) - COUNT(DISTINCT order_id) as duplicates
        FROM orders_staging
    """)

    duplicates = cursor.fetchone()[0]
    if duplicates > 0:
        raise ValueError(f"Found {duplicates} duplicate orders")

    # Check for nulls in critical fields
    cursor.execute("""
        SELECT
            COUNT(*) FILTER (WHERE order_id IS NULL) as null_order_id,
            COUNT(*) FILTER (WHERE user_id IS NULL) as null_user_id,
            COUNT(*) FILTER (WHERE total_amount IS NULL) as null_amount
        FROM orders_staging
    """)

    null_counts = cursor.fetchone()
    if any(null_counts):
        raise ValueError(f"Found nulls in critical fields: {null_counts}")

    logging.info("✅ All quality checks passed")

    return True

# Define tasks
extract = PythonOperator(
    task_id='extract_orders',
    python_callable=extract_orders,
    dag=dag,
)

transform = PythonOperator(
    task_id='transform_orders',
    python_callable=transform_orders,
    dag=dag,
)

load = PythonOperator(
    task_id='load_orders',
    python_callable=load_orders,
    dag=dag,
)

merge_to_main = PostgresOperator(
    task_id='merge_to_main_table',
    postgres_conn_id='warehouse_db',
    sql="""
        INSERT INTO orders_fact (
            order_id, user_id, order_date, total_amount, status,
            order_year, order_month, order_day, order_dayofweek,
            value_category, processed_at, created_at
        )
        SELECT
            order_id, user_id, order_date, total_amount, status,
            order_year, order_month, order_day, order_dayofweek,
            value_category, processed_at, created_at
        FROM orders_staging
        ON CONFLICT (order_id)
        DO UPDATE SET
            status = EXCLUDED.status,
            total_amount = EXCLUDED.total_amount,
            processed_at = EXCLUDED.processed_at;
    """,
    dag=dag,
)

quality_check = PythonOperator(
    task_id='data_quality_check',
    python_callable=data_quality_check,
    dag=dag,
)

cleanup = BashOperator(
    task_id='cleanup_temp_files',
    bash_command='rm -f /tmp/orders_*.csv',
    dag=dag,
)

send_notification = BashOperator(
    task_id='send_success_notification',
    bash_command='echo "ETL completed successfully for {{ ds }}"',
    dag=dag,
)

# Define task dependencies
extract >> transform >> load >> merge_to_main >> quality_check >> [cleanup, send_notification]
```

### Advanced DAG: ML Pipeline

```python
# dags/ml_pipeline.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.dummy import DummyOperator
import logging

default_args = {
    'owner': 'ml-team',
    'depends_on_past': False,
    'email': ['ml-team@example.com'],
    'email_on_failure': True,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'ml_model_training',
    default_args=default_args,
    description='Train and deploy ML model',
    schedule_interval='0 3 * * 0',  # Weekly on Sunday at 3 AM
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['ml', 'training', 'weekly'],
)

def extract_training_data(**context):
    """Extract data for model training"""
    logging.info("Extracting training data...")
    # Simulate data extraction
    # In production: query warehouse, fetch from S3, etc.
    return {'rows': 50000, 'features': 20}

def preprocess_data(**context):
    """Preprocess and feature engineering"""
    logging.info("Preprocessing data...")
    # Simulate preprocessing
    # In production: scaling, encoding, feature creation
    return {'processed_rows': 48500}  # After cleaning

def train_model(**context):
    """Train ML model"""
    logging.info("Training model...")
    # Simulate training
    # In production: scikit-learn, TensorFlow, PyTorch
    metrics = {
        'accuracy': 0.89,
        'precision': 0.87,
        'recall': 0.91,
        'f1_score': 0.89
    }

    ti = context['task_instance']
    ti.xcom_push(key='metrics', value=metrics)

    return metrics

def evaluate_model(**context):
    """Evaluate model performance"""
    ti = context['task_instance']
    metrics = ti.xcom_pull(task_ids='train_model', key='metrics')

    logging.info(f"Model metrics: {metrics}")

    # Check if model meets threshold
    threshold = 0.85
    if metrics['f1_score'] >= threshold:
        logging.info(f"✅ Model passed evaluation (F1: {metrics['f1_score']})")
        return 'deploy_model'
    else:
        logging.warning(f"❌ Model failed evaluation (F1: {metrics['f1_score']} < {threshold})")
        return 'retrain_with_tuning'

def deploy_model(**context):
    """Deploy model to production"""
    logging.info("Deploying model to production...")
    # In production: save to S3, update serving endpoint, etc.
    return True

def retrain_with_tuning(**context):
    """Retrain with hyperparameter tuning"""
    logging.info("Retraining with hyperparameter tuning...")
    # In production: grid search, Optuna, etc.
    return True

# Define tasks
extract = PythonOperator(
    task_id='extract_training_data',
    python_callable=extract_training_data,
    dag=dag,
)

preprocess = PythonOperator(
    task_id='preprocess_data',
    python_callable=preprocess_data,
    dag=dag,
)

train = PythonOperator(
    task_id='train_model',
    python_callable=train_model,
    dag=dag,
)

evaluate = BranchPythonOperator(
    task_id='evaluate_model',
    python_callable=evaluate_model,
    dag=dag,
)

deploy = PythonOperator(
    task_id='deploy_model',
    python_callable=deploy_model,
    dag=dag,
)

retrain = PythonOperator(
    task_id='retrain_with_tuning',
    python_callable=retrain_with_tuning,
    dag=dag,
)

complete = DummyOperator(
    task_id='complete',
    trigger_rule='none_failed_min_one_success',
    dag=dag,
)

# Define dependencies
extract >> preprocess >> train >> evaluate
evaluate >> [deploy, retrain]
[deploy, retrain] >> complete
```

---

## Building Block 3: ClickHouse (Analytics Database)

ClickHouse is a fast columnar database for analytics.

### Why ClickHouse?

```
PostgreSQL (Row-oriented):
┌─────────────────────────────────────┐
│ Row 1: id=1, name=Alice, age=30,... │
│ Row 2: id=2, name=Bob, age=25,...   │
│ Row 3: id=3, name=Carol, age=35,... │
└─────────────────────────────────────┘
Good for: OLTP, updates, row lookups
Bad for: Aggregations, analytics

ClickHouse (Column-oriented):
┌─────┐  ┌───────┐  ┌─────┐
│ id  │  │ name  │  │ age │
├─────┤  ├───────┤  ├─────┤
│  1  │  │ Alice │  │ 30  │
│  2  │  │ Bob   │  │ 25  │
│  3  │  │ Carol │  │ 35  │
└─────┘  └───────┘  └─────┘
Good for: Analytics, aggregations, scans
Bad for: Updates, row-by-row operations

Performance Example:
Query: SELECT AVG(age) FROM users WHERE age > 20;

PostgreSQL: Read ALL columns for ALL rows = SLOW
ClickHouse: Read ONLY age column = FAST (100x-1000x faster)
```

### ClickHouse Setup

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  clickhouse:
    image: clickhouse/clickhouse-server:23.8
    container_name: clickhouse
    restart: unless-stopped
    ports:
      - "8123:8123"  # HTTP interface
      - "9000:9000"  # Native interface
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./clickhouse/config.xml:/etc/clickhouse-server/config.d/custom.xml
      - ./clickhouse/users.xml:/etc/clickhouse-server/users.d/custom_users.xml
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    environment:
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1
    networks:
      - analytics_network

  clickhouse-client:
    image: clickhouse/clickhouse-client:23.8
    container_name: clickhouse-client
    depends_on:
      - clickhouse
    command: --host clickhouse
    networks:
      - analytics_network

volumes:
  clickhouse_data:

networks:
  analytics_network:
    driver: bridge
```

**clickhouse/config.xml:**

```xml
<clickhouse>
    <logger>
        <level>information</level>
        <console>true</console>
    </logger>

    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>

    <listen_host>::</listen_host>

    <max_connections>4096</max_connections>
    <keep_alive_timeout>3</keep_alive_timeout>

    <max_concurrent_queries>100</max_concurrent_queries>
    <max_server_memory_usage>0</max_server_memory_usage>

    <timezone>UTC</timezone>
</clickhouse>
```

### ClickHouse Schema Design

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS analytics;

-- Events table with partitioning
CREATE TABLE IF NOT EXISTS analytics.events (
    event_id UUID,
    event_type LowCardinality(String),
    user_id UInt64,
    session_id UUID,
    timestamp DateTime,
    date Date,

    -- Event properties
    page_url String,
    referrer String,
    user_agent String,

    -- User properties
    country LowCardinality(String),
    city String,
    device_type LowCardinality(String),
    browser LowCardinality(String),

    -- Custom properties (JSON)
    properties String,

    -- Technical
    created_at DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (event_type, date, user_id, timestamp)
SETTINGS index_granularity = 8192;

-- Orders fact table
CREATE TABLE IF NOT EXISTS analytics.orders_fact (
    order_id String,
    user_id UInt64,
    order_date Date,
    order_timestamp DateTime,

    -- Amounts
    subtotal Decimal(10, 2),
    tax Decimal(10, 2),
    shipping Decimal(10, 2),
    total_amount Decimal(10, 2),

    -- Dimensions
    status LowCardinality(String),
    payment_method LowCardinality(String),
    shipping_method LowCardinality(String),

    -- Location
    country LowCardinality(String),
    state String,
    city String,

    -- Time dimensions
    order_year UInt16,
    order_month UInt8,
    order_day UInt8,
    order_hour UInt8,
    order_dayofweek UInt8,

    -- Flags
    is_first_order UInt8,
    is_mobile_order UInt8,

    created_at DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(order_date)
ORDER BY (order_date, user_id, order_id)
SETTINGS index_granularity = 8192;

-- Materialized view for daily stats
CREATE MATERIALIZED VIEW IF NOT EXISTS analytics.daily_order_stats
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(order_date)
ORDER BY (order_date, country)
AS SELECT
    order_date,
    country,
    count() as order_count,
    sum(total_amount) as total_revenue,
    avg(total_amount) as avg_order_value,
    uniq(user_id) as unique_customers
FROM analytics.orders_fact
GROUP BY order_date, country;

-- User sessions table
CREATE TABLE IF NOT EXISTS analytics.user_sessions (
    session_id UUID,
    user_id UInt64,
    start_time DateTime,
    end_time DateTime,
    date Date,

    -- Session metrics
    duration_seconds UInt32,
    page_views UInt16,
    events_count UInt16,

    -- Entry/Exit
    landing_page String,
    exit_page String,

    -- Conversion
    converted UInt8,
    conversion_value Decimal(10, 2),

    -- Attribution
    utm_source LowCardinality(String),
    utm_medium LowCardinality(String),
    utm_campaign String,

    created_at DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, user_id, start_time)
SETTINGS index_granularity = 8192;
```

### ClickHouse Go Client

```go
// clickhouse-client/main.go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/ClickHouse/clickhouse-go/v2"
    "github.com/google/uuid"
)

type ClickHouseClient struct {
    conn clickhouse.Conn
}

type Event struct {
    EventID     uuid.UUID
    EventType   string
    UserID      uint64
    SessionID   uuid.UUID
    Timestamp   time.Time
    Date        time.Time
    PageURL     string
    Referrer    string
    UserAgent   string
    Country     string
    City        string
    DeviceType  string
    Browser     string
    Properties  string
}

func NewClickHouseClient(addr string) (*ClickHouseClient, error) {
    conn, err := clickhouse.Open(&clickhouse.Options{
        Addr: []string{addr},
        Auth: clickhouse.Auth{
            Database: "analytics",
            Username: "default",
            Password: "",
        },
        DialTimeout:      time.Second * 30,
        MaxOpenConns:     10,
        MaxIdleConns:     5,
        ConnMaxLifetime:  time.Hour,
        ConnOpenStrategy: clickhouse.ConnOpenInOrder,
    })

    if err != nil {
        return nil, fmt.Errorf("failed to connect: %w", err)
    }

    // Test connection
    if err := conn.Ping(context.Background()); err != nil {
        return nil, fmt.Errorf("failed to ping: %w", err)
    }

    log.Println("✅ Connected to ClickHouse")

    return &ClickHouseClient{conn: conn}, nil
}

func (c *ClickHouseClient) InsertEvent(ctx context.Context, event Event) error {
    query := `
        INSERT INTO events (
            event_id, event_type, user_id, session_id, timestamp, date,
            page_url, referrer, user_agent, country, city, device_type, browser, properties
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    `

    err := c.conn.Exec(ctx, query,
        event.EventID,
        event.EventType,
        event.UserID,
        event.SessionID,
        event.Timestamp,
        event.Date,
        event.PageURL,
        event.Referrer,
        event.UserAgent,
        event.Country,
        event.City,
        event.DeviceType,
        event.Browser,
        event.Properties,
    )

    if err != nil {
        return fmt.Errorf("failed to insert event: %w", err)
    }

    return nil
}

func (c *ClickHouseClient) InsertEventsBatch(ctx context.Context, events []Event) error {
    batch, err := c.conn.PrepareBatch(ctx, `
        INSERT INTO events (
            event_id, event_type, user_id, session_id, timestamp, date,
            page_url, referrer, user_agent, country, city, device_type, browser, properties
        )
    `)

    if err != nil {
        return fmt.Errorf("failed to prepare batch: %w", err)
    }

    for _, event := range events {
        err := batch.Append(
            event.EventID,
            event.EventType,
            event.UserID,
            event.SessionID,
            event.Timestamp,
            event.Date,
            event.PageURL,
            event.Referrer,
            event.UserAgent,
            event.Country,
            event.City,
            event.DeviceType,
            event.Browser,
            event.Properties,
        )

        if err != nil {
            return fmt.Errorf("failed to append to batch: %w", err)
        }
    }

    err = batch.Send()
    if err != nil {
        return fmt.Errorf("failed to send batch: %w", err)
    }

    log.Printf("✅ Inserted %d events\n", len(events))

    return nil
}

type DailyStats struct {
    Date              time.Time
    EventCount        uint64
    UniqueUsers       uint64
    UniqueSessionsCount uint64
    PageViews         uint64
}

func (c *ClickHouseClient) GetDailyStats(ctx context.Context, startDate, endDate time.Time) ([]DailyStats, error) {
    query := `
        SELECT
            date,
            count() as event_count,
            uniq(user_id) as unique_users,
            uniq(session_id) as unique_sessions,
            countIf(event_type = 'page_view') as page_views
        FROM events
        WHERE date BETWEEN ? AND ?
        GROUP BY date
        ORDER BY date
    `

    rows, err := c.conn.Query(ctx, query, startDate, endDate)
    if err != nil {
        return nil, fmt.Errorf("failed to query: %w", err)
    }
    defer rows.Close()

    var stats []DailyStats
    for rows.Next() {
        var s DailyStats
        err := rows.Scan(
            &s.Date,
            &s.EventCount,
            &s.UniqueUsers,
            &s.UniqueSessionsCount,
            &s.PageViews,
        )
        if err != nil {
            return nil, fmt.Errorf("failed to scan row: %w", err)
        }
        stats = append(stats, s)
    }

    return stats, nil
}

type TopPage struct {
    PageURL   string
    ViewCount uint64
}

func (c *ClickHouseClient) GetTopPages(ctx context.Context, date time.Time, limit int) ([]TopPage, error) {
    query := `
        SELECT
            page_url,
            count() as view_count
        FROM events
        WHERE date = ? AND event_type = 'page_view'
        GROUP BY page_url
        ORDER BY view_count DESC
        LIMIT ?
    `

    rows, err := c.conn.Query(ctx, query, date, limit)
    if err != nil {
        return nil, fmt.Errorf("failed to query: %w", err)
    }
    defer rows.Close()

    var pages []TopPage
    for rows.Next() {
        var p TopPage
        err := rows.Scan(&p.PageURL, &p.ViewCount)
        if err != nil {
            return nil, fmt.Errorf("failed to scan row: %w", err)
        }
        pages = append(pages, p)
    }

    return pages, nil
}

func (c *ClickHouseClient) Close() error {
    return c.conn.Close()
}

func main() {
    client, err := NewClickHouseClient("localhost:9000")
    if err != nil {
        log.Fatalf("❌ Failed to create client: %v", err)
    }
    defer client.Close()

    ctx := context.Background()

    // Insert single event
    event := Event{
        EventID:    uuid.New(),
        EventType:  "page_view",
        UserID:     12345,
        SessionID:  uuid.New(),
        Timestamp:  time.Now(),
        Date:       time.Now(),
        PageURL:    "/products/widget",
        Referrer:   "/home",
        UserAgent:  "Mozilla/5.0...",
        Country:    "US",
        City:       "New York",
        DeviceType: "desktop",
        Browser:    "Chrome",
        Properties: `{"category": "widgets"}`,
    }

    err = client.InsertEvent(ctx, event)
    if err != nil {
        log.Fatalf("❌ Failed to insert event: %v", err)
    }

    // Insert batch
    events := make([]Event, 1000)
    for i := 0; i < 1000; i++ {
        events[i] = Event{
            EventID:    uuid.New(),
            EventType:  "page_view",
            UserID:     uint64(i),
            SessionID:  uuid.New(),
            Timestamp:  time.Now(),
            Date:       time.Now(),
            PageURL:    fmt.Sprintf("/page/%d", i),
            Country:    "US",
            DeviceType: "mobile",
            Browser:    "Safari",
            Properties: "{}",
        }
    }

    err = client.InsertEventsBatch(ctx, events)
    if err != nil {
        log.Fatalf("❌ Failed to insert batch: %v", err)
    }

    // Query daily stats
    startDate := time.Now().AddDate(0, 0, -7)
    endDate := time.Now()

    stats, err := client.GetDailyStats(ctx, startDate, endDate)
    if err != nil {
        log.Fatalf("❌ Failed to get stats: %v", err)
    }

    log.Println("📊 Daily Stats:")
    for _, s := range stats {
        log.Printf("  %s: %d events, %d users, %d sessions\n",
            s.Date.Format("2006-01-02"),
            s.EventCount,
            s.UniqueUsers,
            s.UniqueSessionsCount,
        )
    }

    // Query top pages
    topPages, err := client.GetTopPages(ctx, time.Now(), 10)
    if err != nil {
        log.Fatalf("❌ Failed to get top pages: %v", err)
    }

    log.Println("🏆 Top Pages:")
    for i, p := range topPages {
        log.Printf("  %d. %s (%d views)\n", i+1, p.PageURL, p.ViewCount)
    }
}
```

---

## Integration: Complete Data Pipeline

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                  Data Sources                        │
│   App Events │ API Calls │ Database Changes │ Logs  │
└────────┬─────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│              Kafka (Event Bus)                      │
│  Topics: events, orders, user-actions, system-logs  │
└────┬─────────────────────────────────────┬──────────┘
     │                                     │
     ▼                                     ▼
┌──────────────┐                   ┌──────────────────┐
│   Stream     │                   │   Raw Storage    │
│  Processing  │                   │   (S3/MinIO)     │
│              │                   │  - Parquet files │
│ - Real-time  │                   │  - Partitioned   │
│   analytics  │                   │    by date       │
│ - Enrichment │                   └──────────────────┘
│ - Filtering  │                            │
└──────┬───────┘                            │
       │                                    │
       ▼                                    ▼
┌──────────────────┐              ┌────────────────────┐
│   ClickHouse     │              │   Airflow (Batch)  │
│  (Real-time DB)  │              │  - Daily ETL       │
│  - Events        │◄─────────────┤  - Weekly reports  │
│  - Analytics     │              │  - ML training     │
│  - Dashboards    │              └────────────────────┘
└──────────────────┘
```

### docker-compose.yml (Complete System)

```yaml
version: '3.8'

services:
  # Kafka Cluster
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    volumes:
      - zookeeper_data:/var/lib/zookeeper

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "19092:19092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:19092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - kafka_data:/var/lib/kafka

  # Stream Processor
  stream-processor:
    build: ./stream-processor
    depends_on:
      - kafka
      - clickhouse
    environment:
      KAFKA_BROKERS: kafka:9092
      CLICKHOUSE_ADDR: clickhouse:9000
    restart: unless-stopped

  # ClickHouse
  clickhouse:
    image: clickhouse/clickhouse-server:23.8
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./clickhouse/init.sql:/docker-entrypoint-initdb.d/init.sql

  # MinIO (S3-compatible storage)
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data

  # Airflow (simplified single-node)
  airflow-standalone:
    image: apache/airflow:2.7.1
    command: standalone
    ports:
      - "8082:8080"
    environment:
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: sqlite:////opt/airflow/airflow.db
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - airflow_data:/opt/airflow

volumes:
  zookeeper_data:
  kafka_data:
  clickhouse_data:
  minio_data:
  airflow_data:
```

---

## Monitoring Data Pipelines

### Key Metrics to Track

```go
// monitoring/metrics.go
package monitoring

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Kafka metrics
    MessagesProduced = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "kafka_messages_produced_total",
            Help: "Total messages produced to Kafka",
        },
        []string{"topic"},
    )

    MessagesConsumed = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "kafka_messages_consumed_total",
            Help: "Total messages consumed from Kafka",
        },
        []string{"topic", "consumer_group"},
    )

    ConsumerLag = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "kafka_consumer_lag",
            Help: "Consumer lag in messages",
        },
        []string{"topic", "partition", "consumer_group"},
    )

    // Processing metrics
    ProcessingDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "message_processing_duration_seconds",
            Help:    "Time to process a message",
            Buckets: prometheus.ExponentialBuckets(0.001, 2, 10), // 1ms to ~1s
        },
        []string{"processor"},
    )

    ProcessingErrors = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "message_processing_errors_total",
            Help: "Total processing errors",
        },
        []string{"processor", "error_type"},
    )

    // Pipeline metrics
    PipelineRunDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "pipeline_run_duration_seconds",
            Help:    "Time to complete pipeline run",
            Buckets: prometheus.ExponentialBuckets(1, 2, 12), // 1s to ~1hr
        },
        []string{"pipeline", "status"},
    )

    RecordsProcessed = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "pipeline_records_processed_total",
            Help: "Total records processed by pipeline",
        },
        []string{"pipeline", "stage"},
    )

    // Data quality metrics
    DataQualityChecks = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "data_quality_checks_total",
            Help: "Total data quality checks",
        },
        []string{"check_type", "result"},
    )

    NullValueCount = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "data_null_values_total",
            Help: "Null values encountered",
        },
        []string{"table", "column"},
    )

    DuplicateRecords = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "data_duplicate_records_total",
            Help: "Duplicate records detected",
        },
        []string{"table"},
    )
)
```

### Grafana Dashboard JSON

Save as `grafana/dashboards/data-pipeline.json`:

```json
{
  "dashboard": {
    "title": "Data Pipeline Monitoring",
    "panels": [
      {
        "title": "Kafka Message Rate",
        "targets": [
          {
            "expr": "rate(kafka_messages_produced_total[5m])",
            "legendFormat": "Produced - {{topic}}"
          },
          {
            "expr": "rate(kafka_messages_consumed_total[5m])",
            "legendFormat": "Consumed - {{topic}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Consumer Lag",
        "targets": [
          {
            "expr": "kafka_consumer_lag",
            "legendFormat": "{{topic}} p{{partition}} - {{consumer_group}}"
          }
        ],
        "type": "graph",
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [10000],
                "type": "gt"
              },
              "query": {
                "params": ["A", "5m", "now"]
              }
            }
          ]
        }
      },
      {
        "title": "Processing Duration (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(message_processing_duration_seconds_bucket[5m]))",
            "legendFormat": "{{processor}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Processing Errors",
        "targets": [
          {
            "expr": "rate(message_processing_errors_total[5m])",
            "legendFormat": "{{processor}} - {{error_type}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Pipeline Success Rate",
        "targets": [
          {
            "expr": "rate(pipeline_run_duration_seconds_count{status=\"success\"}[5m]) / rate(pipeline_run_duration_seconds_count[5m])",
            "legendFormat": "{{pipeline}}"
          }
        ],
        "type": "gauge"
      },
      {
        "title": "Data Quality Issues",
        "targets": [
          {
            "expr": "rate(data_quality_checks_total{result=\"failed\"}[5m])",
            "legendFormat": "{{check_type}}"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

---

## Troubleshooting Guide

### Problem 1: Kafka Consumer Lag Growing

**Symptoms:**
- Consumer lag increasing over time
- Messages piling up in topics
- Processing delays

**Diagnosis:**

```bash
# Check consumer lag
kafka-consumer-groups.sh \
    --bootstrap-server localhost:19092 \
    --group mygroup \
    --describe

# Check partition distribution
kafka-topics.sh \
    --bootstrap-server localhost:19092 \
    --topic mytopic \
    --describe
```

**Solutions:**

1. Scale consumers:
```yaml
# docker-compose.yml
consumer:
  deploy:
    replicas: 3  # Increase consumer instances
```

2. Optimize processing:
```go
// Batch processing instead of one-by-one
func (c *Consumer) ConsumeBatch(ctx context.Context, batchSize int) error {
    batch := make([]Message, 0, batchSize)

    for len(batch) < batchSize {
        msg, err := c.reader.ReadMessage(ctx)
        if err != nil {
            break
        }
        batch = append(batch, msg)
    }

    return c.processBatch(batch)
}
```

3. Increase partitions:
```bash
kafka-topics.sh \
    --bootstrap-server localhost:19092 \
    --topic mytopic \
    --alter \
    --partitions 10
```

### Problem 2: Airflow DAG Running Slow

**Symptoms:**
- DAG takes hours to complete
- Task queue building up
- Scheduler lag

**Diagnosis:**

Check task duration in Airflow UI, or:

```sql
-- Query Airflow metadata DB
SELECT
    task_id,
    AVG(duration) as avg_duration,
    MAX(duration) as max_duration,
    COUNT(*) as run_count
FROM task_instance
WHERE dag_id = 'my_dag'
    AND state = 'success'
    AND start_date > NOW() - INTERVAL '7 days'
GROUP BY task_id
ORDER BY avg_duration DESC;
```

**Solutions:**

1. Parallelize tasks:
```python
# Before: Sequential
task1 >> task2 >> task3 >> task4

# After: Parallel where possible
task1 >> [task2, task3] >> task4
```

2. Use pools for resource management:
```python
task = PythonOperator(
    task_id='heavy_task',
    python_callable=process_data,
    pool='heavy_tasks',  # Limit concurrency
    priority_weight=10,   # Higher priority
)
```

3. Optimize data loading:
```python
# Use pandas chunks for large datasets
for chunk in pd.read_csv('large_file.csv', chunksize=10000):
    process_chunk(chunk)
```

### Problem 3: ClickHouse Queries Slow

**Symptoms:**
- Queries taking seconds instead of milliseconds
- High CPU usage
- Out of memory errors

**Diagnosis:**

```sql
-- Check query log
SELECT
    query,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage
FROM system.query_log
WHERE type = 'QueryFinish'
    AND query_duration_ms > 1000
ORDER BY query_duration_ms DESC
LIMIT 10;

-- Check table sizes
SELECT
    database,
    table,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY sum(bytes) DESC;
```

**Solutions:**

1. Optimize partition key:
```sql
-- Bad: No partitioning
ENGINE = MergeTree()
ORDER BY (user_id, timestamp)

-- Good: Partition by month
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, user_id, timestamp)
```

2. Use materialized views:
```sql
-- Instead of expensive query
CREATE MATERIALIZED VIEW daily_stats_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, category)
AS SELECT
    date,
    category,
    count() as event_count,
    sum(amount) as total_amount
FROM events
GROUP BY date, category;

-- Query the materialized view (fast!)
SELECT * FROM daily_stats_mv WHERE date = today();
```

3. Add proper indexes:
```sql
-- Add skipping index for common filters
ALTER TABLE events
ADD INDEX country_idx country TYPE set(100) GRANULARITY 4;

-- Use in queries
SELECT * FROM events WHERE country = 'US';  -- Uses index
```

### Problem 4: Data Quality Issues

**Symptoms:**
- Null values in critical fields
- Duplicate records
- Inconsistent data types
- Missing records

**Detection:**

```python
# Airflow data quality task
def check_data_quality(**context):
    hook = PostgresHook(postgres_conn_id='warehouse')

    # Check 1: Null values
    null_check = hook.get_first("""
        SELECT COUNT(*)
        FROM orders_staging
        WHERE order_id IS NULL OR total_amount IS NULL
    """)[0]

    if null_check > 0:
        raise ValueError(f"Found {null_check} records with null critical fields")

    # Check 2: Duplicates
    dup_check = hook.get_first("""
        SELECT COUNT(*) - COUNT(DISTINCT order_id)
        FROM orders_staging
    """)[0]

    if dup_check > 0:
        raise ValueError(f"Found {dup_check} duplicate orders")

    # Check 3: Data freshness
    latest_record = hook.get_first("""
        SELECT MAX(created_at) FROM orders_staging
    """)[0]

    if (datetime.now() - latest_record).total_seconds() > 3600:
        raise ValueError("Data is more than 1 hour old")

    # Check 4: Record count
    expected_count = context['task_instance'].xcom_pull(
        task_ids='extract',
        key='row_count'
    )

    actual_count = hook.get_first("""
        SELECT COUNT(*) FROM orders_staging
    """)[0]

    if actual_count != expected_count:
        raise ValueError(f"Count mismatch: {expected_count} vs {actual_count}")

    return True
```

**Prevention:**

```python
# Validate at ingestion time
def validate_event(event: dict) -> bool:
    required_fields = ['event_id', 'event_type', 'user_id', 'timestamp']

    # Check required fields
    for field in required_fields:
        if field not in event or event[field] is None:
            logging.error(f"Missing required field: {field}")
            return False

    # Type validation
    if not isinstance(event['user_id'], int):
        logging.error(f"Invalid type for user_id: {type(event['user_id'])}")
        return False

    # Range validation
    if event['timestamp'] > time.time() + 3600:
        logging.error("Timestamp is in the future")
        return False

    return True

# Use in consumer
def process_message(msg):
    event = json.loads(msg.value)

    if not validate_event(event):
        # Send to DLQ (Dead Letter Queue)
        dlq_producer.produce('events-dlq', msg.value)
        return

    # Process valid event
    process_event(event)
```

---

## Best Practices

### DO ✅

1. **Use Batch Operations**: 100-1000x faster than individual operations
   ```go
   // Good: Batch insert
   batch := make([]Event, 1000)
   client.InsertBatch(ctx, batch)

   // Bad: Individual inserts
   for _, event := range events {
       client.Insert(ctx, event)  // 1000 network round trips!
   }
   ```

2. **Partition Data by Time**: Enables efficient querying and data retention
   ```sql
   PARTITION BY toYYYYMM(date)  -- Monthly partitions
   -- Easy to drop old data:
   ALTER TABLE events DROP PARTITION '202301'
   ```

3. **Monitor Consumer Lag**: Set up alerts before it becomes a problem
   ```yaml
   alert: ConsumerLagHigh
   expr: kafka_consumer_lag > 10000
   for: 5m
   annotations:
     summary: "Consumer lag is {{ $value }}"
   ```

4. **Validate Data Early**: Catch issues at ingestion, not analytics
   ```go
   if !isValid(event) {
       metrics.ValidationErrors.Inc()
       sendToDLQ(event)
       return
   }
   ```

5. **Use Idempotent Operations**: Support exactly-once processing
   ```sql
   INSERT INTO orders (order_id, ...)
   VALUES (...)
   ON CONFLICT (order_id) DO UPDATE
   SET updated_at = NOW();
   ```

6. **Implement Circuit Breakers**: Prevent cascade failures
   ```go
   if errorRate > 0.5 {
       circuitBreaker.Open()
       return ErrCircuitOpen
   }
   ```

7. **Version Your Schemas**: Enable safe evolution
   ```go
   type Event struct {
       SchemaVersion int    `json:"schema_version"`
       EventType     string `json:"event_type"`
       // ...
   }
   ```

8. **Test with Production-Like Data**: Catch issues before production
   ```python
   def test_pipeline_with_real_data():
       # Use anonymized production data
       sample = load_sample('production_orders.csv')
       result = run_pipeline(sample)
       assert_no_quality_issues(result)
   ```

9. **Document Data Lineage**: Track data from source to destination
   ```python
   metadata = {
       'source': 'orders_api',
       'extraction_time': datetime.now(),
       'transformer': 'etl_v2',
       'quality_checks': ['null_check', 'dup_check'],
   }
   ```

10. **Set Up Data Retention Policies**: Don't store forever
    ```sql
    -- ClickHouse TTL
    ALTER TABLE events
    MODIFY TTL date + INTERVAL 90 DAY DELETE;
    ```

### DON'T ❌

1. **Don't Block on External Services**: Use async patterns
   ```go
   // Bad: Synchronous call in stream processor
   response := httpClient.Post(url, data)  // Blocks!

   // Good: Async with timeout
   ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
   defer cancel()
   response := httpClient.Do(ctx, request)
   ```

2. **Don't Ignore Backpressure**: Will cause OOM
   ```go
   // Bad: Unbounded channel
   events := make(chan Event)

   // Good: Bounded channel with monitoring
   events := make(chan Event, 1000)
   if len(events) > 900 {
       metrics.BackpressureActive.Set(1)
   }
   ```

3. **Don't Store Uncompressed Data**: Wastes space and I/O
   ```yaml
   # Good: Enable compression
   kafka:
     compression: snappy  # or lz4, zstd

   # ClickHouse: Use codecs
   amount Decimal(10,2) CODEC(Delta, ZSTD)
   ```

4. **Don't Use SELECT ***: Specify columns, especially in columnar DBs
   ```sql
   -- Bad
   SELECT * FROM events WHERE date = today()

   -- Good
   SELECT event_type, user_id, timestamp
   FROM events
   WHERE date = today()
   ```

5. **Don't Ignore Failed Messages**: They indicate problems
   ```go
   // Bad: Silently drop
   if err := process(msg) {
       return nil  // Lost!
   }

   // Good: Send to DLQ and alert
   if err := process(msg) {
       sendToDLQ(msg)
       metrics.ProcessingErrors.Inc()
       return err
   }
   ```

6. **Don't Run Analytics on OLTP DB**: Use dedicated analytics DB
   ```
   ❌ PostgreSQL (OLTP) ← Complex analytics queries
   ✅ PostgreSQL (OLTP) → CDC → ClickHouse (OLAP) ← Analytics
   ```

7. **Don't Mix Data Models**: Separate transactional and analytical
   ```
   OLTP: Normalized (3NF)
   OLAP: Denormalized (Star/Snowflake schema)
   ```

8. **Don't Forget Graceful Shutdown**: Prevents data loss
   ```go
   // Handle SIGTERM
   sigterm := make(chan os.Signal, 1)
   signal.Notify(sigterm, syscall.SIGINT, syscall.SIGTERM)

   <-sigterm
   consumer.Close()  // Commit offsets before exit
   ```

9. **Don't Trust Input Data**: Always validate
   ```go
   // Validate, sanitize, and set defaults
   if event.Timestamp == 0 {
       event.Timestamp = time.Now().Unix()
   }
   if event.Amount < 0 {
       return errors.New("negative amount")
   }
   ```

10. **Don't Over-Engineer**: Start simple, add complexity as needed
    ```
    Start: Kafka + Simple consumer → PostgreSQL
    Later: Add ClickHouse for analytics
    Much later: Add stream processing, ML pipelines
    ```

---

## Further Reading

**Books:**
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Streaming Systems" by Tyler Akidau et al.
- "The Data Warehouse Toolkit" by Ralph Kimball

**Documentation:**
- Apache Kafka: https://kafka.apache.org/documentation/
- Apache Airflow: https://airflow.apache.org/docs/
- ClickHouse: https://clickhouse.com/docs/

**Online Resources:**
- Confluent Blog: https://www.confluent.io/blog/
- DataEngineering Reddit: r/dataengineering
- ClickHouse Blog: https://clickhouse.com/blog/

---

## Summary

In this chapter, we built a complete data processing system:

1. **Apache Kafka** for event streaming and message bus
2. **Stream Processing** for real-time transformations
3. **Apache Airflow** for batch orchestration
4. **ClickHouse** for analytics storage
5. **MinIO/S3** for data lake storage

**Key Takeaways:**
- Use Lambda or Kappa architecture based on needs
- Kafka is the backbone of modern data pipelines
- ClickHouse excels at analytics workloads
- Airflow orchestrates complex workflows
- Monitor everything: lag, duration, quality
- Start simple, add complexity as needed

**Next Chapter Preview:**

Chapter 8 explores **The Muscular System (Compute/Processing Power)** - how to scale computational workloads with container orchestration, job queues, and distributed computing.

Ready to power up your infrastructure? Let's go! 💪🚀

---

## Apex Data Processing Practices

Elevate your organism's metabolism with the stacks elite data teams rely on:

- **Source of Truth Contracts** — Define schemas with Protocol Buffers + AsyncAPI, validate changes via CI contract tests, and publish lineage through DataHub or OpenMetadata so producers and consumers move in lockstep.
- **Streaming Backbone** — Operate Kafka or Redpanda with tiered storage, Schema Registry, and Cruise Control/Kafka Cruise Control; for low-latency analytics adopt Flink SQL or Materialize for incremental views.
- **Batch & Orchestration** — Use Dagster or Airflow with software-defined assets, task-level observability, and data-aware scheduling; integrate with dbt for SQL-first transformations and Slim CI for rapid feedback.
- **Quality & Trust** — Enforce Great Expectations or Soda Core checks, Monte Carlo or Bigeye anomaly detection, and unit-of-work idempotency via LakeFS branching so you can rewind bad loads instantly.
- **Storage & Governance** — Standardize on Apache Iceberg or Delta Lake tables stored in S3/MinIO with Ranger/Privacera policies, encryption, and row/column-level masking that satisfy regulatory baselines.
- **Privacy & Sharing** — Automate PII tokenization and differential privacy with Tonic.ai or Gretel, and serve governed data products through Snowflake Data Sharing or Apache Arrow Flight for controlled access.
- **Cost & Performance Optimization** — Track scan costs with FinOps dashboards (Zenseact, ProsperOps for data), leverage DuckDB or ClickHouse for ad-hoc exploration, and auto-scale warehouse compute via BigQuery slots or Snowflake warehouses driven by workload telemetry.

These practices keep pipelines trustworthy, explainable, and fast enough to power real-time decisions.
