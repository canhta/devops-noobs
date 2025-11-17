# 07. Kafka Architecture

## Table of Contents
1. [Overview](#overview)
2. [Kafka Fundamentals](#kafka-fundamentals)
3. [Topic Design](#topic-design)
4. [Producer Configuration](#producer-configuration)
5. [Consumer Groups](#consumer-groups)
6. [Message Patterns](#message-patterns)
7. [Schema Management](#schema-management)
8. [Monitoring Kafka](#monitoring-kafka)
9. [Performance Tuning](#performance-tuning)
10. [Best Practices](#best-practices)

---

## Overview

This document covers the Kafka event streaming architecture, including topic design, producer/consumer patterns, and operational best practices for the DevOps platform.

### Kafka in the Architecture

Kafka serves as the central nervous system for asynchronous, event-driven communication between microservices.

**Key Benefits**:
- **Decoupling**: Services don't need to know about each other
- **Scalability**: Handle millions of events per second
- **Reliability**: Persistent message storage with replication
- **Flexibility**: Multiple consumers can process same events
- **Audit Trail**: Complete event history

### Kafka Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Kafka Cluster                            │
│                                                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │  Broker 1   │    │  Broker 2   │    │  Broker 3   │        │
│  │  Leader for │    │  Leader for │    │  Leader for │        │
│  │  Partition  │    │  Partition  │    │  Partition  │        │
│  │  0, 3, 6    │    │  1, 4, 7    │    │  2, 5, 8    │        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                   │
│  Topics:                                                         │
│  ├─ orders.created     (12 partitions, RF=3)                   │
│  ├─ orders.updated     (12 partitions, RF=3)                   │
│  ├─ users.registered   (6 partitions, RF=3)                    │
│  ├─ users.updated      (6 partitions, RF=3)                    │
│  └─ notifications      (3 partitions, RF=3)                    │
└───────────────────────────────────────────────────────────────────┘

Producers                                              Consumers
┌──────────┐                                        ┌──────────┐
│  Order   │─────┐                          ┌───────│ Order    │
│ Service  │     │                          │       │Processor │
└──────────┘     │                          │       └──────────┘
                 ▼                          │
┌──────────┐  Kafka     Consumer Group 1   │       ┌──────────┐
│  User    │─────┼────────────────────────────────▶│Analytics │
│ Service  │     │                          │       │ Service  │
└──────────┘     │                          │       └──────────┘
                 │                          │
┌──────────┐     │     Consumer Group 2    │       ┌──────────┐
│  Auth    │─────┘                          └───────│  Email   │
│ Service  │                                        │ Service  │
└──────────┘                                        └──────────┘
```

---

## Kafka Fundamentals

### Key Concepts

- **Topic**: Logical channel for messages (e.g., `orders.created`)
- **Partition**: Physical division of a topic for parallelism
- **Producer**: Application that publishes messages to topics
- **Consumer**: Application that subscribes to topics
- **Consumer Group**: Set of consumers working together to consume a topic
- **Offset**: Sequential ID of messages within a partition
- **Replication Factor**: Number of copies of each partition

### Message Flow

```
Producer → Topic (Partition 0) → Consumer Group A (Consumer 1)
        → Topic (Partition 1) → Consumer Group A (Consumer 2)
        → Topic (Partition 2) → Consumer Group A (Consumer 3)
                              → Consumer Group B (Consumer 1-3)
```

---

## Topic Design

### Topic Naming Convention

```
<domain>.<entity>.<event-type>

Examples:
orders.created
orders.updated
orders.cancelled
users.registered
users.updated
payments.processed
notifications.sent
```

### Topic Configuration

```yaml
# topics-config.yaml
topics:
  - name: orders.created
    partitions: 12
    replication-factor: 3
    config:
      retention.ms: 604800000  # 7 days
      compression.type: lz4
      min.insync.replicas: 2
  
  - name: orders.updated
    partitions: 12
    replication-factor: 3
    config:
      retention.ms: 604800000
      compression.type: lz4
      min.insync.replicas: 2
  
  - name: users.registered
    partitions: 6
    replication-factor: 3
    config:
      retention.ms: 2592000000  # 30 days
      compression.type: lz4
      min.insync.replicas: 2
  
  - name: notifications
    partitions: 3
    replication-factor: 3
    config:
      retention.ms: 86400000  # 1 day
      compression.type: lz4
      min.insync.replicas: 2
```

### Creating Topics

```bash
# Using kafka-topics.sh
kafka-topics.sh --create \
  --bootstrap-server kafka.prod.internal:9092 \
  --topic orders.created \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config compression.type=lz4 \
  --config min.insync.replicas=2

# Using Strimzi KafkaTopic CRD
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders.created
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 604800000
    compression.type: lz4
    min.insync.replicas: 2
```

---

## Producer Configuration

### KafkaJS Producer Setup

```typescript
// kafka-producer.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy, Logger } from '@nestjs/common';
import { Kafka, Producer, ProducerRecord, RecordMetadata } from 'kafkajs';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class KafkaProducerService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(KafkaProducerService.name);
  private kafka: Kafka;
  private producer: Producer;

  constructor(private configService: ConfigService) {
    this.kafka = new Kafka({
      clientId: this.configService.get('KAFKA_CLIENT_ID'),
      brokers: this.configService.get('KAFKA_BROKERS').split(','),
      ssl: this.configService.get('NODE_ENV') === 'production',
      retry: {
        initialRetryTime: 100,
        retries: 8,
      },
    });

    this.producer = this.kafka.producer({
      allowAutoTopicCreation: false,
      transactionTimeout: 30000,
      idempotent: true,
      maxInFlightRequests: 5,
      compression: 1, // GZIP
    });
  }

  async onModuleInit() {
    await this.producer.connect();
    this.logger.log('Kafka producer connected');
  }

  async onModuleDestroy() {
    await this.producer.disconnect();
    this.logger.log('Kafka producer disconnected');
  }

  async send(record: ProducerRecord): Promise<RecordMetadata[]> {
    try {
      const result = await this.producer.send(record);
      this.logger.debug(`Message sent to ${record.topic}`);
      return result;
    } catch (error) {
      this.logger.error(`Failed to send message to ${record.topic}:`, error);
      throw error;
    }
  }

  async sendBatch(records: ProducerRecord[]): Promise<RecordMetadata[][]> {
    try {
      const result = await this.producer.sendBatch({
        topicMessages: records,
      });
      this.logger.debug(`Batch of ${records.length} messages sent`);
      return result;
    } catch (error) {
      this.logger.error('Failed to send batch:', error);
      throw error;
    }
  }
}
```

### Message Publishing Example

```typescript
// orders.service.ts
async createOrder(createOrderDto: CreateOrderDto): Promise<Order> {
  const order = await this.ordersRepository.save(createOrderDto);

  // Publish event to Kafka
  await this.kafkaProducer.send({
    topic: 'orders.created',
    messages: [
      {
        key: order.id.toString(), // Partition by order ID
        value: JSON.stringify({
          orderId: order.id,
          userId: order.userId,
          amount: order.amount,
          items: order.items,
          status: order.status,
          createdAt: order.createdAt.toISOString(),
        }),
        headers: {
          'event-type': 'order.created',
          'event-version': '1.0',
          'correlation-id': this.generateCorrelationId(),
        },
      },
    ],
  });

  return order;
}
```

---

## Consumer Groups

### Consumer Configuration

```typescript
// kafka-consumer.service.ts
import { Injectable, OnModuleInit, Logger } from '@nestjs/common';
import { Kafka, Consumer, EachMessagePayload } from 'kafkajs';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class KafkaConsumerService implements OnModuleInit {
  private readonly logger = new Logger(KafkaConsumerService.name);
  private kafka: Kafka;
  private consumer: Consumer;

  constructor(private configService: ConfigService) {
    this.kafka = new Kafka({
      clientId: this.configService.get('KAFKA_CLIENT_ID'),
      brokers: this.configService.get('KAFKA_BROKERS').split(','),
      ssl: this.configService.get('NODE_ENV') === 'production',
    });

    this.consumer = this.kafka.consumer({
      groupId: this.configService.get('KAFKA_CONSUMER_GROUP'),
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
      maxWaitTimeInMs: 100,
      retry: {
        initialRetryTime: 100,
        retries: 8,
      },
    });
  }

  async onModuleInit() {
    await this.consumer.connect();
    await this.consumer.subscribe({
      topics: ['orders.created', 'orders.updated'],
      fromBeginning: false,
    });

    await this.consumer.run({
      eachMessage: async (payload: EachMessagePayload) => {
        await this.handleMessage(payload);
      },
    });

    this.logger.log('Kafka consumer started');
  }

  private async handleMessage(payload: EachMessagePayload) {
    const { topic, partition, message } = payload;
    
    try {
      const value = JSON.parse(message.value.toString());
      
      this.logger.debug(
        `Received message from ${topic}[${partition}] offset: ${message.offset}`,
      );

      // Process message based on topic
      switch (topic) {
        case 'orders.created':
          await this.handleOrderCreated(value);
          break;
        case 'orders.updated':
          await this.handleOrderUpdated(value);
          break;
        default:
          this.logger.warn(`Unknown topic: ${topic}`);
      }
    } catch (error) {
      this.logger.error(`Error processing message from ${topic}:`, error);
      // Implement dead letter queue or retry logic
    }
  }

  private async handleOrderCreated(data: any) {
    // Business logic for order created event
    this.logger.log(`Processing order created: ${data.orderId}`);
  }

  private async handleOrderUpdated(data: any) {
    // Business logic for order updated event
    this.logger.log(`Processing order updated: ${data.orderId}`);
  }
}
```

---

## Message Patterns

### Event Envelope Pattern

```typescript
interface EventEnvelope<T> {
  eventId: string;
  eventType: string;
  eventVersion: string;
  timestamp: string;
  correlationId: string;
  payload: T;
  metadata?: Record<string, any>;
}

const orderCreatedEvent: EventEnvelope<OrderCreated> = {
  eventId: uuid(),
  eventType: 'order.created',
  eventVersion: '1.0',
  timestamp: new Date().toISOString(),
  correlationId: correlationId,
  payload: {
    orderId: order.id,
    userId: order.userId,
    amount: order.amount,
    items: order.items,
  },
  metadata: {
    source: 'order-service',
    region: 'us-east-1',
  },
};
```

### Partition Key Strategy

```typescript
// Strategy 1: Hash by entity ID (orders)
const orderKey = order.id.toString();

// Strategy 2: Hash by user ID (for user-specific ordering)
const userKey = order.userId.toString();

// Strategy 3: Composite key
const compositeKey = `${order.userId}:${order.id}`;

// Send message with key
await producer.send({
  topic: 'orders.created',
  messages: [{
    key: orderKey, // Ensures all messages for same order go to same partition
    value: JSON.stringify(event),
  }],
});
```

---

## Monitoring Kafka

### Prometheus Metrics

```typescript
// kafka-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Counter, Histogram, register } from 'prom-client';

@Injectable()
export class KafkaMetricsService {
  private readonly messagesProduced = new Counter({
    name: 'kafka_messages_produced_total',
    help: 'Total number of messages produced',
    labelNames: ['topic'],
  });

  private readonly messagesConsumed = new Counter({
    name: 'kafka_messages_consumed_total',
    help: 'Total number of messages consumed',
    labelNames: ['topic', 'consumer_group'],
  });

  private readonly messageProcessingDuration = new Histogram({
    name: 'kafka_message_processing_duration_seconds',
    help: 'Message processing duration',
    labelNames: ['topic'],
    buckets: [0.1, 0.5, 1, 2, 5],
  });

  recordMessageProduced(topic: string) {
    this.messagesProduced.inc({ topic });
  }

  recordMessageConsumed(topic: string, consumerGroup: string) {
    this.messagesConsumed.inc({ topic, consumer_group: consumerGroup });
  }

  recordProcessingDuration(topic: string, duration: number) {
    this.messageProcessingDuration.observe({ topic }, duration);
  }
}
```

### Kafka Exporter for Prometheus

```yaml
# kafka-exporter-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-exporter
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-exporter
  template:
    metadata:
      labels:
        app: kafka-exporter
    spec:
      containers:
      - name: kafka-exporter
        image: danielqsj/kafka-exporter:latest
        args:
        - --kafka.server=kafka-cluster-kafka-bootstrap:9092
        ports:
        - containerPort: 9308
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-exporter
  namespace: kafka
  labels:
    app: kafka-exporter
spec:
  ports:
  - port: 9308
    name: metrics
  selector:
    app: kafka-exporter
```

---

## Best Practices

### 1. Producer Best Practices

✅ **Do**:
- Use idempotent producers (`idempotent: true`)
- Set appropriate `acks` level (all for critical data)
- Implement retry logic with exponential backoff
- Use compression (lz4 or snappy)
- Batch messages when possible
- Use meaningful partition keys

❌ **Don't**:
- Don't create topics automatically
- Don't send large messages (>1MB)
- Don't ignore send errors
- Don't use synchronous sends in hot paths

### 2. Consumer Best Practices

✅ **Do**:
- Process messages idempotently
- Commit offsets only after successful processing
- Handle poison messages (dead letter queue)
- Monitor consumer lag
- Use appropriate session timeout
- Scale consumers to match partition count

❌ **Don't**:
- Don't auto-commit offsets
- Don't block consumer loop
- Don't process messages out of order within partition
- Don't ignore deserialization errors

### 3. Topic Design Best Practices

✅ **Do**:
- Use clear naming conventions
- Set appropriate retention periods
- Configure replication factor ≥ 3
- Set `min.insync.replicas` ≥ 2
- Plan partition count based on throughput
- Enable compression

❌ **Don't**:
- Don't use too many partitions (overhead)
- Don't use too few partitions (bottleneck)
- Don't change partition count frequently
- Don't store large payloads

---

[← Back to Application Architecture](./06-application-architecture.md) | [Next: Terraform Structure →](./08-terraform-structure.md)
