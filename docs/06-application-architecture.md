# 06. Application Architecture

## Table of Contents
1. [Overview](#overview)
2. [Microservices Design](#microservices-design)
3. [NestJS Application Structure](#nestjs-application-structure)
4. [Kafka Integration](#kafka-integration)
5. [API Gateway Pattern](#api-gateway-pattern)
6. [Service Communication](#service-communication)
7. [Database Patterns](#database-patterns)
8. [Error Handling](#error-handling)
9. [Health Checks](#health-checks)
10. [Deployment Patterns](#deployment-patterns)

---

## Overview

This document describes the application architecture for microservices built with NestJS, integrated with Kafka for event-driven communication, running on Kubernetes.

### Application Stack

- **Runtime**: Node.js 20 LTS
- **Framework**: NestJS
- **Event Streaming**: Apache Kafka (KafkaJS client)
- **Database ORM**: TypeORM / Prisma
- **API Documentation**: Swagger/OpenAPI
- **Logging**: Winston / Pino
- **Metrics**: Prometheus client

### Microservices Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                            External Clients                          │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Application Load Balancer                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
                 ┌───────────────────────┐
                 │    API Gateway        │
                 │    (NestJS)           │
                 │  - Authentication     │
                 │  - Rate Limiting      │
                 │  - Request Routing    │
                 └───────────┬───────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
  ┌──────────┐        ┌──────────┐       ┌──────────┐
  │  Auth    │        │  Order   │       │  User    │
  │ Service  │        │ Service  │       │ Service  │
  │ (NestJS) │        │ (NestJS) │       │ (NestJS) │
  └────┬─────┘        └────┬─────┘       └────┬─────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │  Kafka Cluster  │
                  │                 │
                  │  Topics:        │
                  │  - orders.*     │
                  │  - users.*      │
                  │  - auth.*       │
                  └────────┬────────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐    ┌─────────────┐
│  Order      │     │ Notification│    │  Analytics  │
│  Processor  │     │  Service    │    │  Service    │
│  (Consumer) │     │  (Consumer) │    │  (Consumer) │
└─────────────┘     └─────────────┘    └─────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
   Database           Email/SMS          Time Series DB
```

---

## Microservices Design

### Service Decomposition

**Core Services**:
- **API Gateway**: Entry point, authentication, routing
- **Auth Service**: User authentication, JWT management
- **User Service**: User profile management
- **Order Service**: Order processing, business logic
- **Notification Service**: Email/SMS notifications (Kafka consumer)
- **Analytics Service**: Data aggregation (Kafka consumer)

### Service Communication Patterns

```
Synchronous (REST)       Asynchronous (Kafka)
─────────────────        ────────────────────
API Gateway → Auth       Order Service → Kafka → Notification Service
API Gateway → User       User Service  → Kafka → Analytics Service
API Gateway → Order
```

---

## NestJS Application Structure

### Project Structure

```
api-gateway/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── guards/
│   │   │   └── jwt.guard.ts
│   │   └── strategies/
│   │       └── jwt.strategy.ts
│   ├── orders/
│   │   ├── orders.module.ts
│   │   ├── orders.controller.ts
│   │   ├── orders.service.ts
│   │   └── dto/
│   │       ├── create-order.dto.ts
│   │       └── update-order.dto.ts
│   ├── health/
│   │   ├── health.controller.ts
│   │   └── health.module.ts
│   └── common/
│       ├── filters/
│       ├── interceptors/
│       └── middleware/
├── test/
├── package.json
├── tsconfig.json
└── Dockerfile
```

### Main Application Setup

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import helmet from 'helmet';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log', 'debug'],
  });

  const logger = new Logger('Bootstrap');

  // Security
  app.use(helmet());
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(','),
    credentials: true,
  });

  // Validation
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  // Swagger documentation
  if (process.env.NODE_ENV !== 'production') {
    const config = new DocumentBuilder()
      .setTitle('API Gateway')
      .setDescription('Microservices API Gateway')
      .setVersion('1.0')
      .addBearerAuth()
      .build();
    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup('api', app, document);
  }

  // Graceful shutdown
  app.enableShutdownHooks();

  const port = process.env.PORT || 3000;
  await app.listen(port, '0.0.0.0');
  logger.log(`Application is running on: http://localhost:${port}`);
}

bootstrap();
```

### App Module Configuration

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { PrometheusModule } from '@willsoto/nestjs-prometheus';
import { HealthModule } from './health/health.module';
import { AuthModule } from './auth/auth.module';
import { OrdersModule } from './orders/orders.module';
import { KafkaModule } from './kafka/kafka.module';

@Module({
  imports: [
    // Configuration
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: `.env.${process.env.NODE_ENV || 'development'}`,
    }),

    // Database
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT, 10),
      username: process.env.DB_USERNAME,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME,
      autoLoadEntities: true,
      synchronize: process.env.NODE_ENV === 'development',
      ssl: process.env.NODE_ENV === 'production',
    }),

    // Prometheus metrics
    PrometheusModule.register(),

    // Feature modules
    HealthModule,
    AuthModule,
    OrdersModule,
    KafkaModule,
  ],
})
export class AppModule {}
```

---

## Kafka Integration

### Kafka Module Setup

```typescript
// src/kafka/kafka.module.ts
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { KafkaService } from './kafka.service';

@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        name: 'KAFKA_SERVICE',
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => ({
          transport: Transport.KAFKA,
          options: {
            client: {
              clientId: configService.get('KAFKA_CLIENT_ID'),
              brokers: configService.get('KAFKA_BROKERS').split(','),
              ssl: configService.get('NODE_ENV') === 'production',
            },
            consumer: {
              groupId: configService.get('KAFKA_CONSUMER_GROUP'),
            },
            producer: {
              allowAutoTopicCreation: false,
              transactionTimeout: 30000,
            },
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
  providers: [KafkaService],
  exports: [KafkaService],
})
export class KafkaModule {}
```

### Kafka Producer Service

```typescript
// src/kafka/kafka.service.ts
import { Injectable, Inject, OnModuleDestroy } from '@nestjs/common';
import { ClientKafka } from '@nestjs/microservices';
import { Logger } from '@nestjs/common';

@Injectable()
export class KafkaService implements OnModuleDestroy {
  private readonly logger = new Logger(KafkaService.name);

  constructor(
    @Inject('KAFKA_SERVICE') private readonly kafkaClient: ClientKafka,
  ) {}

  async onModuleInit() {
    await this.kafkaClient.connect();
    this.logger.log('Kafka client connected');
  }

  async onModuleDestroy() {
    await this.kafkaClient.close();
    this.logger.log('Kafka client disconnected');
  }

  async emit(topic: string, message: any): Promise<void> {
    try {
      this.kafkaClient.emit(topic, message);
      this.logger.debug(`Message sent to topic ${topic}`);
    } catch (error) {
      this.logger.error(`Failed to send message to ${topic}:`, error);
      throw error;
    }
  }
}
```

### Order Service Example

```typescript
// src/orders/orders.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Order } from './entities/order.entity';
import { CreateOrderDto } from './dto/create-order.dto';
import { KafkaService } from '../kafka/kafka.service';

@Injectable()
export class OrdersService {
  private readonly logger = new Logger(OrdersService.name);

  constructor(
    @InjectRepository(Order)
    private ordersRepository: Repository<Order>,
    private kafkaService: KafkaService,
  ) {}

  async create(createOrderDto: CreateOrderDto, userId: string): Promise<Order> {
    // Create order in database
    const order = this.ordersRepository.create({
      ...createOrderDto,
      userId,
      status: 'pending',
    });
    const savedOrder = await this.ordersRepository.save(order);

    // Emit event to Kafka
    await this.kafkaService.emit('orders.created', {
      orderId: savedOrder.id,
      userId: savedOrder.userId,
      amount: savedOrder.amount,
      items: savedOrder.items,
      createdAt: savedOrder.createdAt,
    });

    this.logger.log(`Order created: ${savedOrder.id}`);
    return savedOrder;
  }

  async findAll(userId: string): Promise<Order[]> {
    return this.ordersRepository.find({
      where: { userId },
      order: { createdAt: 'DESC' },
    });
  }
}
```

### Kafka Consumer Service

```typescript
// notification-service/src/main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.KAFKA,
      options: {
        client: {
          clientId: 'notification-service',
          brokers: process.env.KAFKA_BROKERS.split(','),
        },
        consumer: {
          groupId: 'notification-consumer-group',
        },
      },
    },
  );

  await app.listen();
}
bootstrap();
```

```typescript
// notification-service/src/notifications/notifications.controller.ts
import { Controller, Logger } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';
import { NotificationsService } from './notifications.service';

@Controller()
export class NotificationsController {
  private readonly logger = new Logger(NotificationsController.name);

  constructor(private readonly notificationsService: NotificationsService) {}

  @MessagePattern('orders.created')
  async handleOrderCreated(@Payload() message: any) {
    this.logger.log(`Received order.created event: ${JSON.stringify(message)}`);
    
    await this.notificationsService.sendOrderConfirmation({
      orderId: message.orderId,
      userId: message.userId,
      amount: message.amount,
    });
  }

  @MessagePattern('users.registered')
  async handleUserRegistered(@Payload() message: any) {
    this.logger.log(`Received user.registered event: ${JSON.stringify(message)}`);
    
    await this.notificationsService.sendWelcomeEmail({
      userId: message.userId,
      email: message.email,
      name: message.name,
    });
  }
}
```

---

## Health Checks

### Health Check Implementation

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheckService,
  HttpHealthIndicator,
  TypeOrmHealthIndicator,
  HealthCheck,
  MicroserviceHealthIndicator,
} from '@nestjs/terminus';
import { Transport } from '@nestjs/microservices';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
    private db: TypeOrmHealthIndicator,
    private microservice: MicroserviceHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () =>
        this.microservice.pingCheck('kafka', {
          transport: Transport.KAFKA,
          options: {
            client: {
              clientId: 'health-check',
              brokers: process.env.KAFKA_BROKERS.split(','),
            },
          },
        }),
    ]);
  }

  @Get('ready')
  @HealthCheck()
  readiness() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }

  @Get('live')
  liveness() {
    return { status: 'ok' };
  }
}
```

### Kubernetes Probes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  template:
    spec:
      containers:
      - name: api-gateway
        image: ${ECR_REGISTRY}/api-gateway:${IMAGE_TAG}
        ports:
        - containerPort: 3000
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

---

## Deployment Patterns

### Dockerfile

```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production image
FROM node:20-alpine

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./

# Switch to non-root user
USER nodejs

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

---

[← Back to IAM and RBAC](./05-iam-rbac.md) | [Next: Kafka Architecture →](./07-kafka-architecture.md)
