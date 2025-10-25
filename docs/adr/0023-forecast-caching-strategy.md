# ADR-0023: Forecast Caching Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 4.2 (affects forecast performance)

---

## Context

Projections are computationally expensive (500ms for 90-day Holt-Winters forecast on 3 years data). Users may reload dashboard frequently (5-10× per session). Need caching strategy to improve performance without serving stale data.

**The challenge:**

```
Dashboard Load → Check Cache → [Hit: 50ms | Miss: 500ms compute] → Display
```

**Requirements:**
- **Performance**: <100ms p95 for dashboard rendering
- **Freshness**: Projections reflect recent transactions (within 24h acceptable)
- **Invalidation**: New transaction triggers cache invalidation
- **Scalability**: Handle 10k+ concurrent users
- **Shared state**: Cache shared across user's devices (mobile, web, desktop)

**Trade-off space:**
- **Freshness** vs **Performance**: Real-time calculation (no cache) = 500ms, perpetual cache = instant but stale
- **Complexity** vs **Reliability**: Client-side cache is simple but not shared across devices
- **Cost** vs **Performance**: Redis cache adds infrastructure cost but 10× performance gain

---

## Decision

**We use Hybrid caching: Redis with 24h TTL + invalidation on material changes.**

- **Cache layer**: Redis (centralized, shared across all servers)
- **TTL**: 24 hours (86400 seconds)
- **Invalidation triggers**:
  - New transaction normalized
  - Goal updated
  - Historical data corrected
- **Stampede prevention**: Lock-based pattern (SETNX)
- **Warming**: Pre-calculate common forecasts for active users (last 7 days)

---

## Rationale

### 1. Performance with Freshness Balance
- **Cache hit**: <50ms (10× faster than 500ms calculation)
- **TTL ensures freshness**: Data <24h stale (acceptable for projections, not real-time data)
- **Invalidation on material changes**: New transaction immediately invalidates cache → next request recalculates

### 2. Shared Across Devices
- Redis cache shared across:
  - Web app (React)
  - Mobile app (iOS/Android)
  - Desktop app (Electron)
- User sees consistent projections on all devices (no client-side cache drift)

### 3. Scalability
- Redis handles 100k+ requests/sec on standard instance
- Horizontal scaling: Redis Cluster shards by cache key
- Memory efficient: 1KB per forecast × 10k users × 5 forecasts = 50MB

### 4. Invalidation Granularity
- Invalidate only affected forecasts:
  - New expense → invalidate `forecast:user123:expenses:*`
  - Goal updated → invalidate `forecast:user123:goal_progress:*`
- Keep unaffected forecasts cached (e.g., income forecast when expense added)

### 5. Stampede Prevention
- Lock-based pattern prevents cache stampede:
  - 1000 users request same forecast simultaneously
  - First request acquires lock, calculates, writes to cache
  - Other 999 requests wait for lock, then read from cache
- Avoids 1000× simultaneous calculations (DoS risk)

---

## Consequences

### ✅ Positive

- **Fast dashboard loads**: <50ms for cached projections (vs 500ms uncached)
- **Fresh data**: Invalidated on transactions, <24h stale otherwise
- **Reduced compute costs**: 90% cache hit rate = 10× fewer calculations
- **Shared state**: Consistent across all user devices
- **Scalable**: Redis horizontal scaling supports millions of users

### ⚠️ Trade-offs

- **Redis dependency**: Single point of failure without HA (mitigated with Redis Sentinel)
- **Memory cost**: $50/month for Redis instance (vs $0 for no cache)
- **Eventual consistency**: 1-2 second delay between transaction → invalidation → recalculation

### 🔴 Risks (Mitigated)

- **Risk**: Cache stampede (all users hit DB if cache expires simultaneously)
  - **Mitigation**: Lock-based pattern (`SETNX forecast_lock:{key} 1 EX 10`)
