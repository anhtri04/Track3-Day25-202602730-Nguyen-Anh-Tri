# Lab 25B — Report

**Họ tên / nhóm:** Nguyen Anh Tri — 202602730  **Commit:** a3f9ce7  **Điểm tự động (`make grade`):** 84 / 88

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL (avail, retry_after_violation=36027, ampl) |
| G1 | yours | 1.0 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS (violations=0) |
| G2 | naive | 0.456 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL (avail, ampl, recovery) |
| G2 | yours | 1.0 | 10.92 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL (retry_ratio 0.164 > 0.10) |
| G3 | yours | 1.0 | 4.66 | 1.06 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL (p95 13.7 > 10) |
| G4 | yours | 1.0 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.03 | 16.03 | 2.124 | None | FAIL (client_error_retries, avail) |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS (0 client_error_retries, 0 unneeded failover) |
| G6 | naive | 0.517 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL (avail) |
| G6 | yours | 1.0 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS (incompatible_routes=0) |

Nguồn: `reports/gameday/naive.json`, `reports/gameday/resilient.json` (seed 25). `make grade`: 6/6 tapes PASS, workflows PASS, SLO public PASS, C1 PASS → 84/88 (4đ hidden SLO do giảng viên chấm).

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | G6 `alpha-eu` treo vô hạn: timeout 8s fail-over kịp → avail 1.0; timeout 30s thử nghiệm avail rớt còn 0.9488, 109 deadline give-up. G4 slow+5s: p95 5.56 với timeout 8 (breaker slow-trip) vs naive 13.7. p99 bình thường ~5.8 (G1) nên 8 là ~p99+margin, vừa dưới queue-wait 10s (tránh 529). |
| max_attempts_per_provider | 2 | G3 retry_ratio 0.034 (≤0.10) với 1 retry/provider; =5 + breaker tắt thì retry_ratio 0.068–0.495, ampl 1.38–1.67 (thí nghiệm §3.1). 1 retry đủ cho brown-out 30% 500, fail-over xử lý phần còn lại. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full-jitter `U(0,min(cap,base·2ⁿ⁻¹))`; retry-after luôn `retry_after+U(0,0.25)`. G1 0 violation (naive 36027) nhờ tôn trọng retry-after + jitter chống thundering herd. Cap 4s vừa deadline 30s cho 2 step × (retry+failover). |
| retry_budget_ratio | 0.1 | G2/G3 ampl ≤1.06 dù breaker tắt vẫn ampl 1.38 (budget giữ). Tắt budget → ampl 1.675, retry_ratio 0.495, recovery 25s (G2). Lớp phòng thủ cuối khi breaker misconfigured. |
| retry_budget_min_per_s / max_tokens | 0.5 / 20 | Đủ cho traffic yên tĩnh vẫn retry lẻ; 20 token đủ burst ngắn, không đủ storm. |
| quota_cooldown_s | 300 | G5 sau bad-deploy hết quota: cooldown dài tránh quay lại alpha trong 300s; test `quota_exhausted` tại t=200 vẫn đi alpha-eu. Không chờ quota khi ladder cạn (chỉ chờ rate-limit ≤5s). |
| breaker.window_size / min_calls | 20 / 10 | Không trip lúc 3h sáng (2 lỗi lẻ); đủ nhanh trip trong incident (G2 trip sau ~10 call). |
| breaker.failure_rate_threshold | 0.5 | G3 30% 500 đơn lẻ không trip (đúng — chỉ cần retry, retry_ratio thấp); G2 529/timeout dày → trip. Phân biệt brown-out vs overload. |
| breaker.slow_call_s | 5.0 | G4 không lỗi nào nhưng latency +5s: slow-rate trip → fail-over → p95 5.56 vs naive 13.76. Ngưỡng trên service_median 1.2–1.8s + queue, dưới timeout 8s. |
| breaker.slow_rate_threshold | 0.6 | Chịu được vài call chậm lẻ, trip khi đa số chậm. |
| breaker.open_duration_s | 10.0 | G2 recovery 0s (naive 155s): mở 10s đủ để queue provider xả, probe lại nhanh, ≥80% traffic về alpha sau restore (G1 primary_share 0.958, G2 1.0). |
| breaker.half_open_max_probes | 2 | Giới hạn probe, tránh herd khi provider vừa hồi phục; test 50 concurrent chỉ 2 probe qua. |
| breaker.half_open_successes | 2 | Cần 2 probe tốt mới close, chống flapping; probe hỏng/chậm → OPEN lại ngay. |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```
classify(err) ─┬─ CLIENT_ERROR (400/401/...) ──→ breaker.release ──→ GiveUp("client_error") [no retry, no failover]
               ├─ QUOTA_EXHAUSTED (429 insufficient_quota) ──→ release + cooldown=now+300s ──→ failover (idx+1, _route)
               ├─ RATE_LIMITED (429 + retry_after) ──→ release + cooldown=now+retry_after ──→ failover ngay (delay=0)
               └─ TRANSIENT (OVERLOADED 529/503, SERVER 500/502/504, TIMEOUT 0/408)
                    ──→ breaker.on_failure
                    ──→ tries+=1; if tries<2 AND budget.try_acquire AND now+backoff+1.0<=deadline:
                           Send(same provider, delay=backoff.delay(tries, retry_after))
                        else: idx+=1, tries=0 → _route (skip cooldown/breaker-OPEN, else GiveUp;
                              nếu cạn ladder: chờ rate-limit cooldown ngắn nhất ≤5s nếu breaker CLOSED và kịp deadline)
on_success: breaker.on_success(permit, latency)  // slow=latency>=5.0
on_cancel: breaker.release(permit)  // slot probe HALF_OPEN
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Giả lập breaker cấu hình sai (`breaker.min_calls: 1000000000`) và đặt
   `max_attempts_per_provider: 5`. Chạy G2, G3 hai lần: (a) giữ retry budget; (b) tắt budget
   (`retry_budget_ratio: 100`, `retry_budget_max_tokens: 1.0e9`, `retry_budget_min_per_s: 1.0e6`).
   So sánh amplification, retry_ratio, recovery. Khi breaker *hoạt động đúng* thì budget còn quan trọng không — vì sao
   vẫn nên có cả hai?
   - Kết quả (seed 25): G2 giữ budget: avail 0.9987, ampl 1.38, retry_ratio 0.058, recovery 0s, p95 19.17. Tắt budget: avail 0.9666, ampl 1.675, retry_ratio 0.495, recovery 25s, 79 deadline give-up. G3 giữ: ampl 1.136/retry 0.068; tắt: ampl 1.162/retry 0.159.
   - Budget cắt retry cùng-provider về ~10% traffic (token bucket) nên dù `max_attempts=5` cũng không storm. Tắt budget, mỗi call retry tới 5 lần → tải giữ trên năng lực → recovery kéo dài (metastable).
   - Khi breaker đúng, budget gần như không kích (retry_ratio 0.007–0.034) nhưng vẫn cần cả hai theo defense-in-depth: breaker phản ứng theo sức khỏe provider (mili-giây–giây, có false-negative khi ngưỡng sai/thay đổi traffic), budget là giới hạn cứng theo tỷ lệ không phụ thuộc phát hiện sự cố. Breaker sai config là lỗi thường gặp (min_calls quá lớn) — budget cứu.

2. **`attempt_timeout_s: 30`.** Chạy G6 (và G4 để đối chứng). Nhìn **availability** và `give_up_reasons`
   trước, rồi mới nhìn p95. Lưu ý: `latency_p95_s` chỉ tính trên *lượt thành công*, nên khi nhiều lượt treo 30 s
   rồi hết hạn, p95 có thể *giảm* trong khi availability tụt (survivorship bias). Vì sao G4 gần như không đổi?
   Timeout "đúng" nên dựa vào đâu (p99 lúc bình thường, deadline của user, hay thời gian cần cho một lần fail over)?
   - G6 timeout 30: avail 0.9488 (gốc 1.0), give_up `deadline:109 + no_provider:13`, p95 5.31 (gốc 5.96). G6 provider A sập + alpha-eu treo: attempt treo 30s ăn hết deadline 30s của user → không còn thời gian fail-over sang beta → chết. p95 *giảm* vì chỉ lượt nhanh (sang beta sớm/chỉ chat) sống sót — survivorship bias.
   - G4 timeout 30: avail 1.0→1.0, p95 5.60→5.56, gần như không đổi vì G4 chỉ chậm thêm 5s (service ~6–7s), cả timeout 8 và 30 đều chờ được; breaker slow-trip (5s) đã fail-over nên timeout dài không kích.
   - Timeout đúng = min(p99 bình thường + margin, thời gian còn lại cho ≥1 fail-over). Ở đây p99 G1 ~5.8s, queue-wait 10s (quá là 529), deadline turn 30s/2 step ≈ 15s/call → chọn 8s: trên slow_call 5s, dưới 10s, còn ~20s cho retry+fail-over.

3. **Bỏ lọc tool calling khỏi `ladder()`.** Chạy G6. Giải thích con số `incompatible_routes` bằng ngôn ngữ của
   một product manager.
   - ladder không lọc: G6 avail 1.0→0.9019, `incompatible_routes=234`, ampl 1.046.
   - Nói với PM: “ShopAgent có 2 việc: (1) nhờ AI bấm nút hoàn tiền (cần bản AI biết bấm nút), (2) viết câu trả lời. Bản giá rẻ gamma-mini không biết bấm nút. Khi bỏ bộ lọc, lúc đông khách hệ thống đẩy việc (1) sang bản giá rẻ → 234 lượt bấm nút hỏng ngay lập tức, khách thấy ‘xin lỗi, không hoàn tiền được’ dù máy chủ vẫn khỏe. Lọc tương thích là dây an toàn: việc cần tool chỉ đi alpha/alpha-eu/beta.”

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 (500/500) | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.0 (500/500) | 66 | 0 | 0 | 0 | 0 | 0.303 |

Nguồn `reports/workflows.json` (500 sessions, seed 25). Store writes yours=2500 (5 step × 500).

Trả lời: vì sao chỉ idempotency key là **chưa đủ**, và chỉ checkpoint cũng **chưa đủ**? Điều gì xảy ra nếu ghi
checkpoint *trước* khi gọi payment?

- Chỉ idempotency key (không checkpoint): không trùng tiền (payment dedup theo `session:step`) nhưng mỗi lần crash worker mới chạy lại từ đầu → LLM classify/summarize + orders.get chạy lại → `wasted_llm_per_crash` cao (naive 1.12 vs yours 0.30). Key không nhớ *kết quả* các bước không có side-effect.
- Chỉ checkpoint (key random mỗi lần): sau crash, worker mới không có checkpoint của `issue_refund` (vì crash ngay sau khi tiền đi, trước khi checkpoint) → gọi payment lại với key mới → provider tưởng là hoàn tiền khác → trùng tiền (128 trùng ở naive). Checkpoint chỉ giúp *bỏ qua* bước đã xong, không giúp *dedup* bước đang dở.
- Ghi checkpoint *trước* khi gọi payment: worker crash sau khi ghi nhưng trước/khi đang gọi payment → worker mới thấy checkpoint đã có → bỏ qua payment → mất tiền hoàn (lost_refund) dù khách đáng được hoàn. Thứ tự đúng: execute → checkpoint sau thành công; retry dùng cùng key.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page delay 1 | page delay 27 | 27 |
| provider_outage | page | page delay 0 | page delay 4 | 4 |
| deploy_blip | none | PAGED (false) | none | - |
| slow_leak | ticket | miss | ticket delay 1337 | 1337 |
| degraded_evening | page | page delay 214 | page delay 291 | 291 |
| night_spike | none | PAGED (false) | none | - |
| small_bump | none | PAGED (false) | none | - |

Naive: pages=7, false_pages=3 → FAIL. Burn-rate: pages=3, tickets=1, false_pages=0 → PASS (`reports/slo.json`).

Error budget 30 ngày: requests 25901755, allowed_bad 129508.8, bad 55618, availability 0.997853, consumed 0.4295 (42.95%), remaining 0.5705 (57.05%). `freeze_releases=False`. Có đóng băng release không? Không — còn 57% budget.

Vì sao `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa? Burn-rate 1h ≈ (12%×20/60)/0.5% ≈ 8× < 14.4 (fast-burn) và burn 6h ≈ 1.3× < 6 → không đủ *cả* significance (long window) lẫn still-burning (short window). Đánh thức là false page: sự cố tự hết, không đốt đáng kể budget tháng (20 phút × 12% ≪ 0.5%×30 ngày). SRE Workbook dùng 2 cửa sổ để chỉ page khi vừa lớn vừa còn cháy; blip ngắn → không page.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):** Lúc 14:00 UTC alpha mất 75% GPU trong 3 phút (trigger). NaiveClient với 3 tầng retry (36 attempt/call), delay cố định, không jitter/budget/breaker đã khuếch đại tải 9.9× (34k attempt cho 3.4k call) và vi phạm retry-after 28k lần, biến brown-out 3 phút thành sập 5+ phút. Sau khi provider hồi phục lúc t=300, hàng đợi vẫn đầy request mà client đã bỏ (server vẫn xử lý) + retry tiếp tục → tải > năng lực → metastable, recovery 155s, availability 45.6%.

