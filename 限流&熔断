# 限流与熔断：保障分布式系统稳定的基石

限流和熔断是构建健壮分布式系统的关键模式，用于防止系统因过载或依赖服务故障而崩溃。它们共同构成了系统的"弹性"能力。

## 一、限流 (Rate Limiting)

限流通过控制请求的速率来保护系统，防止系统被突发流量冲垮。

### 常见限流算法

#### 1. 固定窗口计数器 (Fixed Window Counter)
- **原理**：将时间划分为固定窗口（如1秒），在每个窗口内计数请求数，超过阈值则拒绝请求
- **优点**：实现简单，内存效率高
- **缺点**：存在窗口边界处的流量突刺问题（可能在窗口开始和结束时各通过一次满负荷请求）

```go
// 固定窗口计数器实现示例
type FixedWindowLimiter struct {
    windowSize  time.Duration
    maxRequests int
    counters    map[string]*windowCounter
    mu          sync.Mutex
}

type windowCounter struct {
    count    int
    windowStart time.Time
}

func (l *FixedWindowLimiter) Allow(key string) bool {
    l.mu.Lock()
    defer l.mu.Unlock()
    
    now := time.Now()
    counter, exists := l.counters[key]
    
    if !exists || now.Sub(counter.windowStart) >= l.windowSize {
        // 新窗口或窗口已过期，重置计数器
        l.counters[key] = &windowCounter{
            count:       1,
            windowStart: now,
        }
        return true
    }
    
    if counter.count >= l.maxRequests {
        return false
    }
    
    counter.count++
    return true
}
```

#### 2. 滑动窗口日志 (Sliding Window Log)
- **原理**：记录每个请求的时间戳，统计最近时间窗口内的请求数
- **优点**：更精确地控制流量，解决了固定窗口的边界问题
- **缺点**：需要存储所有时间戳，内存开销大

```go
// 滑动窗口日志实现示例
type SlidingWindowLimiter struct {
    windowSize  time.Duration
    maxRequests int
    requests    map[string][]time.Time
    mu          sync.Mutex
}

func (l *SlidingWindowLimiter) Allow(key string) bool {
    l.mu.Lock()
    defer l.mu.Unlock()
    
    now := time.Now()
    windowStart := now.Add(-l.windowSize)
    
    // 清理过期的请求记录
    requests := l.requests[key]
    i := 0
    for i < len(requests) && requests[i].Before(windowStart) {
        i++
    }
    requests = requests[i:]
    
    // 检查当前窗口内的请求数
    if len(requests) >= l.maxRequests {
        return false
    }
    
    // 记录新请求
    requests = append(requests, now)
    l.requests[key] = requests
    return true
}
```

#### 3. 令牌桶算法 (Token Bucket)
- **原理**：以恒定速率向桶中添加令牌，请求需要获取令牌才能通过
- **优点**：允许突发流量，平滑限流
- **缺点**：实现相对复杂

```go
// 令牌桶算法实现示例
type TokenBucketLimiter struct {
    capacity    int        // 桶容量
    tokens      int        // 当前令牌数
    refillRate  float64   // 每秒添加的令牌数
    lastRefill  time.Time // 上次补充令牌的时间
    mu          sync.Mutex
}

func (l *TokenBucketLimiter) Allow() bool {
    l.mu.Lock()
    defer l.mu.Unlock()
    
    // 补充令牌
    now := time.Now()
    elapsed := now.Sub(l.lastRefill).Seconds()
    tokensToAdd := int(elapsed * l.refillRate)
    
    if tokensToAdd > 0 {
        l.tokens = min(l.capacity, l.tokens + tokensToAdd)
        l.lastRefill = now
    }
    
    // 检查是否有可用令牌
    if l.tokens > 0 {
        l.tokens--
        return true
    }
    
    return false
}
```

#### 4. 漏桶算法 (Leaky Bucket)
- **原理**：请求以任意速率进入桶中，但以固定速率流出处理
- **优点**：输出流量非常平滑，严格控制系统处理速率
- **缺点**：无法应对突发流量

## 二、单机限流 vs 分布式限流

### 单机限流
- **适用场景**：单实例服务，或不需要全局精确控制的场景
- **实现方式**：在单个服务实例内实现限流逻辑
- **优点**：实现简单，无网络开销，性能高
- **缺点**：无法在多个实例间共享状态，限流不精确

