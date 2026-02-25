# Market Data Service — Design Document


## 1. Problem Statement

The Security API currently returns dummy market data. We need to fetch real-time market data from LSEG via WebSocket. Each trader call to the Security API must return fresh market data **without** opening a new WebSocket connection per request.

---

## 2. Architecture Decision

**Pattern: Shared Singleton WebSocket + In-Memory Cache**

A single, application-scoped WebSocket connection to LSEG is maintained by a `MarketDataService` bean. Market data received is stored in an in-memory `ConcurrentHashMap` keyed by `securityId`. All trader requests read from this cache — no per-request connections are opened.

### Why not per-request connections?
- LSEG rate-limits/connection-limits WebSocket clients
- Connection handshake + auth overhead per request (~100-300ms) is unacceptable
- Thousands of concurrent traders would exhaust server resources

### Why not Redis?
- Single-instance deployment (per requirements)
- Market data is ephemeral; process-local cache is sufficient
- No cross-service consumers for market data

---

## 3. Component Design

```
┌──────────────────────────────────────────────────────────┐
│                     Spring Boot App                       │
│                                                          │
│  ┌──────────────┐    ┌─────────────────┐                │
│  │  Security     │───▶│  Security       │                │
│  │  Controller   │    │  Service        │                │
│  └──────────────┘    └────────┬────────┘                │
│         POST /security        │                          │
│                               │ getMarketData(securityId)│
│                               ▼                          │
│                      ┌─────────────────┐                │
│                      │  MarketData     │                │
│                      │  Service        │                │
│                      │  (Singleton)    │                │
│                      └────────┬────────┘                │
│                               │                          │
│                  ┌────────────┼────────────┐            │
│                  ▼                         ▼            │
│         ┌──────────────┐        ┌──────────────────┐   │
│         │  In-Memory   │        │  WebSocket       │   │
│         │  Cache        │◀───────│  Connection Mgr  │   │
│         │  (Concurrent  │        │  (Singleton)     │   │
│         │   HashMap)   │        └────────┬─────────┘   │
│         └──────────────┘                 │              │
│                                          │              │
└──────────────────────────────────────────┼──────────────┘
                                           │ WSS
                                           ▼
                                   ┌──────────────┐
                                   │  LSEG        │
                                   │  WebSocket   │
                                   └──────────────┘
```

---

## 4. Component Responsibilities

### 4.1 SecurityController
- Unchanged. POST endpoint receives security query request.
- Delegates to `SecurityService`.

### 4.2 SecurityService
- Fetches security details from DB (existing flow).
- Calls `MarketDataService.getMarketData(securityId, symbol, exchangeCode)` to enrich the response with live market data.
- Merges DB data + market data into the response DTO.

### 4.3 MarketDataService (New — Singleton Bean)

**Responsibilities:**
- Owns the in-memory market data cache.
- Owns the **in-flight request map** for request coalescing (see Section 5.3).
- Exposes `getMarketData(securityId)` → returns `MarketData` (price, high, low, open, close, last traded time).
- On cache hit: returns cached data directly (with optional staleness check).
- On cache miss: checks in-flight map before subscribing to LSEG (see Section 5.3).

**Cache Strategy:**

| Config | Value | Rationale |
|--------|-------|-----------|
| Structure | `ConcurrentHashMap<String, MarketDataEntry>` | Thread-safe, lock-free reads |
| In-flight Map | `ConcurrentHashMap<String, CompletableFuture<MarketData>>` | Request coalescing; one LSEG call per symbol |
| Key | `securityId` | Unique per instrument |
| TTL | Configurable (e.g. 30s) | Prevents serving stale data post-market-close |
| Max Size | Configurable (e.g. 10,000) | Bounded memory; LRU eviction if using Caffeine |