**Ảnh hưởng:** 2362 lượt user, chỉ 1077 thành công → 1285 lượt thất bại (54.4%) trong ~8 phút. cost/turn 4.627 (gấp 2.2× bình thường ~2.1). Giả sử tháng chỉ có sự cố này: traffic tháng ≈ 4 rps×2.6M s ≈ 10.4M turn, lỗi 1285/10.4M ≈ 0.012% → ~2.5% monthly budget (0.5%). Nhưng trong 8 phút incident, error ratio ~54% ≫ SLO 0.5% (burn ~108×).

**Timeline** (giây ảo — lấy từ CSV):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 0–120 | Bình thường, alpha gánh hết | calls_ok≈calls_done (6–7/30s), attempts_alpha ~5–9/30s |
| 120 | alpha mất 75% năng lực | attempts_alpha vọt 163/30s tại t=150, calls_ok=0/3 done tại t=150 |
| 150–300 | Retry storm: 529+timeout+rate-limit, retry ngay, không jitter | attempts 105–163/30s, success 0, errors `overloaded 3194 + timeout 2737 + rate_limit 26298`, violations 28381 |
| 300 | alpha hồi phục hoàn toàn | cấu hình restore_at=300 |
| 300–455 | Hệ thống vẫn chưa hồi phục — vì sao? | t=330 attempts 6 nhưng done 1; t=360 attempts 50 done 1; t=390 attempts 139 done 6; recovery_s=155 |
| 455+ | Xả hết queue mới hồi phục | sau 155s success ratio 30s mới ≥95% |

