# Day 10 Reliability Final Report

## 1. Architecture summary

Gateway duoc thiet ke theo thu tu fail-safe:

```text
User Request
    |
    v
[ReliabilityGateway]
    |
    +--> [ResponseCache / SharedRedisCache]
    |         |
    |         +--> cache hit => tra ve ngay
    |
    +--> [Circuit Breaker: primary] --> [Provider primary]
    |         |
    |         +--> fail/open => fallback
    |
    +--> [Circuit Breaker: backup] --> [Provider backup]
    |         |
    |         +--> fail/open => static fallback
    |
    +--> [Static fallback message]
```

Muc tieu cua flow nay la giam retry storm, tan dung cache de tiet kiem chi phi, va van giu availability cao khi provider chinh bi loi.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Du nhay de mo circuit som khi provider xau, nhung khong qua nhay gay false open |
| reset_timeout_seconds | 2 | Phu hop chaos test ngan, de quan sat OPEN -> HALF_OPEN -> CLOSED |
| success_threshold | 1 | Don gian, dung voi contract test cua bai lab |
| cache TTL | 300 | Giup lap lai query trong load test van hit cache |
| similarity_threshold | 0.92 | Uu tien an toan, han che false-hit cho query gan giong |
| load_test requests | 100 moi scenario | Du mau de co metrics va van chay nhanh trong local lab |

## 3. SLO definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.0% | Yes |
| Latency P95 | < 2500 ms | 317.72 ms | Yes |
| Fallback success rate | >= 95% | 96.63% | Yes |
| Cache hit rate | >= 10% | 56.67% | Yes |
| Recovery time | < 5000 ms | 2498.93 ms | Yes |

## 4. Metrics

Nguon: `reports/metrics.json`

| Metric | Value |
|---|---:|
| availability | 0.99 |
| error_rate | 0.01 |
| latency_p50_ms | 276.28 |
| latency_p95_ms | 317.72 |
| latency_p99_ms | 319.37 |
| fallback_success_rate | 0.9663 |
| cache_hit_rate | 0.5667 |
| estimated_cost_saved | 0.17 |
| circuit_open_count | 8 |
| recovery_time_ms | 2498.93 |

Nhan xet:

- He thong giu availability 99% du primary co scenario fail nang.
- Cache hit rate cao giup giam cost ro ret.
- Circuit mo 8 lan trong 3 scenario va van hoi phuc trong khoang 2.5s trung binh.

## 5. Cache comparison

Nguon:

- `reports/metrics.json` = cache enabled
- `reports/metrics_no_cache.json` = cache disabled

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 273.35 | 276.28 | +2.93 ms |
| latency_p95_ms | 316.49 | 317.72 | +1.23 ms |
| estimated_cost | 0.122912 | 0.055474 | -0.067438 |
| cache_hit_rate | 0.0 | 0.5667 | +56.67 pts |
| circuit_open_count | 21 | 8 | -13 |

Nhan xet:

- Loi ich lon nhat cua cache trong bai nay la giam chi phi va giam so lan mo circuit.
- Percentile latency khong cai thien ro trong ket qua hien tai vi implementation metrics chi ghi latency khi co provider call; cache hit `latency_ms = 0` dang khong dua vao mang percentile. Day la mot han che do luong, khong phai do cache khong hieu qua.

## 6. Redis shared cache

Tai sao can shared cache:

- In-memory cache chi dung trong 1 process, khong chia se du lieu giua nhieu instance gateway.
- Redis giup nhieu instance thay cung mot tap cache, hop voi production co nhieu replica.

Trang thai hien tai:

- Code `SharedRedisCache.get()` va `set()` da duoc implement.
- Redis integration da duoc xac minh bang test va chaos run voi Redis backend.

### Evidence of shared state

Hai cache instance doc/ghi cung prefix da chia se du lieu thanh cong:

```text
('shared response', 1.0)
```

Redis-specific test status:

- `tests/test_redis_cache.py`: 6/6 passed
- Full suite voi Redis active: `35 passed, 7 xpassed`

### Redis CLI output

```text
rl:cache:734852f3cf4a
rl:cache:dacb2b833659
rl:evidence:11956e8badb2
rl:cache:095946136fea
rl:cache:3936614ac4c2
rl:cache:fff10da1c72c
rl:cache:4fc3c69b9376
rl:cache:0bc3b1acf73d
rl:cache:d354658dc020
rl:cache:844ef0143a5c
rl:cache:3dab98c0e49e
rl:cache:9e413fd814eb
rl:cache:98332d0d1c9c
rl:cache:da61fb49b4f6
```

Redis backend metrics (`reports/metrics_redis.json`):

- availability: 0.9833
- cache_hit_rate: 0.6867
- estimated_cost: 0.038104
- estimated_cost_saved: 0.206

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Tat ca traffic fallback sang backup, circuit primary mo | Scenario pass, gateway tiep tuc phuc vu nho fallback | Pass |
| primary_flaky_50 | Circuit dao dong, co mix giua primary va fallback | Scenario pass, co circuit open va fallback success | Pass |
| all_healthy | Chu yeu di qua primary, it loi, it/no circuit open | Scenario pass, metrics tong the dat SLO | Pass |
| all_providers_degraded | Ca primary va backup cung fail, gateway phai fail fast va tra static fallback | Chay voi cache tat, availability = 0.0, error_rate = 1.0, circuit_open_count = 2, static fallback duoc kich hoat | Pass |

### Concurrent load evidence

Concurrent run duoc thuc hien bang `run_concurrent_scenario(..., max_workers=12)` voi Redis backend va scenario `primary_flaky_50`.

Ket qua tu `reports/metrics_concurrent.json`:

| Metric | Value |
|---|---:|
| availability | 0.98 |
| error_rate | 0.02 |
| latency_p95_ms | 316.55 |
| fallback_success_rate | 0.9535 |
| cache_hit_rate | 0.49 |
| circuit_open_count | 1 |
| estimated_cost | 0.019286 |

Nhan xet:

- Duoi concurrent pressure, gateway van giu availability 98%.
- Fallback van hoat dong tot tren 95%.
- Redis cache van co hit rate 49%, cho thay shared cache van giup giam cost ngay ca khi co nhieu request dong thoi.
- `circuit_open_count = 1` cho thay breaker van cat som mot dot loi thay vi de retry storm lan rong.

## 8. Failure analysis

Diem yeu con lai la observability latency cho cache hit chua phan anh dung trai nghiem end-to-end. Hien tai `run_scenario()` chi dua latency > 0 vao `latencies_ms`, nen cache hit bi loai khoi P50/P95/P99. Neu dua di production, minh se tinh latency toan request, ke ca cache hit, va tach rieng `provider_latency_ms` voi `end_to_end_latency_ms`.

Mot rui ro khac la circuit state hien van local theo process. Neu scale ngang nhieu instance, moi instance tu mo/tu dong circuit rieng. Huong nang cap phu hop la dua breaker counters/state vao Redis.

## 9. Next steps

1. Bo sung metric end-to-end latency cho cache hit de cache comparison chinh xac hon.
2. Dua breaker state vao Redis neu muon scale ngang nhieu gateway instance.
3. Them policy budget-aware routing de uu tien cache/backup khi cost vuot nguong.
