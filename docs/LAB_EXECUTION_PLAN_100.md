# Lab Execution Plan To Reach 100/100

## 1. Muc tieu

Hoan thanh lab theo dung yeu cau trong `README.md`, dat du cac deliverable va toi uu theo rubric de huong toi `100/100`, khong chi dung o muc "code chay duoc".

## 2. Ket luan nhanh sau khi phan tich repo

Trang thai hien tai:

- Repo dang o starter state, chua hoan thanh core TODO.
- Cac ham con thieu implementation:
  - `src/reliability_lab/circuit_breaker.py`
  - `src/reliability_lab/cache.py`
  - `src/reliability_lab/gateway.py`
  - `src/reliability_lab/chaos.py`
  - `src/reliability_lab/metrics.py`
- Da co san:
  - de bai chi tiet trong `README.md`
  - rubric tong trong `docs/RUBRIC.md`
  - test contract cho tung phan trong `tests/`
  - report template trong `reports/report_template.md`
  - config mac dinh va 3 chaos scenario trong `configs/default.yaml`

Rui ro hien tai:

- Chua the xac nhan test pass/fail thuc te vi may hien tai chua co `pytest` trong PATH.
- Chua the xac nhan Redis evidence vi can Docker/Redis chay that.
- `generate_report.py` chi tao report toi thieu; de dat diem cao can report dien tay theo `reports/report_template.md`.

## 3. Rubric hop nhat de huong toi 100/100

Minh hop nhat thong tin tu `README.md` va `docs/RUBRIC.md` nhu sau:

| Hang muc | Diem muc tieu | Dieu kien de an diem toi da |
|---|---:|---|
| Circuit breaker va fallback | 25 | Dung state machine 3 trang thai, co transition log, khong retry storm, route ro rang |
| Cache va cost | 20 | Similarity dung n-gram cosine, co privacy guardrail, false-hit detection, hit rate va cost saved duoc do |
| Redis shared cache | 15 | `SharedRedisCache` chay dung, shared state duoc chung minh, Redis tests pass |
| Observability va metrics | 20 | Co `metrics.json` va CSV, co P50/P95/P99, availability, cache/circuit metrics, tai lap duoc |
| Chaos/load testing | 20 | It nhat 3 scenario, co pass/fail, co recovery evidence, co so sanh cache/no-cache |
| Report va code quality | 15 | Report day du, giai thich config, failure analysis, type hints, tests pass |

Tong diem rubric tong hop vuot 100 khi cong chong cheo, vi hai file rubric khac muc chi tiet. Cach an toan nhat la lam day du tat ca bang chung ben duoi.

## 4. Dinh nghia "done" de xem nhu 100/100-ready

Bai lam duoc xem la san sang dat diem toi da khi dong thoi thoa tat ca:

- `make test` pass toan bo.
- Redis dang chay va `tests/test_redis_cache.py` khong bi skip.
- `make run-chaos` tao ra `reports/metrics.json`.
- `RunMetrics.write_csv()` tao duoc CSV tai lap.
- `reports/final_report.md` duoc dien day du theo template, co so lieu that.
- Co evidence cache vs no-cache.
- Co evidence Redis shared state.
- Co it nhat 3 scenario co expected, observed, pass/fail.
- Co failure analysis va next steps ro rang.

## 5. Gap analysis theo file

### A. `src/reliability_lab/circuit_breaker.py`

Can lam:

- `allow_request()`
- `call()`
- `record_success()`
- `record_failure()`

Muc tieu pass:

- `tests/test_circuit_breaker.py`
- xfail lien quan trong `tests/test_todo_requirements.py`

Luu y an diem toi da:

- `HALF_OPEN` fail phai mo lai voi reason `probe_failure`
- threshold fail phai reason `failure_threshold_reached`
- phai dung `if/elif`, khong gop logic

### B. `src/reliability_lab/cache.py`

Can lam:

- `ResponseCache.similarity()`
- `ResponseCache.get()`
- `ResponseCache.set()`
- `SharedRedisCache.get()`
- `SharedRedisCache.set()`

Muc tieu pass:

- `tests/test_cache.py`
- `tests/test_redis_cache.py`
- xfail lien quan cache trong `tests/test_todo_requirements.py`

Luu y an diem toi da:

- Similarity phai la cosine tren word token + char 3-gram
- Phai co `false_hit_log`
- Query privacy-sensitive khong duoc cache
- Query khac nam/ID 4 chu so phai bi chan false-hit

### C. `src/reliability_lab/gateway.py`

Can lam:

- `ReliabilityGateway.complete()`

Muc tieu pass:

- `tests/test_gateway_contract.py`
- xfail gateway trong `tests/test_todo_requirements.py`

Luu y an diem toi da:

- Thu tu dung: cache -> breaker -> provider chain -> static fallback
- `route` phan biet cache hit, primary, fallback, static fallback
- Nhan cache hit phai co score trong route

### D. `src/reliability_lab/chaos.py`

Can lam:

- `calculate_recovery_time_ms()`
- `run_scenario()`
- giu nguyen `run_simulation()` hoac nang cap pass/fail criteria ro hon

Muc tieu pass:

- `make run-chaos`
- tao `reports/metrics.json`

Luu y an diem toi da:

- Dem dung total/success/fail/fallback/cache hits/cost
- Tinh recovery time tu breaker transition logs
- Dinh nghia pass/fail tung scenario ro rang trong report

### E. `src/reliability_lab/metrics.py`

Can lam:

- `write_csv()`

Muc tieu pass:

- test xfail CSV
- rubric observability

Luu y an diem toi da:

- Flatten `scenarios` thanh `scenario_<name>`
- Ghi duoc CSV reproducible 1 row

## 6. Thu tu thuc hien toi uu

Day la thu tu minh khuyen nghi va cung la thu tu minh se coi la "quyet dinh thuc hien dung":

1. Implement `circuit_breaker.py`
2. Implement in-memory `ResponseCache`
3. Implement `gateway.py`
4. Implement `metrics.py`
5. Implement `chaos.py`
6. Implement `SharedRedisCache`
7. Chay full test
8. Chay chaos voi memory cache
9. Chay chaos voi cache disabled de so sanh
10. Chuyen sang Redis backend va chay lai
11. Dien `reports/final_report.md`

Ly do:

- Gateway phu thuoc breaker + cache
- Chaos phu thuoc gateway
- Redis la lop mo rong, co the de sau khi luong core da on dinh
- Report chi co gia tri khi da co metrics that

## 7. Step-by-step execution checklist

### Phase 0. Setup

- [ ] Tao env va cai `pip install -e ".[dev]"`
- [ ] Xac nhan `python --version`
- [ ] Xac nhan `pytest` chay duoc
- [ ] Neu lam Redis: `docker compose up -d`

Definition of done:

- [ ] `pytest --version` chay duoc
- [ ] `python -c "import reliability_lab"` khong loi

### Phase 1. Circuit breaker

- [ ] Implement `allow_request()`
- [ ] Implement `call()`
- [ ] Implement `record_success()`
- [ ] Implement `record_failure()`
- [ ] Bao dam co transition log khi doi state

Verification:

- [ ] `pytest tests/test_circuit_breaker.py -v`
- [ ] Kiem tra xfail lien quan breaker da unexpected PASS

### Phase 2. In-memory cache

- [ ] Them import `Counter` va `math`
- [ ] Them `false_hit_log` trong `__init__`
- [ ] Implement `similarity()`
- [ ] Implement `get()`
- [ ] Implement `set()`
- [ ] Bao dam query privacy-sensitive khong luu
- [ ] Bao dam false-hit theo nam/ID bi reject

Verification:

- [ ] `pytest tests/test_cache.py -v`
- [ ] Kiem tra xfail lien quan cache da unexpected PASS

### Phase 3. Gateway