**Trigger vs root cause:** trigger là gì? root cause là gì? Vì sao provider đã hồi phục mà hệ thống vẫn sập
(metastable failure — vòng lặp nào tự duy trì?).
- Trigger: alpha mất 75% GPU 120–300s (yếu tố ngoài).
- Root cause: thiết kế client: 3 tầng retry chồng (app×gateway×SDK = 36 attempt), delay cố định không jitter (đồng loạt quay lại), không tôn trọng retry-after, không retry budget, không breaker/fallback (100% attempts vẫn vào alpha — `attempts_by_provider alpha:34475`). Trigger nhỏ + khuếch đại 9.9× = sập lớn.
- Metastable: (1) client timeout nhưng server vẫn xử lý → công phí; (2) queue đầy request-chết → request mới chờ >10s → 529/timeout → retry thêm → queue càng đầy. Provider khỏe lại nhưng tải retry (giữ > năng lực 18 slot) tự duy trì sập. Chỉ hết khi retry cạn (deadline) + queue xả — mất 155s.

**Điều gì giúp / điều gì làm tệ hơn:** Giúp: không có — naive không có cơ chế nào (không fail-over sang alpha-eu/beta rỗng, không cooldown). Làm tệ hơn: retry 36×, delay cố định (đồng pha), bỏ qua retry-after (36k violation tự DDoS), timeout client nhưng server vẫn chạy (wasted work), không breaker nên mọi retry vẫn vào alpha đang nghẹt.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Chỉ 1 tầng retry + full jitter + tôn trọng retry-after + budget 10% (Finagle) | prevent | Client team | G2 ampl ≤1.2, violations=0 trên CI (hiện 1.041/0) |
| 2 | Breaker sliding-window (failure 50% + slow 5s/60%, half-open 2 probe) + fallback ladder có kiểm tra tool + cooldown rate-limit | mitigate | Client team | G2 recovery ≤30s, primary_share ≥80% (hiện 0s/100%); G4 p95 ≤10s |
| 3 | Dashboard attempts/s + success-ratio 30s + alert burn-rate (fast 14.4× 1h/5m, slow 6× 6h/30m) thay vì ngưỡng 5m | detect | SRE | deploy_blip/night_spike không page, outage page ≤5 phút (hiện 4 phút, 0 false) |

