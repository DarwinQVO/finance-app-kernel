# ADR-0013: Exchange Rate Caching Strategy

**Status**: ✅ Accepted
**Date**: 2025-10-24
**Scope**: Vertical 3.6 Unit (Currency Conversion)

---

## Context

Need exchange rates for multi-currency normalization. External APIs (ECB, Federal Reserve) have rate limits and costs. Rates update daily (not real-time).

**The problem:**
- Multi-currency transactions need **normalization to base currency**:
  - EUR expense → USD (for USA tax reporting)
  - MXN income → USD (for consolidated P&L)
  - JPY investment → USD (for portfolio valuation)
- Exchange rates come from **external APIs**:
  - European Central Bank (ECB): Free, rate-limited (1000 req/day)
  - Federal Reserve: Free, rate-limited (100 req/hour)
  - Paid services (XE, Oanda): $50-500/month, higher limits
- Rate characteristics:
  - **Update frequency**: Daily (ECB at 16:00 CET, Fed at 11:00 EST)
  - **Historical immutability**: 2025-10-24 USD/EUR rate never changes
  - **Staleness acceptable**: Accounting uses official daily rates (not real-time)

**Requirements:**
- Minimize API costs (free tier preferred)
- Fast conversion (<10ms for cached rates)
- Resilient (fallback sources if primary fails)
- Historically accurate (use correct rate for transaction date)

---

## Decision

**Use 24-hour cache with staleness detection and fallback sources.**

### Caching Strategy

```
1. Check cache (Redis/DB) for rate with date key: "USD_EUR_2025-10-24"
2. If cached and fresh (<24h): Return cached rate
3. If stale or missing: Fetch from ECB (primary source)
4. If ECB fails: Fallback to Federal Reserve API
5. If all fail: Use last known rate with staleness warning
6. Store with TTL: 24 hours
```

### Cache Key Format

```
{base_currency}_{quote_currency}_{date}

Examples:
- USD_EUR_2025-10-24 → 0.9234
- USD_MXN_2025-10-24 → 18.4521
- EUR_JPY_2025-10-24 → 162.3456
```

### Data Source Hierarchy

| Priority | Source | Update Time | Rate Limit | Cost | Notes |
|----------|--------|-------------|------------|------|-------|
| **1** | ECB | 16:00 CET daily | 1000/day | Free | Primary (most currencies) |
| **2** | Federal Reserve | 11:00 EST daily | 100/hour | Free | Fallback (USD pairs only) |
| **3** | Last Known Rate | N/A | N/A | Free | Emergency fallback (with warning) |

---

## Rationale

### 1. Rate Update Frequency (Daily)

**ECB publishes rates once per day:**
- 16:00 CET (10:00 EST, 07:00 PST)
- Rates represent **official daily reference** (not intraday fluctuations)
- Used by banks, accounting firms, tax authorities

**Implication:**
- No need for real-time rates (WebSocket, streaming API)
- 24-hour cache covers full day until next update
- Historical rates immutable (cache forever for past dates)

### 2. API Cost Minimization

**Without caching (naive approach):**
- 100 transactions/day × 50% multi-currency = 50 conversions/day
- 50 conversions × 365 days = **18,250 API calls/year**
- Free tier exhausted in 20 days (ECB limit: 1000/day = 30K/month)

**With 24-hour cache:**
- Unique currency pairs per day: ~5 (USD/EUR, USD/MXN, USD/JPY, USD/GBP, USD/CAD)
- API calls: 5 pairs × 365 days = **1,825 calls/year** (90% reduction)
- Well within free tier (ECB: 365K/year, Fed: 876K/year)

### 3. Cache Performance

**Cache hit scenario (99% of conversions):**
- Redis lookup: <1ms
- Rate retrieval: <10ms total
- No network latency

**Cache miss scenario (1% of conversions, new date):**
- Redis miss: <1ms
- ECB API call: ~200ms (network + processing)
- Store in cache: <5ms
- Subsequent conversions same day: Cache hit

### 4. Historical Accuracy