> **Alternative:** Replace `ConcurrentHashMap` with [Caffeine](https://github.com/ben-manes/caffeine) for built-in TTL expiry and size-based eviction without custom housekeeping.

### 4.4 WebSocketConnectionManager (New — Singleton Bean)

**Responsibilities:**
- Manages the single persistent WebSocket connection to LSEG.
- Handles connection lifecycle: connect, authenticate, heartbeat, reconnect.
- Routes incoming LSEG messages to `MarketDataService` to update cache.
- Exposes `send(message)` for outbound subscription requests.

**Implementation:** Spring's `WebSocketClient` (standard) or `WebSocketStompClient` depending on LSEG protocol.

---

## 5. Request Flow

### 5.1 Happy Path (Cache Hit)

```
Trader → Controller → SecurityService → DB (security details)
                                       → MarketDataService.getMarketData()
                                           → Cache HIT → return cached MarketData
                     ← Response (security + marketData)
```

**Latency added:** ~0ms (in-memory lookup)

### 5.2 Cache Miss — First Request for a Symbol (Request Coalescing)

**Problem:** N traders request the same symbol simultaneously on a cold cache → without protection, N identical subscribe messages are sent to LSEG.

**Solution:** An **in-flight request map** (`ConcurrentHashMap<String, CompletableFuture<MarketData>>`) ensures only ONE subscribe is sent per symbol, regardless of concurrent callers.

```
Trader 1 ─┐
Trader 2 ─┤  (all request AAPL concurrently)
  ...      ├→ Controller → SecurityService → MarketDataService.getMarketData("AAPL")
Trader N ─┘
                                                │
                                                ▼
                                        ┌─── Cache MISS ───┐
                                        │                   │
                                        ▼                   │
                              inFlightMap.computeIfAbsent() │
                                   │            │           │
                              (key absent)  (key exists)    │
                                   │            │           │
                                   ▼            ▼           │
                            Create new     Return existing  │
                         CompletableFuture  CompletableFuture
                                   │            │
                                   ▼            │
                          Send ONE subscribe    │
                          to LSEG via WS        │
                                   │            │
                                   ▼            │
                          LSEG responds         │
                                   │            │
                                   ▼            │
                          future.complete()     │
                          cache.put()           │
                          inFlightMap.remove()  │
                                   │            │
                                   ▼            ▼
                          All N traders receive the same MarketData
```

**Key behaviour:**

| Sequence | What happens |
|----------|-------------|
| Trader 1 requests AAPL | Cache miss → `computeIfAbsent` creates `CompletableFuture` → sends subscribe to LSEG |
| Traders 2–N request AAPL (before LSEG responds) | Cache miss → `computeIfAbsent` finds existing future → all wait on same future (no duplicate subscribe) |
| LSEG responds | Future completes → all N callers unblock → cache populated → future removed from in-flight map |
| Trader N+1 requests AAPL (after response) | Cache hit → served directly from cache |

**Latency added:** Depends on LSEG response time (typically <500ms). Only the first caller pays the cost; concurrent callers piggyback.

### 5.3 Request Coalescing (Thundering Herd Prevention)

**Problem:** 50 traders request AAPL simultaneously with an empty cache → without protection, 50 identical subscribe messages are sent to LSEG → wasteful, potentially rate-limited.

**Solution:** An **in-flight request map** (`ConcurrentHashMap<String, CompletableFuture<MarketData>>`) sits alongside the data cache.

**Flow:**

```
Trader 1 → getMarketData("AAPL")
              → Cache MISS
              → In-flight map MISS
              → Create CompletableFuture, put in in-flight map
              → Send ONE subscribe to LSEG
              → Wait on future

Trader 2..50 → getMarketData("AAPL")  (concurrent)
              → Cache MISS
              → In-flight map HIT (future exists)
              → Wait on SAME future (no subscribe sent)

LSEG responds with AAPL data
              → Cache updated
              → CompletableFuture completed
              → All 50 callers receive result
              → Future removed from in-flight map

Trader 51 → getMarketData("AAPL")
              → Cache HIT → return immediately
```

**Key Mechanics:**

| Aspect | Detail |
|--------|--------|
| Map operation | `computeIfAbsent(securityId, k -> triggerSubscribe(k))` — atomic, guarantees single subscribe |
| Cleanup | Remove entry from in-flight map in `future.whenComplete()` (success or failure) |
| Timeout | Each waiting caller has its own `future.get(timeout)` — if LSEG is slow, callers timeout independently |
| Failure | If subscribe fails, future completes exceptionally → all waiters get the error → entry removed → next request retries fresh |

**Result:** No matter how many concurrent requests arrive for the same symbol, only **one** subscribe message is sent to LSEG.

---

## 6. WebSocket Connection Lifecycle

### 6.1 Startup
1. Application starts → `WebSocketConnectionManager` initialized (`@PostConstruct` or `SmartLifecycle`).
2. Connects to LSEG WebSocket endpoint.
3. Authenticates (token/credentials from config).
4. Connection is ready.

### 6.2 Reconnection Strategy

| Scenario | Action |
|----------|--------|
| Connection drops | Exponential backoff reconnect (1s, 2s, 4s, ... max 60s) |
| Auth failure | Log error, alert, retry with refreshed token |
| LSEG maintenance | Backoff + circuit breaker opens after N failures |

### 6.3 Heartbeat
- Send periodic ping/heartbeat per LSEG protocol spec.
- If no pong within threshold → trigger reconnect.

### 6.4 Shutdown
- `@PreDestroy`: Unsubscribe all, close connection gracefully.

---

## 7. Resilience & Error Handling

| Failure Mode | Handling |
|--------------|----------|
| WebSocket disconnected | Return stale cache (if available) + header flag `X-MarketData-Stale: true` |
| Cache miss + WS down | Return response without market data; populate `marketData: null` with error reason |
| LSEG timeout on subscribe | `CompletableFuture` times out → fallback to no market data |
| Poison message from LSEG | Log + discard; don't crash the listener |

**Circuit Breaker (optional):** Wrap WS subscribe calls with Resilience4j circuit breaker to avoid thundering-herd retries.

---

## 8. Thread Model

| Component | Threading |
|-----------|-----------|
| WebSocket listener | Single Netty/reactor thread (inbound messages) |
| Cache writes | Non-blocking via `ConcurrentHashMap.put()` |
| Cache reads | Lock-free concurrent reads |
| Cache-miss subscribe | Caller thread blocks on `CompletableFuture` (bounded by timeout) |
| Request coalescing | `computeIfAbsent` is atomic; multiple callers share one future — no extra threads |

No custom thread pools needed. Spring WebSocket client handles I/O threads internally.

---

## 9. Configuration (application.yml)

```yaml
market-data:
  lseg:
    websocket-url: wss://lseg-endpoint/market-data
    auth-token: ${LSEG_AUTH_TOKEN}
    heartbeat-interval-seconds: 30
    reconnect-max-backoff-seconds: 60
  cache:
    ttl-seconds: 30
    max-size: 10000
  subscribe:
    timeout-seconds: 5
```

---

## 10. Package Structure

```
com.example.security
├── controller
│   └── SecurityController
├── service
│   ├── SecurityService
│   └── marketdata
│       ├── MarketDataService
│       ├── WebSocketConnectionManager
│       ├── LsegMessageHandler
│       └── model
│           └── MarketData
├── repository
│   └── SecurityRepository
└── config
    └── MarketDataConfig
```

---

## 11. Key Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Singleton WebSocket connection | Avoids per-request connection overhead; respects LSEG connection limits |
| 2 | In-memory cache over distributed cache | Single instance, ephemeral data, minimal latency |
| 3 | Fetch-on-demand (not pre-subscribe all) | Only cache securities that traders actually request; keeps memory bounded |
| 4 | Request coalescing via in-flight future map | Prevents thundering herd; N concurrent requests for same symbol = 1 LSEG subscribe |
| 5 | Stale-data fallback on WS failure | Availability over consistency — slightly stale price is better than no response |
| 6 | Caffeine-compatible cache interface | Easy migration from ConcurrentHashMap to Caffeine if TTL/eviction needed |

---

## 12. Future Considerations

- **Horizontal scaling:** If multiple instances are deployed, each would hold its own WS connection and cache. Consider shared cache (Redis) only if cross-instance consistency is needed.
- **Pre-warming:** Subscribe to top-N traded securities at startup to reduce cache-miss latency for common requests.
- **Streaming mode:** If requirements evolve to push real-time updates to traders, add SSE/WebSocket endpoint on the API side.