## 7. Một điểm yếu còn lại

G2 p95 của client resilient vẫn 10.92s (ngưỡng PASS nhưng cao, gần 2× các băng khác ~4–6s; p99 15.7s). Nguyên nhân: khi alpha nghẹt, fail-over sang alpha-eu (1.5s) + beta (1.8s, đắt 1.4/call) vẫn phải chờ queue + 1 retry cùng-provider (max_attempts=2) trước khi đi; cost/turn G2 2.119 cao hơn G1 2.101. Nếu cả 4 provider cùng chậm (không mô phỏng ở đây), breaker tất cả OPEN → GiveUp hàng loạt.
Sửa: (a) deadline-aware: bỏ retry cùng-provider khi `time_left < 2×p99` mà fail-over ngay; (b) hedged request cho chat (sau 2s chưa về gửi thêm 1 bản sang gamma-mini, lấy bản về trước) vì chat không cần tool và gamma rẻ/nhanh (0.6s); (c) adaptive timeout theo p99 EWMA thay vì 8s cố định.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.0 | 0 | 0 | 0 | 1.386 | FAIL (faq<0.93, hit<0.5) |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

Config `cache: similarity_threshold 0.90, ttl_s 60, max_stale_s 600, max_entries 500`.

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.90 | 0.80 → wrong 181 (`other_question`: refund_policy vs refund_time, ship city vs province — centroid 0.86); 0.90 → wrong 0, hit 0.652; 0.97 → wrong 0 nhưng hit rớt 0.579, cost tăng 0.915→0.965. Paraphrase cùng intent 0.975–0.99 nên 0.90 giữ paraphrase, loại near-miss. |
| ttl_s | 60 | 300 → outdated 95 (>15 FAIL) vì ship-fee đổi lặng lẽ t=480 không webhook, entry cũ sống tới 780s; 60 → outdated 3; 30 → outdated 2 nhưng hit 0.592 (thấp hơn 0.652), cost 0.955. 60 cân bằng freshness/hit; webhook xử lý refund-đổi t=150 nên TTL chỉ cần chặn silent-update. |
| max_stale_s | 600 | 0 → stale 0, faq_avail 0.8733 (FAIL); 600 → stale 157, faq_avail 0.9645 PASS. Outage 240–360s (120s) cần cửa sổ stale ≥120s + TTL; 600 phủ outage + trễ restore, personal/tool-call vẫn fail (đúng) nên faq không thể 100%. |
| max_entries | 500 | Catalog ~20 intent × vài paraphrase ≪500, không evict trong tape 720s; giữ LRU an toàn cho prod. |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Chạy C1 với `similarity_threshold` = 0,80 / giá trị của bạn / 0,97. Ghi hit ratio, `wrong_answers`, cost.
   Câu nào bị trả nhầm ở 0,80 (gợi ý: `wrong_answers_by_reason`, các cặp "gần mà khác")?
   - 0.80: hit 0.659, wrong 175 (`other_question`), cost 0.978, faq 0.8605. 0.90: hit 0.652, wrong 0, cost 0.915, faq 0.9645. 0.97: hit 0.579, wrong 0, cost 0.965, faq 0.964.
   - Ở 0.80 các cặp near-miss (refund_policy↔refund_time cùng source policy/refund nhưng khác câu; ship_city↔ship_province; exchange_size↔color) bị lẫn vì centroid chúng 0.86 > 0.80. 0.97 quá chặt: paraphrase xa (0.975) vẫn hit nhưng biên mỏng + nhiễu embedding → miss nhiều → hit rớt, cost tăng.