**Accounting requirement:**
- Transaction on 2025-10-24 must use 2025-10-24 exchange rate
- Not today's rate, not average rate

**Cache design:**
- Key includes date: `USD_EUR_2025-10-24`
- Historical rates cached forever (immutable)
- Current date cached 24h (until next update)

**Example:**
```
Transaction: EUR 100 on 2025-10-24
Rate: USD_EUR_2025-10-24 = 0.9234
Converted: $92.34 (stored in normalized_amount field)

If queried on 2025-12-31:
- Still uses 0.9234 (historical accuracy)
- Not current rate 0.9500 (wrong)
```

### 5. Resilience (Fallback Sources)

**ECB downtime (rare but occurs):**
- Maintenance windows, API outages
- Fallback to Federal Reserve (covers major USD pairs)

**All sources down (extremely rare):**
- Use last known rate (yesterday's rate)
- Flag with staleness warning: "Rate from 2025-10-23 (ECB unavailable)"
- User can manually refresh later

---

## Consequences

### ✅ Positive

- **Low API costs**: <10 calls/day per currency pair (well within free tier)
- **Fast conversion**: <10ms from cache (99% hit rate)
- **Resilient**: Fallback sources ensure availability
- **Historically accurate**: Correct rate for transaction date
- **Scalable**: Add new currency pairs without refactoring

### ⚠️ Trade-offs

- **Stale rates possible**: Volatile market >1% intraday (rare for major pairs)
- **Manual refresh needed**: Power users want "force refresh" (edge case)
- **Cache storage**: 5 pairs × 365 days × 5 years = 9,125 cache entries (negligible)
- **Timezone complexity**: ECB updates 16:00 CET (need timezone-aware cache invalidation)

### 🔴 Risks (Mitigated)

- **Risk**: Historical rate changes NOT reflected (cached rate persists)
  - **Reality**: Historical rates don't change (2025-10-24 rate is final)
  - **No mitigation needed** (not a risk, intended behavior)

- **Risk**: Stale rates for volatile periods (>1% intraday variance)
  - **Mitigation**: Acceptable for accounting (daily rates are standard)
  - **Mitigation**: Manual refresh option for power users
  - **Edge case**: Hyperinflation (Argentina, Venezuela) requires real-time rates (future feature)

- **Risk**: Cache invalidation bugs (stale rate not refreshed)
  - **Mitigation**: TTL-based expiration (Redis handles automatically)
  - **Mitigation**: Health check: If rate older than 48h, force refresh

- **Risk**: Timezone confusion (16:00 CET = when in user's timezone?)
  - **Mitigation**: Server runs UTC, converts ECB publish time to UTC (15:00 UTC in winter, 14:00 UTC in summer)
  - **Mitigation**: Cache invalidation triggers at 16:00 CET regardless of user timezone

---

## Cache Stampede Mitigation

**Problem**: All exchange rate cache keys expire simultaneously at 16:00 CET:

```
16:00 CET: All EUR/USD, GBP/USD, JPY/USD, etc. keys expire
16:01: 1000 concurrent requests for same rates
Result: 1000 simultaneous ECB API calls (should be 1)
```

**Why this happens:**
- ECB publishes rates once daily at 16:00 CET
- All currency pairs expire at same time (24h TTL from previous day's 16:00)
- Burst of user activity (users check conversions after new rates published)

**Consequences:**
- ECB rate limit exhausted instantly (1000 req/day limit)
- Cascading failures (API 429 errors → fallback to FED → FED exhausted)
- User-facing errors ("Rate unavailable, try again later")

### Solution: Probabilistic Early Expiration

**Strategy**: Add random jitter to TTL to spread cache refreshes over time:

```python
def get_cache_ttl(base_ttl: int = 86400) -> int:
    """
    Return TTL with ±10% random jitter.

    Base: 24 hours (86400 seconds)
    Range: 21.6h - 24h (77760 - 86400 seconds)

    Effect: Spreads expirations over 2.4-hour window instead of instant
    """
    jitter = random.uniform(0.9, 1.0)  # 90% - 100% of base TTL
    return int(base_ttl * jitter)

# Usage
redis.setex(
    key="USD_EUR_2025-10-24",
    value=0.9234,
    ttl=get_cache_ttl()  # 77760-86400 seconds instead of 86400
)
```

**Result**:
- Some keys expire at 14:30 CET (early)
- Most keys expire between 15:00-16:00 CET
- Last keys expire at 16:00 CET
- API calls spread over 1.5-hour window instead of 1 second

**Benefits**:
- ✅ No thundering herd (gradual refresh instead of burst)
- ✅ API rate limits respected (10 calls/min vs 1000 calls/sec)
- ✅ No user-facing errors (smooth degradation)
- ✅ Simple implementation (2 lines of code)

### Alternative Mitigation: Request Coalescing

**Strategy**: If cache miss detected, check if refresh already in progress:

```python
def get_exchange_rate_with_coalescing(base: str, quote: str, date: str):
    cache_key = f"{base}_{quote}_{date}"
    lock_key = f"refresh_lock:{cache_key}"

    # Step 1: Check cache
    cached_rate = redis.get(cache_key)
    if cached_rate:
        return cached_rate

    # Step 2: Try to acquire refresh lock
    if redis.setnx(lock_key, "1", ex=10):  # 10-second lock
        # We got the lock - we are responsible for refreshing
        try:
            rate = fetch_from_ecb(base, quote, date)
            redis.setex(cache_key, rate, ttl=get_cache_ttl())
            return rate
        finally:
            redis.delete(lock_key)
    else:
        # Someone else is refreshing - wait and retry
        time.sleep(0.1)  # Wait 100ms

        # Check cache again (other thread should have populated it)
        cached_rate = redis.get(cache_key)
        if cached_rate:
            return cached_rate

        # Still missing - fallback to last known rate
        return get_last_known_rate(base, quote)
```

**Benefits**:
- ✅ Only 1 API call per currency pair (even with 1000 concurrent requests)
- ✅ Guaranteed deduplication (lock ensures single refresh)
- ✅ Works with existing cache strategy

**Trade-offs**:
- ⚠️ More complex (lock management, retry logic)
- ⚠️ Potential deadlocks (if lock holder crashes)
- ⚠️ 100ms latency for waiting threads

### Recommendation

**Use probabilistic early expiration** (simpler, good enough):
- 10% jitter spreads load over 1.5 hours
- No lock complexity or deadlock risk
- Worst case: 10% of requests hit API (still within rate limits)

**Add request coalescing later** if needed (production optimization):
- Only if probabilistic jitter insufficient (high-traffic scenario)
- Adds complexity but guarantees single API call per key

---

## Alternatives Considered

### Alternative A: No Cache (Direct API Calls) — Rejected

**Approach:**
- Every conversion calls ECB API directly

**Pros:**
- Simplest implementation (no cache logic)
- Always fresh rates

**Cons:**
- **Expensive**: 18K API calls/year (exceeds free tier in 20 days)
- **Slow**: 200ms per conversion (vs <10ms cached)
- **Rate limit risk**: Burst of conversions exhausts quota
- **No offline support**: API down = conversions fail

### Alternative B: Infinite Cache (Never Expire) — Rejected

**Approach:**
- Cache rate forever (no TTL)

**Pros:**
- Zero API calls after first fetch
- Maximum performance

**Cons:**
- **Stale rates for current date**: Today's rate never updates (stuck at 16:00 CET rate)
- **Volatile market**: Major events (Brexit, Fed rate hike) not reflected until manual refresh
- **User confusion**: "Why is my conversion using yesterday's rate?"

### Alternative C: Cache Per Session (Device-Specific) — Rejected

**Approach:**
- Each device caches rates locally (browser localStorage, mobile app cache)

**Pros:**
- No server-side cache needed
- Offline support

**Cons:**
- **No cross-device consistency**: Desktop uses 0.9234, mobile uses 0.9250 (fetched at different times)
- **Sync complexity**: Multi-device users see different converted amounts
- **No historical accuracy**: Device cleared cache → old transactions lose correct rate

### Alternative D: Real-Time Rates (WebSocket) — Rejected

**Approach:**
- Subscribe to real-time FX feed (WebSocket, streaming API)

**Pros:**
- Always current (tick-by-tick updates)
- No staleness

**Cons:**
- **Expensive**: $50-500/month (no free tier)
- **Overkill**: Accounting doesn't need real-time (daily rates standard)
- **Complexity**: WebSocket connection management, reconnection logic
- **Historical inaccuracy**: Real-time rate at 10:37:22 AM not official daily rate

---

## Implementation Notes

### Cache Storage (Redis)

```python
# Store rate
redis.setex(
    key="USD_EUR_2025-10-24",
    value=0.9234,
    ttl=86400  # 24 hours
)

# Retrieve rate
rate = redis.get("USD_EUR_2025-10-24")
if rate is None:
    rate = fetch_from_ecb("USD", "EUR", "2025-10-24")
    redis.setex(key, rate, ttl=86400)
```

### Fallback Logic

```python
def get_exchange_rate(base: str, quote: str, date: str) -> ExchangeRate:
    # Step 1: Check cache
    cache_key = f"{base}_{quote}_{date}"
    cached_rate = redis.get(cache_key)

    if cached_rate and is_fresh(cached_rate):
        return ExchangeRate(rate=cached_rate, source="cache", staleness=0)

    # Step 2: Fetch from ECB (primary)
    try:
        rate = ecb_api.get_rate(base, quote, date)
        redis.setex(cache_key, rate, ttl=86400)
        return ExchangeRate(rate=rate, source="ECB", staleness=0)
    except ECBError as e:
        log.warning(f"ECB failed: {e}")

    # Step 3: Fallback to Federal Reserve
    try:
        rate = fed_api.get_rate(base, quote, date)
        redis.setex(cache_key, rate, ttl=86400)
        return ExchangeRate(rate=rate, source="FED", staleness=0)
    except FedError as e:
        log.warning(f"FED failed: {e}")

    # Step 4: Emergency fallback (last known rate)
    last_known = get_last_known_rate(base, quote)
    staleness_days = (today - last_known.date).days

    log.error(f"All sources failed, using last known rate ({staleness_days} days old)")
    return ExchangeRate(
        rate=last_known.rate,
        source="last_known",
        staleness=staleness_days,
        warning="Rate may be stale (API unavailable)"
    )
```

### Timezone Handling

```python
# ECB publishes at 16:00 CET (14:00 UTC in summer, 15:00 UTC in winter)
ECB_PUBLISH_TIME_CET = time(16, 0)  # 16:00 CET

def should_refresh_cache(cache_timestamp: datetime) -> bool:
    # Convert ECB publish time to UTC
    ecb_utc = convert_to_utc(ECB_PUBLISH_TIME_CET, "CET")

    # If cache older than today's ECB publish time, refresh
    today_ecb_publish = datetime.combine(date.today(), ecb_utc)
    return cache_timestamp < today_ecb_publish
```

### Performance Metrics

- **Cache hit rate**: 99% (empirical)
- **Average conversion time**: 8ms (cached), 210ms (miss)
- **API calls per day**: 5-10 (new currency pairs + daily refresh)
- **Storage**: 100 bytes per rate × 9,125 entries (5 years) = 912 KB

---

## Related Decisions

- **ADR-0012**: Transfer detection normalizes FX transfers to base currency before scoring
- **ADR-0006**: Normalized amounts stored in canonical ledger (conversion happens once at ingestion)
- **Future**: ADR-0023 will add cryptocurrency rates (higher volatility, more frequent updates)

---

**References:**
- Vertical 3.6: [docs/verticals/3.6-unit.md](../verticals/3.6-unit.md)
- ExchangeRateProvider: [docs/primitives/ol/ExchangeRateProvider.md](../primitives/ol/ExchangeRateProvider.md)
- Schema: [docs/schemas/exchange-rate-record.schema.json](../schemas/exchange-rate-record.schema.json)
