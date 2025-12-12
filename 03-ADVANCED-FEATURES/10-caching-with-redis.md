# Module 9: Caching with Redis

## 📚 Table of Contents
1. [Overview](#overview)
2. [Redis Installation & Setup](#redis-installation--setup)
3. [Connecting NestJS to Redis](#connecting-nestjs-to-redis)
4. [Basic Caching Operations](#basic-caching-operations)
5. [Cache Strategies](#cache-strategies)
6. [Cache Invalidation](#cache-invalidation)
7. [Session Management](#session-management)
8. [Pub/Sub with Redis](#pubsub-with-redis)
9. [Performance Optimization](#performance-optimization)
10. [Best Practices](#best-practices)
11. [Mini Project: High-Performance API](#mini-project-high-performance-api)

---

## Overview

Redis is an in-memory data structure store that can be used as a database, cache, or message broker. In NestJS applications, Redis is primarily used for caching to improve performance and reduce database load.

### Key Benefits:
- **Fast**: In-memory storage for sub-millisecond response times
- **Versatile**: Supports strings, hashes, lists, sets, sorted sets
- **Scalable**: Can be clustered or used as primary database
- **Pub/Sub**: Built-in publish-subscribe messaging

---

## Redis Installation & Setup

### macOS

```bash
brew install redis
brew services start redis
```

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

### Windows

Download from: https://redis.io/download

### Docker

```bash
docker run -d -p 6379:6379 --name redis redis:7-alpine
```

### Verify Installation

```bash
redis-cli ping
# Should return: PONG
```

---

## Connecting NestJS to Redis

### Install Dependencies

```bash
npm install ioredis
npm install @nestjs/cache-manager cache-manager cache-manager-redis-store
npm install -D @types/cache-manager-redis-store
```

### Configure Redis Module

```typescript
// src/redis/redis.module.ts
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { ConfigModule, ConfigService } from '@nestjs/config';
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        store: redisStore,
        host: configService.get('REDIS_HOST', 'localhost'),
        port: configService.get('REDIS_PORT', 6379),
        ttl: configService.get('CACHE_TTL', 300), // 5 minutes default
      }),
      inject: [ConfigService],
    }),
  ],
  exports: [CacheModule],
})
export class RedisModule {}
```

### Using ioredis Directly

```typescript
// src/redis/redis.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import Redis from 'ioredis';

@Injectable()
export class RedisService implements OnModuleInit, OnModuleDestroy {
  private client: Redis;

  constructor(private configService: ConfigService) {
    this.client = new Redis({
      host: this.configService.get('REDIS_HOST', 'localhost'),
      port: this.configService.get('REDIS_PORT', 6379),
      retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
      },
    });
  }

  onModuleInit() {
    this.client.on('connect', () => {
      console.log('Connected to Redis');
    });

    this.client.on('error', (err) => {
      console.error('Redis error:', err);
    });
  }

  onModuleDestroy() {
    this.client.quit();
  }

  getClient(): Redis {
    return this.client;
  }
}
```

---

## Basic Caching Operations

### Using Cache Manager

Cache Manager is NestJS's built-in caching abstraction. It provides a simple API for caching that works with different cache stores (Redis, in-memory, etc.). This is the recommended approach for most applications.

**Cache-Aside Pattern:** This example demonstrates the "cache-aside" (also called "lazy loading") pattern, which is the most common caching strategy:
1. Check cache first
2. If found (cache hit), return cached data
3. If not found (cache miss), fetch from database
4. Store result in cache for future requests
5. Return the data

**Benefits:**
- Reduces database load significantly
- Improves response times for frequently accessed data
- Simple to implement and understand

```typescript
// src/products/products.service.ts
import { Injectable, Inject, CACHE_MANAGER } from '@nestjs/common';
import { Cache } from 'cache-manager';

@Injectable()
export class ProductsService {
  constructor(
    // @Inject(CACHE_MANAGER) injects the cache manager instance
    // The Cache type provides type safety for cache operations
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    @InjectRepository(Product)
    private productsRepository: Repository<Product>,
  ) {}

  async findAll(): Promise<Product[]> {
    // Cache key: Use a descriptive, unique key
    // Convention: 'resource:identifier' or 'resource:all' for lists
    const cacheKey = 'products:all';
    
    // Step 1: Try to get from cache first (cache hit)
    // get<T>() returns the cached value or null if not found
    // Generic type <Product[]> provides TypeScript type safety
    const cached = await this.cacheManager.get<Product[]>(cacheKey);
    
    // If data exists in cache, return it immediately
    // This avoids the expensive database query
    if (cached) {
      return cached; // Cache hit - return immediately
    }

    // Step 2: Cache miss - fetch from database
    // This is the expensive operation we're trying to avoid
    const products = await this.productsRepository.find();
    
    // Step 3: Store in cache for future requests
    // set() stores the data with an optional TTL (Time To Live)
    // ttl: 300 means cache expires in 300 seconds (5 minutes)
    // After 5 minutes, the next request will fetch from database again
    await this.cacheManager.set(cacheKey, products, { ttl: 300 }); // 5 minutes

    // Step 4: Return the fresh data
    return products;
  }

  async findOne(id: number): Promise<Product> {
    const cacheKey = `product:${id}`;
    
    const cached = await this.cacheManager.get<Product>(cacheKey);
    if (cached) {
      return cached;
    }

    const product = await this.productsRepository.findOne({ where: { id } });
    
    if (product) {
      await this.cacheManager.set(cacheKey, product, { ttl: 300 });
    }

    return product;
  }

  async remove(id: number): Promise<void> {
    await this.productsRepository.delete(id);
    
    // Invalidate cache
    await this.cacheManager.del(`product:${id}`);
    await this.cacheManager.del('products:all');
  }
}
```

### Using ioredis Directly

```typescript
// src/products/products.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class ProductsService {
  private redis: Redis;

  constructor(
    private redisService: RedisService,
    @InjectRepository(Product)
    private productsRepository: Repository<Product>,
  ) {
    this.redis = this.redisService.getClient();
  }

  async findAll(): Promise<Product[]> {
    const cacheKey = 'products:all';
    
    // Try to get from cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // Fetch from database
    const products = await this.productsRepository.find();
    
    // Store in cache (expires in 5 minutes)
    await this.redis.setex(cacheKey, 300, JSON.stringify(products));

    return products;
  }

  async findOne(id: number): Promise<Product> {
    const cacheKey = `product:${id}`;
    
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    const product = await this.productsRepository.findOne({ where: { id } });
    
    if (product) {
      await this.redis.setex(cacheKey, 300, JSON.stringify(product));
    }

    return product;
  }
}
```

---

## Cache Strategies

### Cache-Aside (Lazy Loading)

```typescript
async findOne(id: number): Promise<Product> {
  // 1. Check cache
  const cached = await this.getFromCache(id);
  if (cached) return cached;

  // 2. If not in cache, load from database
  const product = await this.loadFromDatabase(id);

  // 3. Store in cache for future requests
  if (product) {
    await this.setInCache(id, product);
  }

  return product;
}
```

### Write-Through

```typescript
async create(createProductDto: CreateProductDto): Promise<Product> {
  // 1. Write to database
  const product = await this.productsRepository.save(createProductDto);

  // 2. Immediately write to cache
  await this.setInCache(product.id, product);

  return product;
}
```

### Write-Behind (Write-Back)

```typescript
async update(id: number, updateDto: UpdateProductDto): Promise<Product> {
  // 1. Update cache first (fast)
  const cached = await this.getFromCache(id);
  const updated = { ...cached, ...updateDto };
  await this.setInCache(id, updated);

  // 2. Queue database update (can be async)
  this.queueDatabaseUpdate(id, updateDto);

  return updated;
}
```

---

## Cache Invalidation

### Manual Invalidation

```typescript
async update(id: number, updateDto: UpdateProductDto): Promise<Product> {
  // Update database
  await this.productsRepository.update(id, updateDto);
  const product = await this.productsRepository.findOne({ where: { id } });

  // Invalidate specific cache
  await this.cacheManager.del(`product:${id}`);
  
  // Invalidate list cache
  await this.cacheManager.del('products:all');
  
  // Invalidate related caches
  await this.invalidateRelatedCaches(product);

  return product;
}

private async invalidateRelatedCaches(product: Product): Promise<void> {
  // Invalidate category cache
  if (product.categoryId) {
    await this.cacheManager.del(`category:${product.categoryId}:products`);
  }

  // Invalidate search cache
  await this.cacheManager.del('products:search:*');
}
```

### Pattern-Based Invalidation

```typescript
async invalidatePattern(pattern: string): Promise<void> {
  const keys = await this.redis.keys(pattern);
  
  if (keys.length > 0) {
    await this.redis.del(...keys);
  }
}

// Usage
await this.invalidatePattern('products:*');
await this.invalidatePattern('user:*:orders');
```

### TTL-Based Expiration

```typescript
// Set cache with TTL
await this.redis.setex('key', 300, JSON.stringify(data)); // 5 minutes

// Or with cache-manager
await this.cacheManager.set('key', data, { ttl: 300 });
```

---

## Session Management

### Store Sessions in Redis

```typescript
// src/auth/session.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class SessionService {
  private redis: Redis;

  constructor(private redisService: RedisService) {
    this.redis = this.redisService.getClient();
  }

  async createSession(userId: number, sessionData: any): Promise<string> {
    const sessionId = crypto.randomUUID();
    const sessionKey = `session:${sessionId}`;
    
    await this.redis.setex(
      sessionKey,
      3600, // 1 hour
      JSON.stringify({
        userId,
        ...sessionData,
        createdAt: new Date(),
      }),
    );

    return sessionId;
  }

  async getSession(sessionId: string): Promise<any> {
    const sessionKey = `session:${sessionId}`;
    const session = await this.redis.get(sessionKey);
    
    return session ? JSON.parse(session) : null;
  }

  async updateSession(sessionId: string, data: any): Promise<void> {
    const sessionKey = `session:${sessionId}`;
    const existing = await this.getSession(sessionId);
    
    if (existing) {
      await this.redis.setex(
        sessionKey,
        3600,
        JSON.stringify({ ...existing, ...data }),
      );
    }
  }

  async deleteSession(sessionId: string): Promise<void> {
    const sessionKey = `session:${sessionId}`;
    await this.redis.del(sessionKey);
  }

  async extendSession(sessionId: string, ttl: number): Promise<void> {
    const sessionKey = `session:${sessionId}`;
    await this.redis.expire(sessionKey, ttl);
  }
}
```

---

## Pub/Sub with Redis

### Publisher Service

```typescript
// src/redis/redis-pub.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from './redis.service';

@Injectable()
export class RedisPubService {
  private publisher: Redis;

  constructor(private redisService: RedisService) {
    this.publisher = new Redis({
      host: this.redisService.getConfig().host,
      port: this.redisService.getConfig().port,
    });
  }

  async publish(channel: string, message: any): Promise<void> {
    await this.publisher.publish(channel, JSON.stringify(message));
  }

  onModuleDestroy() {
    this.publisher.quit();
  }
}
```

### Subscriber Service

```typescript
// src/redis/redis-sub.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { RedisService } from './redis.service';
import Redis from 'ioredis';

@Injectable()
export class RedisSubService implements OnModuleInit {
  private subscriber: Redis;

  constructor(private redisService: RedisService) {
    this.subscriber = new Redis({
      host: this.redisService.getConfig().host,
      port: this.redisService.getConfig().port,
    });
  }

  onModuleInit() {
    // Subscribe to channels
    this.subscriber.subscribe('product:created', 'product:updated', 'product:deleted');
    
    // Handle messages
    this.subscriber.on('message', (channel, message) => {
      const data = JSON.parse(message);
      this.handleMessage(channel, data);
    });
  }

  private handleMessage(channel: string, data: any): void {
    switch (channel) {
      case 'product:created':
        this.handleProductCreated(data);
        break;
      case 'product:updated':
        this.handleProductUpdated(data);
        break;
      case 'product:deleted':
        this.handleProductDeleted(data);
        break;
    }
  }

  private handleProductCreated(data: any): void {
    // Invalidate cache
    console.log('Product created, invalidating cache');
  }

  private handleProductUpdated(data: any): void {
    console.log('Product updated, invalidating cache');
  }

  private handleProductDeleted(data: any): void {
    console.log('Product deleted, invalidating cache');
  }
}
```

---

## Performance Optimization

### Connection Pooling

```typescript
// Use connection pooling for high-traffic applications
const cluster = new Redis.Cluster([
  {
    host: '127.0.0.1',
    port: 6379,
  },
]);

// Or use sentinel for high availability
const sentinel = new Redis({
  sentinels: [
    { host: '127.0.0.1', port: 26379 },
  ],
  name: 'mymaster',
});
```

### Caching with Compression

```typescript
import * as zlib from 'zlib';
import { promisify } from 'util';

const gzip = promisify(zlib.gzip);
const gunzip = promisify(zlib.gunzip);

async setCompressed(key: string, value: any, ttl: number): Promise<void> {
  const json = JSON.stringify(value);
  const compressed = await gzip(json);
  await this.redis.setex(key, ttl, compressed.toString('base64'));
}

async getCompressed(key: string): Promise<any> {
  const compressed = await this.redis.get(key);
  if (!compressed) return null;
  
  const buffer = Buffer.from(compressed, 'base64');
  const decompressed = await gunzip(buffer);
  return JSON.parse(decompressed.toString());
}
```

---

## Best Practices

### 1. Choose Appropriate TTL

```typescript
// Short TTL for frequently changing data
await cache.set('products:hot', data, { ttl: 60 }); // 1 minute

// Longer TTL for stable data
await cache.set('categories', data, { ttl: 3600 }); // 1 hour
```

### 2. Use Meaningful Cache Keys

```typescript
// Good
'product:123'
'user:456:orders'
'category:789:products:page:1'

// Bad
'data1'
'cache_xyz'
```

### 3. Handle Cache Misses Gracefully

```typescript
async findOne(id: number): Promise<Product> {
  try {
    const cached = await this.getFromCache(id);
    if (cached) return cached;
  } catch (error) {
    // If cache fails, fallback to database
    console.error('Cache error:', error);
  }

  return await this.loadFromDatabase(id);
}
```

### 4. Monitor Cache Hit Rates

```typescript
private cacheHits = 0;
private cacheMisses = 0;

async findOne(id: number): Promise<Product> {
  const cached = await this.getFromCache(id);
  
  if (cached) {
    this.cacheHits++;
    return cached;
  }

  this.cacheMisses++;
  const product = await this.loadFromDatabase(id);
  await this.setInCache(id, product);
  
  return product;
}

getCacheStats() {
  const total = this.cacheHits + this.cacheMisses;
  return {
    hits: this.cacheHits,
    misses: this.cacheMisses,
    hitRate: total > 0 ? (this.cacheHits / total) * 100 : 0,
  };
}
```

### 5. Set Memory Limits

```bash
# In redis.conf
maxmemory 256mb
maxmemory-policy allkeys-lru
```

---

## Mini Project: High-Performance API

Build an API with Redis caching that:

1. Caches product listings
2. Implements cache invalidation
3. Uses Redis for session management
4. Publishes cache invalidation events
5. Tracks cache hit rates

### Features:
- Product list caching with pagination
- Individual product caching
- Category-based cache invalidation
- Real-time cache stats endpoint
- Session-based rate limiting

---

## Next Steps

✅ Understand Redis caching patterns  
✅ Implement cache invalidation strategies  
✅ Use Redis for session management  
✅ Move to Module 10: Background Jobs & Queues  

---

*You now know how to optimize your APIs with Redis! ⚡*