2. **TTL.** Chạy với `ttl_s` = 30 / 60 / 300. Vì sao webhook không giúp được gì cho `outdated_answers`, và TTL là thứ
   duy nhất giới hạn nó?
   - 30: hit 0.592/outdated 2/cost 0.955; 60: 0.652/3/0.915; 300: 0.731/95/0.863. TTL dài hit cao, cost thấp nhưng outdated vọt.
   - Webhook chỉ bắn cho refund-đổi t=150 (`on_content_update(policy/refund)` → invalidate), còn ship-fee đổi t=480 là `silent_update` (không webhook). Entry ship cũ chỉ hết hạn khi `age>ttl+stale` và stale chỉ dùng khi provider sập; lúc provider khỏe, lookup chỉ trả fresh (`age≤ttl`) → TTL là trần duy nhất của outdated. TTL 300 giữ entry 480→780s suốt tape → 95 outdated.

3. **Stale-if-error.** Chạy với `max_stale_s: 0`. `faq_availability` thay đổi thế nào?
   - 600→0: stale 157→0, faq 0.9645→0.8733 (−9.1pp, FAIL), avail tổng 0.9124→0.8565, give_up 246→403. Trong outage 120s mọi provider refuse, chỉ còn cache-stale cứu FAQ shared (personal + tool-call đúng là phải fail). Không stale = mất đúng phần outage.

Trả lời:

- Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") **không** được cache, kể cả khi cache theo từng user?
  Hai user khác nhau có embedding ~0.99 (chỉ khác số order, `toward(...,0.997)`) → threshold nào cũng lẫn nếu cache chung; cache theo user vẫn sai khi cùng user hỏi 2 đơn khác nhau (#123 vs #124 cũng ~0.99) → trả nhầm đơn/tiền/điểm của chính mình. Ngoài ra answer grounded `account/{user}` là PII — lộ qua cache là vi phạm; personal_share 25% + FAQ 60% nên kể cả cache đúng vẫn chỉ cứu được shared. Vì vậy `personal → never store/lookup` (test `other_user`).

- Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?
  Không alert theo `availability/faq_availability` (được cache che). Alert theo `stale_served` rate, `cache_hit_ratio` spike bất thường, và provider-side `attempts/errors (529/503)`, `give_up no_provider_available` (246 ở C1), breaker OPEN count. Cụ thể: page khi `stale_served/faq_turns > 5% trong 5 phút` hoặc burn-rate trên *provider error ratio* (loại trừ Serve), ticket khi hit_ratio tăng đột biến mà attempts rớt — tức “xanh nhờ stale”.