### 分布式限流
- **适用场景**：多实例服务，需要全局精确控制流量
- **实现方式**：使用外部存储（如Redis）协调多个实例的限流状态
- **优点**：全局精确控制，适用于微服务架构
- **缺点**：实现复杂，有网络开销，依赖外部服务

```go
// 基于Redis的分布式限流示例（使用固定窗口）
type RedisRateLimiter struct {
    client      *redis.Client
    windowSize  time.Duration
    maxRequests int
}

func (l *RedisRateLimiter) Allow(key string) (bool, error) {
    now := time.Now().Unix()
    windowStart := now - int64(l.windowSize.Seconds())
    
    // 使用Redis管道保证原子性
    pipe := l.client.Pipeline()
    pipe.ZRemRangeByScore(key, "0", strconv.FormatInt(windowStart, 10))
    countCmd := pipe.ZCard(key)
    pipe.ZAdd(key, redis.Z{Score: float64(now), Member: strconv.FormatInt(now, 10)})
    pipe.Expire(key, l.windowSize)
    
    _, err := pipe.Exec()
    if err != nil {
        return false, err
    }
    
    count, err := countCmd.Result()
    if err != nil {
        return false, err
    }
    
    return count < int64(l.maxRequests), nil
}
```

## 三、熔断器模式 (Circuit Breaker)

熔断器是一种防止"雪崩效应"的模式，当依赖服务失败率达到阈值时，自动"熔断"请求，避免资源耗尽。

### 熔断器三种状态
1. **关闭(Closed)**：正常处理请求，统计失败率
2. **打开(Open)**：拒绝所有请求，直接返回错误
3. **半开(Half-Open)**：允许少量请求通过，测试依赖服务是否恢复

### 熔断器实现

```go
// 熔断器实现
type CircuitBreaker struct {
    name          string
    maxFailures   int           // 最大失败次数
    resetTimeout  time.Duration // 熔断后等待时间
    state         State         // 当前状态
    failureCount  int           // 失败计数
    lastFailure   time.Time     // 最后失败时间
    mu            sync.Mutex
}

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

func (cb *CircuitBreaker) Execute(req func() error) error {
    cb.mu.Lock()
    
    // 检查熔断器状态
    switch cb.state {
    case StateOpen:
        if time.Since(cb.lastFailure) > cb.resetTimeout {
            cb.state = StateHalfOpen
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker is open")
        }
    case StateHalfOpen:
        // 在半开状态，允许一个请求测试
    case StateClosed:
        // 正常状态
    }
    
    cb.mu.Unlock()
    
    // 执行请求
    err := req()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.recordFailure()
        return err
    }
    
    cb.recordSuccess()
    return nil
}

func (cb *CircuitBreaker) recordFailure() {
    cb.failureCount++
    cb.lastFailure = time.Now()
    
    if cb.state == StateHalfOpen || cb.failureCount >= cb.maxFailures {
        cb.state = StateOpen
        cb.failureCount = 0
    }
}

func (cb *CircuitBreaker) recordSuccess() {
    if cb.state == StateHalfOpen {
        cb.state = StateClosed
    }
    cb.failureCount = 0
}
```

## 四、实践建议

1. **分层防御**：在API网关、服务网格和应用层等多个层面实施限流
2. **动态配置**：使限流阈值和熔断参数可动态调整，无需重启服务
3. **监控告警**：监控限流和熔断事件，设置适当告警
4. **优雅降级**：被限流或熔断时，提供有意义的错误响应或降级内容
5. **客户端限流**：在客户端实现限流和重试策略，减轻服务端压力

## 五、常用库和框架

- **Go标准库**：`golang.org/x/time/rate`（基于令牌桶）
- **Hystrix**：Netflix开源的熔断器库（已停止维护，但概念影响深远）
- **gobreaker**：基于Hystrix思想的Go熔断器库
- **resilience4j**：轻量级容错库（Java，但有类似Go的实现）
- **Envoy/Istio**：服务网格层面的限流和熔断

限流和熔断是构建弹性分布式系统的关键模式，正确实施可以显著提高系统的稳定性和可用性。在实际应用中，通常需要结合多种算法和策略，并根据具体业务需求进行调整。
