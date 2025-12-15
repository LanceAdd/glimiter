# Rate Limiter for GoFrame

[English](README.md) | [中文版本](README_cn.md)

Implementation of rate limiter for GoFrame, supporting both in-memory and Redis-based rate limiting strategies.

## Features

- ✅ **Multiple Implementations**: Supports both in-memory and Redis storage backends
- ✅ **Sliding Window**: Redis uses precise sliding window algorithm
- ✅ **Concurrency Safe**: In-memory version uses CAS atomic operations, Redis version uses Lua scripts
- ✅ **Flexible Configuration**: Supports custom key generation and error handling
- ✅ **Standardized**: Response headers compliant with RFC 6585
- ✅ **High Performance**: No extra overhead for in-memory version, atomic operations for Redis version

## Installation

```bash
go get github.com/LanceAdd/glimiter@latest
```

## Quick Start

### 1. Basic Usage

```go
import "github.com/LanceAdd/glimiter"

// Create limiter: up to 100 requests per minute
limiter := glimiter.NewMemoryLimiter(100, time.Minute)

// Check if request is allowed
allowed, err := limiter.Allow(ctx, "user:123")
if !allowed {
    // Request is rate limited
}
```

### 2. HTTP Middleware

#### Rate Limit by IP

```go
s := g.Server()
limiter := glimiter.NewMemoryLimiter(100, time.Minute)

s.Group("/api", func(group *ghttp.RouterGroup) {
    group.Middleware(glimiter.MiddlewareByIP(limiter))
    
    group.GET("/users", handler)
})
```

#### Rate Limit by API Key

```go
limiter := glimiter.NewMemoryLimiter(1000, time.Hour)

s.Group("/api", func(group *ghttp.RouterGroup) {
    group.Middleware(glimiter.MiddlewareByAPIKey(limiter, "X-API-Key"))
    
    group.GET("/data", handler)
})
```

#### Custom Rate Limit Logic

```go
limiter := glimiter.NewMemoryLimiter(50, time.Minute)

s.Group("/api", func(group *ghttp.RouterGroup) {
    group.Middleware(glimiter.Middleware(glimiter.MiddlewareConfig{
        Limiter: limiter,
        KeyFunc: func(r *ghttp.Request) string {
            // Custom key: combine IP and User-Agent
            return r.GetClientIp() + ":" + r.UserAgent()
        },
        ErrorHandler: func(r *ghttp.Request) {
            r.Response.WriteStatus(429)
            r.Response.WriteJson(g.Map{
                "error": "Rate limit exceeded",
                "retry": time.Now().Add(time.Minute).Unix(),
            })
        },
    }))
    
    group.GET("/resource", handler)
})
```

### 3. Using Redis Limiter

```go
import (
    "github.com/LanceAdd/glimiter"
    "github.com/gogf/gf/v2/database/gredis"
)

// Create Redis connection
redis, err := gredis.New(&gredis.Config{
    Address: "127.0.0.1:6379",
})

// Create Redis limiter
limiter := glimiter.NewRedisLimiter(redis, 100, time.Minute)

// Usage is the same as in-memory limiter
allowed, err := limiter.Allow(ctx, "user:123")
```

## Core Interface

### Limiter Interface

```go
type Limiter interface {
    // Check if a single request is allowed
    Allow(ctx context.Context, key string) (bool, error)
    
    // Check if N requests are allowed
    AllowN(ctx context.Context, key string, n int) (bool, error)
    
    // Block until request is allowed
    Wait(ctx context.Context, key string) error
    
    // Get limit configuration
    GetLimit() int
    GetWindow() time.Duration
    
    // Get remaining quota
    GetRemaining(ctx context.Context, key string) (int, error)
    
    // Reset limit
    Reset(ctx context.Context, key string) error
}
```

## Usage Scenarios

### 1. Multi-layer Rate Limiting

Set multiple layers of restrictions for different time windows to prevent burst traffic and long-term abuse:

```go
// Layer 1: Burst protection (per second)
burstLimiter := glimiter.NewMemoryLimiter(10, time.Second)

// Layer 2: Normal limit (per minute)
normalLimiter := glimiter.NewMemoryLimiter(100, time.Minute)

// Layer 3: Long-term limit (per hour)
hourlyLimiter := glimiter.NewMemoryLimiter(1000, time.Hour)

s.Group("/api", func(group *ghttp.RouterGroup) {
    group.Middleware(
        glimiter.MiddlewareByIP(burstLimiter),
        glimiter.MiddlewareByIP(normalLimiter),
        glimiter.MiddlewareByIP(hourlyLimiter),
    )
    
    group.GET("/search", handler)
})
```

### 2. Route-level Rate Limiting

Different API routes use different rate limiting strategies:

```go
s := g.Server()

// Public API: loose restriction
s.Group("/public", func(group *ghttp.RouterGroup) {
    publicLimiter := glimiter.NewMemoryLimiter(100, time.Minute)
    group.Middleware(glimiter.MiddlewareByIP(publicLimiter))
    group.GET("/info", handler)
})

// Authenticated API: moderate restriction
s.Group("/auth", func(group *ghttp.RouterGroup) {
    authLimiter := glimiter.NewMemoryLimiter(5, time.Minute)
    group.Middleware(glimiter.MiddlewareByIP(authLimiter))
    group.POST("/login", handler)
})

// Sensitive operations: strict restriction
s.Group("/admin", func(group *ghttp.RouterGroup) {
    adminLimiter := glimiter.NewMemoryLimiter(10, time.Hour)
    group.Middleware(glimiter.MiddlewareByIP(adminLimiter))
    group.POST("/delete", handler)
})
```

### 3. Per-user Rate Limiting

```go
limiter := glimiter.NewMemoryLimiter(1000, time.Hour)

middleware := glimiter.MiddlewareByUser(limiter, func(r *ghttp.Request) string {
    // Get user ID from context
    user := r.GetCtxVar("user").String()
    return user
})

s.Group("/api", func(group *ghttp.RouterGroup) {
    group.Middleware(middleware)
    group.GET("/profile", handler)
})
```

### 4. Direct Use of Limiter

Using the limiter directly in business code without middleware:

```go
limiter := glimiter.NewMemoryLimiter(10, time.Minute)

func ProcessTask(ctx context.Context, taskID string) error {
    // Check if processing is allowed
    allowed, err := limiter.Allow(ctx, "task:"+taskID)
    if err != nil {
        return err
    }
    
    if !allowed {
        return errors.New("rate limit exceeded")
    }
    
    // Execute task
    return doTask(taskID)
}
```

### 5. Wait for Quota Availability

```go
limiter := glimiter.NewMemoryLimiter(5, time.Second)

func SendRequest(ctx context.Context) error {
    // Block until quota is available
    if err := limiter.Wait(ctx, "api-call"); err != nil {
        return err
    }
    
    // Send request
    return makeAPICall()
}
```

## Response Headers

The rate limiting middleware automatically sets the following HTTP response headers:

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum requests in the time window |
| `X-RateLimit-Remaining` | Remaining available requests |
| `X-RateLimit-Reset` | Rate limit reset time (Unix timestamp) |

## Best Practices

### 1. Reasonable Time Window Settings

- **Short time windows** (within 1 minute): Suitable for in-memory limiters, optimal performance
- **Long time windows** (over 1 hour): Recommended to use Redis limiters, support distributed environments

### 2. Use Multi-layer Rate Limiting

Combine rate limiting strategies for different time windows to prevent both burst traffic and long-term abuse:

- Layer 1: Second-level rate limiting to prevent burst attacks
- Layer 2: Minute-level rate limiting for regular usage restrictions
- Layer 3: Hour-level rate limiting for long-term quota management

### 3. Differentiate Between Scenarios

Set different rate limiting strategies according to the sensitivity and importance of APIs:

- **Public APIs**: Loose restrictions for good user experience
- **Authenticated APIs**: Moderate restrictions to prevent brute force attacks
- **Sensitive operations**: Strict restrictions to protect critical functions

### 4. Provide Friendly Error Messages

Customize error handlers to inform users when they can retry:

```go
ErrorHandler: func(r *ghttp.Request) {
    r.Response.WriteStatus(429)
    r.Response.WriteJson(g.Map{
        "error": "Rate limit exceeded",
        "message": "You have exceeded the rate limit. Please try again later.",
        "retry_after": limiter.GetWindow().Seconds(),
    })
}
```

### 5. Monitoring and Alerting

In production environments, it is recommended to monitor limiter usage:

```go
// Periodically check remaining quota
remaining, _ := limiter.GetRemaining(ctx, key)
if remaining < 10 {
    // Send alert
    log.Warn("Rate limit nearly exhausted", "key", key, "remaining", remaining)
}
```

## Performance Considerations

### In-Memory Limiter

- **Advantages**: Extremely high performance, no network overhead
- **Disadvantages**: Single-machine limitation, not suitable for distributed environments
- **Applicable**: Monolithic applications, short time windows

### Redis Limiter

- **Advantages**: Supports distributed environments, data persistence
- **Disadvantages**: Network latency overhead
- **Applicable**: Distributed applications, long time windows

## Concurrency Safety

### In-Memory Limiter

Uses CAS (Compare-And-Swap) atomic operations to ensure concurrency safety:

### Redis Limiter

Uses Lua scripts to ensure atomic operations:

## Frequently Asked Questions

### Q: How to use in distributed environments?

A: Use `RedisLimiter` instead of `MemoryLimiter`, ensuring all service instances share the same Redis.

### Q: How to implement dynamic adjustment of rate limiting configuration?

A: New `Limiter` instances can be created at runtime and middleware configuration updated.

### Q: How do time windows work?

A:
- **In-memory limiter**: Uses gcache's automatic expiration feature
- **Redis limiter**: Uses sliding window algorithm for precise time range control

### Q: How to avoid the limiter becoming a performance bottleneck?

A:
1. Use in-memory limiters for short time windows
2. Reasonably set rate limiting quotas
3. Use multi-layer rate limiting instead of single strict restrictions