- [ ] Implement cache-first lookup
- [ ] Implement provider chain qua circuit breaker
- [ ] Implement fallback sang provider thu 2 tro di
- [ ] Implement static fallback khi tat ca fail
- [ ] Cache lai response thanh cong

Verification:

- [ ] `pytest tests/test_gateway_contract.py -v`
- [ ] Kiem tra xfail gateway da unexpected PASS

### Phase 4. Metrics va chaos

- [ ] Implement `RunMetrics.write_csv()`
- [ ] Implement `calculate_recovery_time_ms()`
- [ ] Implement `run_scenario()`
- [ ] Giu lai 3 scenario co san trong `configs/default.yaml`
- [ ] Them 1 scenario rieng de an diem report, vi du `all_providers_degraded` hoac `cache_disabled_baseline`

Verification:

- [ ] `python scripts/run_chaos.py --config configs/default.yaml --out reports/metrics.json`
- [ ] Kiem tra co P50/P95/P99 trong `reports/metrics.json`
- [ ] Ghi duoc CSV metrics

### Phase 5. Redis shared cache

- [ ] `docker compose up -d`
- [ ] Implement `SharedRedisCache.set()`
- [ ] Implement `SharedRedisCache.get()`
- [ ] Chay test Redis
- [ ] Doi `configs/default.yaml` sang `backend: redis` de test shared cache path

Verification:

- [ ] `pytest tests/test_redis_cache.py -v`
- [ ] `docker compose exec redis redis-cli KEYS "rl:cache:*"`

### Phase 6. Final evidence va report

- [ ] Chay full test suite
- [ ] Chay chaos voi cache enabled
- [ ] Chay chaos voi cache disabled
- [ ] Chay chaos voi Redis backend
- [ ] Dien `reports/final_report.md` theo template
- [ ] Dam bao so lieu report khop `metrics.json`

Verification:

- [ ] `make test`
- [ ] `make run-chaos`
- [ ] `make report`

## 8. Checklist de dat 100/100 theo rubric

### Circuit breaker va fallback

- [ ] OPEN circuit tu choi request truoc timeout
- [ ] OPEN -> HALF_OPEN sau timeout
- [ ] HALF_OPEN success -> CLOSED
- [ ] HALF_OPEN failure -> OPEN voi reason `probe_failure`
- [ ] CLOSED failure du nguong -> OPEN voi reason `failure_threshold_reached`
- [ ] Co `transition_log`
- [ ] Gateway khong retry storm khi primary loi
- [ ] Gateway co fallback sang provider khac
- [ ] Route phan biet `primary`, `fallback`, `static_fallback`, `cache_hit:<score>`

### Cache va cost

- [ ] Similarity la n-gram cosine, khong phai Jaccard
- [ ] Cache co TTL eviction
- [ ] Query privacy-sensitive khong bao gio duoc cache
- [ ] False-hit do nam/ID khac nhau bi reject
- [ ] Co `false_hit_log`
- [ ] Co metric `cache_hit_rate`
- [ ] Co metric `estimated_cost_saved`
- [ ] Trong report co giai thich `ttl_seconds` va `similarity_threshold`
- [ ] Trong report co it nhat 1 vi du false-hit duoc chan

### Redis shared cache

- [ ] `SharedRedisCache.get()` hoat dong cho exact hit
- [ ] `SharedRedisCache.get()` hoat dong cho semantic hit
- [ ] `SharedRedisCache.set()` luu hash va TTL
- [ ] Guardrail privacy van duoc giu nguyen tren Redis path
- [ ] Redis tests pass
- [ ] Co evidence 2 instance cung thay 1 key cache
- [ ] Co Redis CLI output trong report

### Observability va metrics

