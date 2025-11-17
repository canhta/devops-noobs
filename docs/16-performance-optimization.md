# 16. Performance Optimization

## Table of Contents
1. [Overview](#overview)
2. [Application Performance](#application-performance)
3. [Database Optimization](#database-optimization)
4. [Caching Strategies](#caching-strategies)
5. [Network Optimization](#network-optimization)
6. [Kubernetes Optimization](#kubernetes-optimization)
7. [Cost vs Performance](#cost-vs-performance)
8. [Load Testing](#load-testing)
9. [Monitoring Performance](#monitoring-performance)
10. [Best Practices](#best-practices)

---

## Overview

This document covers performance optimization strategies across the entire stack, from application code to infrastructure.

### Performance Goals

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| **API Response Time (p95)** | < 200ms | 180ms | ✅ |
| **API Response Time (p99)** | < 500ms | 450ms | ✅ |
| **Database Query Time (p95)** | < 50ms | 45ms | ✅ |
| **Error Rate** | < 0.1% | 0.05% | ✅ |
| **Throughput** | > 1000 req/s | 1200 req/s | ✅ |

---

## Application Performance

### Node.js Optimization

```typescript
// 1. Use connection pooling
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20,  // Maximum pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// 2. Enable HTTP keep-alive
import http from 'http';
import https from 'https';

const httpAgent = new http.Agent({
  keepAlive: true,
  maxSockets: 50,
});

const httpsAgent = new https.Agent({
  keepAlive: true,
  maxSockets: 50,
});

// 3. Use compression
import compression from 'compression';
app.use(compression());

// 4. Implement request timeout
app.use((req, res, next) => {
  req.setTimeout(30000, () => {
    res.status(408).send('Request timeout');
  });
  next();
});
```

### Async Optimization

```typescript
// Bad: Sequential execution
async function getOrders(userId: string) {
  const user = await userService.findById(userId);
  const orders = await orderService.findByUser(userId);
  const products = await productService.findByOrders(orders);
  return { user, orders, products };
}

// Good: Parallel execution
async function getOrders(userId: string) {
  const [user, orders] = await Promise.all([
    userService.findById(userId),
    orderService.findByUser(userId),
  ]);
  
  const products = await productService.findByOrders(orders);
  
  return { user, orders, products };
}

// Better: With error handling
async function getOrders(userId: string) {
  const [userResult, ordersResult] = await Promise.allSettled([
    userService.findById(userId),
    orderService.findByUser(userId),
  ]);
  
  if (userResult.status === 'rejected') {
    throw new Error('User not found');
  }
  
  const orders = ordersResult.status === 'fulfilled' 
    ? ordersResult.value 
    : [];
  
  return { user: userResult.value, orders };
}
```

### NestJS Performance

```typescript
// 1. Use caching interceptor
import { CacheInterceptor, CacheTTL } from '@nestjs/cache-manager';

@Controller('users')
@UseInterceptors(CacheInterceptor)
export class UsersController {
  @Get(':id')
  @CacheTTL(60)  // Cache for 60 seconds
  async findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }
}

// 2. Implement pagination
@Get()
async findAll(
  @Query('page') page: number = 1,
  @Query('limit') limit: number = 10,
) {
  const offset = (page - 1) * limit;
  const [items, total] = await this.repository.findAndCount({
    skip: offset,
    take: limit,
  });
  
  return {
    items,
    meta: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    },
  };
}

// 3. Use data transformation pipes
import { Transform } from 'class-transformer';

export class CreateOrderDto {
  @Transform(({ value }) => value.trim())
  name: string;
  
  @Transform(({ value }) => parseFloat(value))
  amount: number;
}
```

---

## Database Optimization

### Index Strategy

```sql
-- Create indexes on frequently queried columns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite indexes for common query patterns
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- Partial indexes for specific conditions
CREATE INDEX idx_orders_pending ON orders(created_at) 
  WHERE status = 'pending';

-- Check index usage
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Find missing indexes
SELECT 
  schemaname,
  tablename,
  seq_scan,
  seq_tup_read,
  idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 1000 AND idx_scan < seq_scan
ORDER BY seq_tup_read DESC;
```

### Query Optimization

```sql
-- Use EXPLAIN ANALYZE
EXPLAIN ANALYZE
SELECT o.*, u.name 
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
AND o.created_at > NOW() - INTERVAL '7 days';

-- Avoid SELECT *
-- Bad
SELECT * FROM orders;

-- Good
SELECT id, user_id, amount, status FROM orders;

-- Use JOINs instead of subqueries when possible
-- Bad
SELECT * FROM orders 
WHERE user_id IN (SELECT id FROM users WHERE country = 'US');

-- Good
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'US';

-- Use LIMIT for large result sets
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 100 OFFSET 0;
```

### Connection Pooling

```typescript
// TypeORM configuration
import { TypeOrmModule } from '@nestjs/typeorm';

TypeOrmModule.forRoot({
  type: 'postgres',
  host: process.env.DB_HOST,
  port: 5432,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  entities: [__dirname + '/**/*.entity{.ts,.js}'],
  
  // Connection pool settings
  extra: {
    max: 20,  // Maximum connections
    min: 5,   // Minimum connections
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
  },
  
  // Enable query logging in dev
  logging: process.env.NODE_ENV === 'development',
  
  // Statement timeout
  statementTimeout: 30000,  // 30 seconds
});
```

---

## Caching Strategies

### Redis Cache

```typescript
// Redis setup
import { CacheModule } from '@nestjs/cache-manager';
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.register({
      store: redisStore,
      host: process.env.REDIS_HOST,
      port: 6379,
      ttl: 60,  // Default TTL in seconds
      max: 1000,  // Maximum number of items in cache
    }),
  ],
})
export class AppModule {}

// Cache service
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class CacheService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async get<T>(key: string): Promise<T | undefined> {
    return await this.cacheManager.get<T>(key);
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    await this.cacheManager.set(key, value, ttl);
  }

  async del(key: string): Promise<void> {
    await this.cacheManager.del(key);
  }

  async reset(): Promise<void> {
    await this.cacheManager.reset();
  }
}

// Usage in service
@Injectable()
export class UsersService {
  constructor(private cacheService: CacheService) {}

  async findOne(id: string): Promise<User> {
    const cacheKey = `user:${id}`;
    
    // Try cache first
    const cached = await this.cacheService.get<User>(cacheKey);
    if (cached) {
      return cached;
    }
    
    // Fetch from database
    const user = await this.userRepository.findOne({ where: { id } });
    
    // Store in cache
    await this.cacheService.set(cacheKey, user, 300);  // 5 minutes
    
    return user;
  }

  async update(id: string, data: UpdateUserDto): Promise<User> {
    const user = await this.userRepository.update(id, data);
    
    // Invalidate cache
    await this.cacheService.del(`user:${id}`);
    
    return user;
  }
}
```

### HTTP Caching

```typescript
// Add Cache-Control headers
import { Controller, Get, Header } from '@nestjs/common';

@Controller('api')
export class ApiController {
  @Get('public-data')
  @Header('Cache-Control', 'public, max-age=3600')  // Cache for 1 hour
  getPublicData() {
    return this.service.getPublicData();
  }

  @Get('user-data')
  @Header('Cache-Control', 'private, max-age=300')  // Cache for 5 minutes
  getUserData() {
    return this.service.getUserData();
  }

  @Get('real-time')
  @Header('Cache-Control', 'no-cache, no-store, must-revalidate')
  getRealTimeData() {
    return this.service.getRealTimeData();
  }
}
```

---

## Network Optimization

### CDN for Static Assets

```hcl
# Configure CloudFront
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name = aws_s3_bucket.static_assets.bucket_regional_domain_name
    origin_id   = "S3-static-assets"
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-static-assets"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    compress               = true
  }
}
```

### HTTP/2

```yaml
# ALB already supports HTTP/2
# Ensure application supports it
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
```

---

## Kubernetes Optimization

### Resource Requests/Limits

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api-gateway
```

---

## Cost vs Performance

### Resource Right-Sizing

```yaml
# Before optimization
resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
  limits:
    memory: "2Gi"
    cpu: "2000m"

# After optimization (actual usage: 300Mi / 200m)
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "768Mi"
    cpu: "500m"

# Cost savings: ~60% per pod
```

### Spot Instances

```hcl
# Use spot instances for non-critical workloads
resource "aws_eks_node_group" "spot" {
  capacity_type = "SPOT"
  
  instance_types = [
    "t3.medium",
    "t3a.medium",
    "t2.medium"
  ]
  
  scaling_config {
    desired_size = 2
    min_size     = 1
    max_size     = 5
  }
}
```

---

## Load Testing

### k6 Load Test

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp to 200
    { duration: '5m', target: 200 },  // Stay at 200
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://api.example.com/health');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });
  
  sleep(1);
}
```

### Run Load Test

```bash
# Install k6
brew install k6

# Run test
k6 run load-test.js

# Run with custom VUs
k6 run --vus 100 --duration 30s load-test.js

# Output to InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 load-test.js
```

---

## Monitoring Performance

### Application Performance Monitoring (APM)

```typescript
// OpenTelemetry instrumentation
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { PrometheusExporter } from '@opentelemetry/exporter-prometheus';

const sdk = new NodeSDK({
  traceExporter: new PrometheusExporter({ port: 9464 }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

### Custom Metrics

```typescript
import { Counter, Histogram } from 'prom-client';

// Request counter
const httpRequestCounter = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

// Request duration histogram
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
});

// Middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestCounter.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode,
    });
    
    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || req.path,
      },
      duration
    );
  });
  
  next();
});
```

---

## Best Practices

### Performance Checklist

- ✅ Use connection pooling
- ✅ Implement caching (Redis)
- ✅ Add database indexes
- ✅ Enable HTTP/2
- ✅ Use CDN for static assets
- ✅ Compress responses (gzip/brotli)
- ✅ Optimize images
- ✅ Lazy load data
- ✅ Implement pagination
- ✅ Monitor performance metrics
- ✅ Right-size resources
- ✅ Use async operations
- ✅ Avoid N+1 queries
- ✅ Set request timeouts

### Code Review Guidelines

```typescript
// ❌ Bad: Sequential database calls
const users = await User.findAll();
for (const user of users) {
  user.orders = await Order.findByUser(user.id);
}

// ✅ Good: Use JOIN
const users = await User.findAll({
  include: [{ model: Order }],
});

// ❌ Bad: No caching
async function getConfig() {
  return await db.query('SELECT * FROM config');
}

// ✅ Good: Cache config
const config = cache.wrap('config', async () => {
  return await db.query('SELECT * FROM config');
}, { ttl: 300 });

// ❌ Bad: Large payload
res.json(users);  // Returns all fields

// ✅ Good: Return only needed fields
res.json(users.map(u => ({
  id: u.id,
  name: u.name,
  email: u.email,
})));
```

---

[← Back to Autoscaling Strategy](./15-autoscaling-strategy.md) | [Next: Operations Guide →](./17-operations-guide.md)