- **Risk**: Redis failure causes 500ms latency for all requests
  - **Mitigation**: Redis Sentinel (automatic failover), circuit breaker falls back to direct calculation
- **Risk**: Invalidation logic bug causes perpetually stale cache
  - **Mitigation**: 24h TTL ensures cache refreshes daily, monitoring alerts if cache hit rate <80%

---

## Alternatives Considered

### Alternative A: No Caching (Real-time Calculation Every Time) (Rejected)

**Pros:**
- **Always fresh**: No stale data risk
- **Simple**: No cache infrastructure

**Cons:**
- **Slow**: 500ms per page load (poor UX, especially on mobile)
- **High compute cost**: 10× more calculations
- **Scalability bottleneck**: 10k concurrent users = 10k simultaneous calculations (CPU exhaustion)

### Alternative B: Pre-calculate Batch Job (Nightly) (Rejected)

**Pros:**
- **Fast reads**: Pre-calculated, instant response
- **Predictable load**: Batch job runs at 2am (off-peak)

**Cons:**
- **Stale data**: Up to 24h old (unacceptable for users adding transactions during day)
- **No invalidation**: New transaction doesn't update forecast until next batch run
- **Wasted compute**: Calculates forecasts for inactive users (last login 6 months ago)

### Alternative C: Client-side Caching (localStorage) (Rejected)

**Pros:**
- **Fast**: Instant cache hit
- **No server infrastructure**: Zero cost

