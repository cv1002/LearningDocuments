# 限流
- [限流](#限流)
  - [Redis + Lua脚本 实现网关层限流](#redis--lua脚本-实现网关层限流)
  - [接入 Sentinel 实现网关层和应用层限流](#接入-sentinel-实现网关层和应用层限流)

## Redis + Lua脚本 实现网关层限流
SpringCloud Gateway 官方提供了 `RequestRateLimiter` 过滤器工厂。使用 Redis + Lua 脚本实现限流。使用的算法是令牌桶算法。

以下示例定义了每个用户的请求数率为 10。允许最多 20 的突发请求，但在下一秒内，只有 10 个请求可用。
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: requestratelimiter_route
        uri: https://example.org
        filters:
        - name: RequestRateLimiter
          args:
            bucket4j-rate-limiter.capacity: 20
            bucket4j-rate-limiter.refillTokens: 10
            bucket4j-rate-limiter.refillPeriod: 1s
            bucket4j-rate-limiter.requestedTokens: 1
```

## 接入 Sentinel 实现网关层和应用层限流


