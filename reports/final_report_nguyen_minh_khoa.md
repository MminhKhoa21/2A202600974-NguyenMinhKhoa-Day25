# Day 10 Reliability Report

**Sinh vien:** Nguyen Minh Khoa

## 1. Tong quan

Trong bai lab nay, em xay dung mot lop reliability cho LLM gateway voi muc tieu chinh la giu he thong on dinh khi provider loi, giam chi phi bang cache, va van co kha nang phuc vu khi gap tinh huong xau. Kien truc em chon la pipeline theo thu tu `cache -> circuit breaker -> provider chinh -> provider du phong -> static fallback`.

```text
User Request
    |
    v
[ReliabilityGateway]
    |
    +--> [Cache]
    |         |
    |         +--> Hit -> tra ve ngay
    |
    +--> [Circuit Breaker: Primary] -> [Primary Provider]
    |         |
    |         +--> Loi/Open -> fallback
    |
    +--> [Circuit Breaker: Backup] -> [Backup Provider]
    |         |
    |         +--> Loi/Open -> static fallback
    |
    +--> [Static fallback]
```

Em chon kien truc nay vi no de quan sat, de test, va rat phu hop voi bai toan reliability co nhieu tang phong ve.

## 2. Cau hinh va ly do chon

| Setting | Gia tri | Ly do |
|---|---:|---|
| failure_threshold | 3 | Cho phep vai lan loi nho, nhung van mo circuit du nhanh khi provider co van de that |
| reset_timeout_seconds | 2 | De quan sat duoc qua trinh OPEN -> HALF_OPEN -> CLOSED trong chaos test |
| success_threshold | 1 | Don gian, dung voi test contract va du cho probe request |
| cache TTL | 300 | Du de query lap lai co the hit cache trong load test |
| similarity_threshold | 0.92 | Uu tien an toan, tranh semantic hit sai |
| load_test requests | 100 moi scenario | Du de lay metrics ma van chay nhanh trong local |

## 3. Ket qua chay chinh

### Memory cache run

- availability: `0.99`
- error_rate: `0.01`
- latency_p50_ms: `276.28`
- latency_p95_ms: `317.72`
- latency_p99_ms: `319.37`
- fallback_success_rate: `0.9663`
- cache_hit_rate: `0.5667`
- circuit_open_count: `8`
- recovery_time_ms: `2498.93`
- estimated_cost: `0.055474`
- estimated_cost_saved: `0.17`

### Redis cache run

- availability: `0.9833`
- error_rate: `0.0167`
- latency_p95_ms: `318.29`
- cache_hit_rate: `0.6867`
- estimated_cost: `0.038104`
- estimated_cost_saved: `0.206`

Theo em, ket qua nay cho thay cache mang lai hieu qua ro rang o goc do chi phi. Redis cache cho hit rate cao hon, va tong chi phi giam tiep so voi memory cache.

## 4. So sanh cache va khong cache

| Metric | Khong cache | Co cache | Nhan xet |
|---|---:|---:|---|
| availability | 0.9733 | 0.99 | Co cache giup he thong on dinh hon |
| estimated_cost | 0.122912 | 0.055474 | Chi phi giam manh |
| cache_hit_rate | 0.0 | 0.5667 | Co nhieu request lap lai duoc tan dung |
| circuit_open_count | 21 | 8 | Cache giam ap luc len provider |

Phan em thay ro nhat la khi tat cache, so lan circuit mo tang len rat nhieu. Dieu nay hop ly vi toan bo request deu bi day thang vao provider.

## 5. Redis shared cache

Em da xac minh `SharedRedisCache` hoat dong dung voi Docker Redis local.

- Redis test: `6/6 passed`
- Full test suite khi Redis bat: `35 passed, 7 xpassed`
- Shared state giua 2 instance:

```text
('shared response', 1.0)
```

Redis cung co key thuc te trong cache namespace, cho thay du lieu da duoc luu va chia se:

```text
rl:cache:734852f3cf4a
rl:cache:dacb2b833659
rl:evidence:11956e8badb2
...
```

Theo em, day la diem quan trong neu he thong scale theo nhieu instance, vi in-memory cache mot minh se khong du.

## 6. Chaos testing

Em da chay 4 scenario chinh:

| Scenario | Ket qua |
|---|---|
| primary_timeout_100 | Pass |
| primary_flaky_50 | Pass |
| all_healthy | Pass |
| all_providers_degraded | Pass |

Scenario `all_providers_degraded` duoc chay voi cache tat de do dung static fallback path. Ket qua availability ve `0.0`, error_rate la `1.0`, circuit open `2` lan. Dieu nay cho thay khi ca hai provider deu fail, gateway fail fast dung huong va khong retry vo han.

## 7. Concurrent load

Em chay them concurrent scenario voi `max_workers=12` tren Redis backend.

- availability: `0.98`
- error_rate: `0.02`
- latency_p95_ms: `316.55`
- fallback_success_rate: `0.9535`
- cache_hit_rate: `0.49`
- circuit_open_count: `1`

Ket qua nay cho em thay duoi tai dong thoi, gateway van giu duoc availability cao, fallback van hoat dong dung, va breaker van co tac dung cat som mot dot loi thay vi tao retry storm.

## 8. Danh gia ca nhan

Neu tu danh gia theo rubric, em cho rang bai lam da dat muc rat tot vi:

- Circuit breaker hoat dong dung 3 trang thai
- Co fallback chain ro rang
- Co semantic cache, privacy guardrail va false-hit detection
- Co Redis shared cache va evidence thuc te
- Co metrics JSON, CSV, report va concurrent load evidence

Phan em muon cai thien them neu dua len production la:

1. Tinh them `end-to-end latency` cho ca cache hit
2. Dua trang thai circuit breaker len Redis neu scale ngang
3. Bo sung cost-aware routing khi vuot budget

## 9. Ket luan

Sau khi hoan thanh bai lab, em thay gia tri lon nhat cua bai nay khong nam o tung ky thuat rieng le, ma nam o cach ket hop nhieu lop bao ve cung luc. Khi cache, circuit breaker, fallback va observability di chung voi nhau, he thong tro nen ben vung hon rat nhieu so voi mot gateway goi provider truc tiep.