**Cons:**
- **Not shared across devices**: User's mobile shows different projection than web
- **Can't invalidate centrally**: User adds transaction on mobile, web still shows old cache
- **Storage limits**: localStorage limited to 5-10MB (can't cache large datasets)
- **Security risk**: Sensitive financial data in client storage (XSS vulnerability)

### Alternative D: Perpetual Cache (Never Expires) (Rejected)

**Pros:**
- **Instant**: Always cache hit
- **Zero compute**: Calculate once, serve forever

**Cons:**
- **Stale forever**: User adds transaction, projection never updates (broken UX)
- **Invalidation required**: Manual invalidation logic complex (easy to miss edge cases)
- **Memory leak**: Cache grows indefinitely (no eviction)

---

## Implementation Notes

### Cache Key Structure

```
forecast:{user_id}:{metric}:{horizon}:{algorithm}

Examples:
- forecast:user123:balance:90d:holt_winters
- forecast:user123:expenses:30d:linear_regression
- forecast:user456:income:180d:holt_winters
```

### Redis Operations

```python
import redis
from typing import Optional

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def get_forecast_from_cache(user_id: str, metric: str, horizon: str, algorithm: str) -> Optional[dict]:
    """
    Returns cached forecast or None if cache miss.
    """
    key = f"forecast:{user_id}:{metric}:{horizon}:{algorithm}"
    cached = redis_client.get(key)
    if cached:
        return json.loads(cached)
    return None

def set_forecast_in_cache(user_id: str, metric: str, horizon: str, algorithm: str, forecast: dict):
    """
    Writes forecast to cache with 24h TTL.
    """
    key = f"forecast:{user_id}:{metric}:{horizon}:{algorithm}"
    redis_client.setex(key, 86400, json.dumps(forecast))  # 24h TTL

def invalidate_user_forecasts(user_id: str):
    """
    Invalidates all forecasts for a user (called on new transaction).
    """
    pattern = f"forecast:{user_id}:*"
    keys = redis_client.keys(pattern)
    if keys:
        redis_client.delete(*keys)
```

### Stampede Prevention with Lock

```python
import time

def get_or_compute_forecast(user_id: str, metric: str, horizon: str, algorithm: str) -> dict:
    """
    Returns forecast from cache or computes with stampede prevention.
    """
    # Check cache
    cached = get_forecast_from_cache(user_id, metric, horizon, algorithm)
    if cached:
        return cached

    # Cache miss - acquire lock
    lock_key = f"forecast_lock:{user_id}:{metric}:{horizon}:{algorithm}"
    lock_acquired = redis_client.setnx(lock_key, 1)
    redis_client.expire(lock_key, 10)  # Lock expires in 10s

    if lock_acquired:
        # This process won the lock - compute forecast
        try:
            forecast = compute_forecast(user_id, metric, horizon, algorithm)  # 500ms
            set_forecast_in_cache(user_id, metric, horizon, algorithm, forecast)
            return forecast
        finally:
            redis_client.delete(lock_key)
    else:
        # Another process is computing - wait for it
        for _ in range(20):  # Wait up to 2 seconds
            time.sleep(0.1)
            cached = get_forecast_from_cache(user_id, metric, horizon, algorithm)
            if cached:
                return cached

        # Timeout - compute anyway (lock holder may have crashed)
        forecast = compute_forecast(user_id, metric, horizon, algorithm)
        set_forecast_in_cache(user_id, metric, horizon, algorithm, forecast)
        return forecast
```

### Invalidation Triggers

```python
# In transaction normalization flow
def normalize_transaction(upload_id: str, user_id: str):
    # ... normalization logic ...

    # Invalidate forecasts
    invalidate_user_forecasts(user_id)

    # ... rest of flow ...

# In goal update flow
def update_goal(goal_id: str, user_id: str, updates: dict):
    # ... update logic ...

    # Invalidate goal-related forecasts
    invalidate_user_forecasts(user_id)

    # ... rest of flow ...
```

### Batch Warming (Optional Optimization)

```python
# Cron job runs every 6 hours
def warm_cache_for_active_users():
    """
    Pre-calculates forecasts for users active in last 7 days.
    """
    active_users = db.query("SELECT user_id FROM users WHERE last_login > NOW() - INTERVAL '7 days'")

    for user in active_users:
        for metric in ['balance', 'expenses', 'income']:
            for horizon in ['30d', '90d']:
                # Check if already cached
                cached = get_forecast_from_cache(user.user_id, metric, horizon, 'holt_winters')
                if not cached:
                    # Compute and cache
                    forecast = compute_forecast(user.user_id, metric, horizon, 'holt_winters')
                    set_forecast_in_cache(user.user_id, metric, horizon, 'holt_winters', forecast)
```

### Monitoring

```python
# Prometheus metrics
cache_hit_rate = Gauge('forecast_cache_hit_rate', 'Percentage of cache hits')
cache_miss_latency = Histogram('forecast_cache_miss_latency_seconds', 'Time to compute on cache miss')

def get_forecast_with_metrics(user_id: str, metric: str, horizon: str, algorithm: str) -> dict:
    cached = get_forecast_from_cache(user_id, metric, horizon, algorithm)

    if cached:
        cache_hit_rate.inc()
        return cached
    else:
        start = time.time()
        forecast = compute_forecast(user_id, metric, horizon, algorithm)
        cache_miss_latency.observe(time.time() - start)
        set_forecast_in_cache(user_id, metric, horizon, algorithm, forecast)
        return forecast
```

### Redis High Availability (Production)

```yaml
# docker-compose.yml
services:
  redis-master:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  redis-replica:
    image: redis:7-alpine
    command: redis-server --replicaof redis-master 6379
    depends_on:
      - redis-master

  redis-sentinel:
    image: redis:7-alpine
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica
```

---

## Related Decisions

- **ADR-0021**: Projection algorithm (Holt-Winters is what we're caching)
- **ADR-0022**: Goal tracking (goals trigger cache invalidation)
- **Future**: May add edge caching (Cloudflare) for public dashboards (shared forecasts)

---

**References:**
- Vertical 4.2: [docs/verticals/4.2-forecast.md](../verticals/4.2-forecast.md)
- Primitive: [docs/primitives/ol/ForecastCache.md](../primitives/ol/ForecastCache.md)
- Redis Best Practices: https://redis.io/docs/manual/patterns/
- Cache Stampede Prevention: https://redis.io/docs/manual/patterns/cache-aside/