- [ ] `reports/metrics.json` ton tai
- [ ] Co `availability`
- [ ] Co `error_rate`
- [ ] Co `latency_p50_ms`
- [ ] Co `latency_p95_ms`
- [ ] Co `latency_p99_ms`
- [ ] Co `fallback_success_rate`
- [ ] Co `cache_hit_rate`
- [ ] Co `circuit_open_count`
- [ ] Co `recovery_time_ms`
- [ ] Co `estimated_cost`
- [ ] Co `estimated_cost_saved`
- [ ] Co CSV export

### Chaos/load testing

- [ ] Co it nhat 3 scenario co ten
- [ ] Moi scenario co expected behavior
- [ ] Moi scenario co observed behavior
- [ ] Moi scenario co pass/fail
- [ ] Co recovery evidence dua tren transition log
- [ ] Co so sanh cache enabled vs disabled
- [ ] Neu muon toi da hoa rubric `docs/RUBRIC.md`, bo sung concurrent load scenario

### Report va code quality

- [ ] `reports/final_report.md` du 9 muc
- [ ] Co architecture diagram
- [ ] Co bang config + ly do chon tham so
- [ ] Co bang SLO va danh gia dat/khong dat
- [ ] Co cache comparison table
- [ ] Co Redis evidence
- [ ] Co failure analysis
- [ ] Co next steps
- [ ] Code co type hints day du
- [ ] `make test` pass
- [ ] `make typecheck` pass
- [ ] `make lint` pass

## 9. Quyết định cau hinh de huong toi diem cao

Minh de xuat giu cac quyet dinh sau, tru khi test/metrics cho thay can dieu chinh:

- `failure_threshold = 3`
  - Du nhay de mo circuit nhanh, khong qua nhay gay false open.
- `reset_timeout_seconds = 2`
  - Vua du de thay hanh vi OPEN -> HALF_OPEN trong chaos test ngan.
- `success_threshold = 1`
  - Phu hop voi bai lab va test hien tai.
- `cache.ttl_seconds = 300`
  - Don gian, de quan sat hit rate trong 100 requests.
- `cache.similarity_threshold = 0.92`
  - An toan cho false-hit; neu hit rate qua thap moi ha xuong sau khi do.
- `load_test.requests = 100`
  - Du de sinh metrics co y nghia ma van chay nhanh.

Neu muon report dep hon:

- Bo sung 1 lan chay `cache.enabled = false` lam baseline.
- Bo sung 1 scenario "all_providers_degraded" de chung minh static fallback.

## 10. Phan minh co the tu thuc hien ngay

Nhung viec minh da tu thuc hien trong lan nay:

- Doc de bai, rubric, tests, config, template report.
- Xac dinh chinh xac cac file TODO va thu tu implementation toi uu.
- Tong hop thanh runbook `.md` de ban bam sat khi lam.
- Xac dinh truoc cac bang chung can co de huong toi `100/100`.

## 11. Phan bat buoc ban phai tu thuc hien tren may

Nhung viec nay can moi truong local day du nen ban can tu chay:

- Cai dependency dev de co `pytest`, `ruff`, `mypy`
- Chay Docker/Redis
- Chay `make test`
- Chay `make run-chaos`
- Chay `make report`
- Chup/log output de nop bai

Lenh nen chay:

```powershell
pip install -e ".[dev]"
docker compose up -d
make test
make run-chaos
make report
make lint
make typecheck
```

## 12. Bao cao trang thai hien tai

Tinh den thoi diem phan tich:

- Chua dat muc nop bai.
- Core logic chua duoc implement.
- Chua xac minh duoc test vi thieu `pytest` trong PATH.
- Da co day du huong dan va tieu chi de chuyen sang giai doan implementation.

## 13. Buoc tiep theo minh khuyen nghi

Buoc tiep theo tot nhat la:

1. Cai dependency dev.
2. Implement code theo thu tu Phase 1 -> 5.
3. Chay test sau moi phase.
4. Sau khi co metrics that, moi dien `reports/final_report.md`.

Neu ban muon, buoc tiep theo minh co the lam tiep la truc tiep implement toan bo code TODO trong `src/reliability_lab/` theo plan nay.
