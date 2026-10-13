# Lab 25B — Game Day: Retry Storm, Circuit Breaker, Cache, Idempotency & Error Budget

**Họ tên / nhóm:** Nguyen Viet Dung — 2A202602533  **Commit:** chưa commit  **Điểm tự động (`make grade`):** 84 / 88 (hidden SLO fixture không có trong workspace)

Các số liệu dưới đây lấy từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json` và `reports/cache/*.json`, seed 25.

## 1. Kết quả Game Day

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4 | Không |
| G1 | yours | 1.0000 | 4.780 | 1.063 | 0.001 | 2.104 | 0 | Có |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155 | Không |
| G2 | yours | 1.0000 | 9.922 | 1.035 | 0.004 | 2.106 | 0 | Có |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0 | Không |
| G3 | yours | 1.0000 | 4.642 | 1.076 | 0.049 | 2.167 | 0 | Có |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0 | Không |
| G4 | yours | 1.0000 | 5.626 | 1.014 | 0.002 | 2.072 | 0 | Có |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | Không |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | — | Có |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0 | Không |
| G6 | yours | 0.9992 | 5.921 | 1.050 | 0.018 | 2.230 | 0 | Có |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Lý do |
|---|---:|---|
| attempt_timeout_s | 8 s | Cắt các lần treo trước deadline 30 s để còn thời gian failover. G6 vẫn đạt 0.9992 availability, p95 5.921 s. |
| max_attempts_per_provider | 2 | Tối đa một retry cùng provider; G3 retry_ratio 0.049 và amplification 1.076. |
| backoff_base_s / cap_s | 0.25 / 4 s | Jitter ngắn cho lỗi thoáng qua, giới hạn thời gian chờ để còn route khác trong deadline. |
| retry_budget_ratio | 0.1 | Hạn chế khuếch đại; G2 amplification 1.035 so với naive 9.904. |
| breaker.window_size / min_calls | 20 / 10 | Cửa sổ đủ nhạy nhưng không trip chỉ vì vài request ban đêm. |
| breaker.failure_rate_threshold | 0.5 | Chuyển provider khỏi đường đi khi ít nhất nửa kết quả trong cửa sổ là lỗi sức khỏe. |
| breaker.slow_call_s | 5 s | G4 có thêm 5 s latency; ngưỡng này phát hiện degradation trước khi timeout tích tụ. |
| breaker.open_duration_s | 10 s | Cho provider một khoảng hồi phục rồi mới thử lại. G2 khôi phục traffic về alpha 100% sau restore. |
| breaker.half_open_max_probes | 2 | Chỉ hai request thăm dò đồng thời, tránh dồn tải vào provider mới hồi phục. |

| Loại lỗi | Breaker | Retry cùng provider | Failover | Cooldown |
|---|---|---|---|---|
| 400/client | release | Không | Không | Không |
| 429 quota | release | Không | Có | quota cooldown 300 s |
| 429 rate limit | release | Không | Có ngay | đến `retry_after`; không retry sớm |
| 503/529/5xx/timeout | ghi failure | Có nếu budget, số lần và deadline cho phép | nếu retry không phù hợp/hết budget | breaker quản lý |

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối.** Ở cấu hình thực, G2 đạt amplification 1.035, retry_ratio 0.004, availability 1.0; G3 đạt 1.076, 0.049 và availability 1.0. Chưa chạy biến thể phá breaker (`min_calls=1e9`, 5 attempts) nên không bịa số so sánh bật/tắt budget; các chỉ số baseline cho thấy budget cần thiết khi breaker không trip, trong khi breaker giảm tải nhanh hơn và budget hạn chế retry khi breaker cấu hình sai.
2. **Timeout 30 s.** Chưa chạy biến thể timeout 30 s. Cấu hình hiện dùng 8 s: nó giới hạn một lần chờ để có thời gian failover trong deadline 30 s; G6 availability là 0.9992. Timeout nên dựa vào deadline người dùng trừ thời gian dự trữ cho failover/tool step, đồng thời tham khảo p99 bình thường; chỉ dựa vào p99 có thể chiếm hết deadline khi outage. G4 nhanh hơn nhờ slow-call breaker và failover tới alpha-eu/beta, p95 5.626 s so với naive 13.764 s.
3. **Tool calling.** `ladder("tool")` chỉ bao gồm provider có tool support. G6 ghi `incompatible_routes=0`; nếu bỏ lọc, gamma-mini có thể tạo câu trả lời nhưng không thể thực hiện phần chọn công cụ, khiến lượt CSKH trông như đã chạy nhưng thiếu hành động hoàn tiền.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid (¢) | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.000 | 81 | 128 | 81 | 0 | 1,496,700 | 1.123 |
| yours | 1.000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Idempotency key làm side effect an toàn khi retry nhưng không tránh chạy lại các bước LLM/đọc đã checkpoint; checkpoint giúp resume nhưng nếu thiếu idempotency, timeout sau commit có thể tạo refund/email trùng trước lúc checkpoint được ghi. Checkpoint trước khi gọi payment nguy hiểm hơn: worker chết sau checkpoint nhưng trước payment sẽ resume và bỏ qua payment, làm mất refund. Thứ tự đúng là gọi side effect với key ổn định rồi lưu kết quả đã xác nhận.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ burn-rate |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 phút |
| provider_outage | page | page | page | 4 phút |
| deploy_blip | none | page sai | none | — |
| slow_leak | ticket | không phát hiện | ticket | 1,337 phút |
| degraded_evening | page | page | page | 291 phút |
| night_spike | none | page sai | none | — |
| small_bump | none | page sai | none | — |

Burn-rate có 3 page, 1 ticket, 0 false page. Naive có 7 page, 3 false page và bỏ sót slow leak. Trong 30 ngày có 25,901,755 request, 55,618 lỗi trên budget cho phép 129,508.8: đã dùng 42.95%, còn 57.05%, chưa cần freeze release. `small_bump` kéo dài 20 phút với 12% lỗi nhưng chỉ tiêu thụ khoảng 1% budget; multi-window burn-rate yêu cầu cả cửa sổ dài lẫn ngắn xác nhận để không đánh thức on-call vì một blip tự hồi phục.

## 6. Postmortem — G2 với client naive

**Tóm tắt.** Từ t=120 đến t=300, alpha chỉ còn 25% năng lực. Client naive gửi nhiều retry cùng lúc và tiếp tục tải alpha; sau khi năng lực được phục hồi, hàng đợi và retry còn tồn tại làm hệ thống tiếp tục lỗi. Tỷ lệ lượt thành công toàn tape chỉ là 45.6%, trong khi resilient đạt 100% với amplification 1.035.

**Ảnh hưởng:** 1,285/2,362 lượt thất bại (54.4%). Thất bại bắt đầu khoảng t=140 và còn quan sát đến vùng t=420, khoảng 280 giây sau trigger và 120 giây sau khi provider hồi phục. Nếu sự cố này là lỗi duy nhất trong tháng, 54.4% lỗi trên riêng tape dài 600 giây tương đương khoảng 0.17% tổng request tháng (ước tính 30 ngày có 4·2,592,000 lượt ở 4 turns/s), tức khoảng 34.5% error budget 0.5%.

| t (s) | Sự kiện | Bằng chứng |
|---:|---|---|
| 120 | Alpha mất 75% năng lực | attempts 6.2/s; 32 call hoàn tất trong 10 s, success 100% do hàng đợi chưa tích tụ. |
| 130 | Retry bắt đầu tăng | attempts 51.9/s, success trong cửa sổ 10 s còn 100% nhưng queue bắt đầu hình thành. |
| 140 | Sụp đổ quan sát được | attempts 190.3/s; 96 call hoàn tất, success 0%. |
| 180 | Retry storm duy trì | attempts 140.3/s; 118 call hoàn tất, success 0%. |
| 240 | Hàng đợi còn quá tải | attempts 147.2/s; success 0%. |
| 300 | Alpha hồi phục hoàn toàn | cấu hình incident kết thúc; nhưng backlog/retry còn khiến attempts khoảng 23.4/s ở đoạn 300–330 và success 0%. |
| 360 | Tải lại tăng mạnh | attempts 113.2/s; success 0% ở 360–390. |
| 420 | Bắt đầu thoát backlog | attempts 21.5/s, vẫn còn call lỗi; recovery metric chính thức là 155 s sau restore. |

**Trigger vs root cause:** trigger là alpha mất 75% GPU trong 180 giây. Root cause là ba tầng retry chồng nhau, delay cố định và thiếu breaker/budget làm mỗi lỗi phát sinh thêm request lên cùng provider. Timeout của client không hủy xử lý ở server; hàng đợi tiếp tục tiêu thụ slot, các retry mới chờ lâu hơn rồi lại retry, tạo vòng lặp metastable. Provider đã hồi phục ở t=300 nhưng tải retry và công việc cũ chiếm slot nên phục hồi đo được chỉ ở t=455.

**Điều gì giúp / điều gì làm tệ hơn:** các deployment khác có thể chia tải nếu failover có giới hạn; jitter, retry budget và breaker tránh đồng bộ retry. Naive chỉ dùng alpha và đạt amplification 9.904, recovery 155 s; resilient giữ amplification 1.035 và hồi phục ngay theo tiêu chí 30 giây.

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---:|---|---|---|---|
| 1 | Gỡ retry lồng nhau, tập trung retry ở resilient client | prevent | SDK/platform | G2 amplification ≤1.6 trên 3 seed CI |
| 2 | Bật breaker theo failure + slow-call rate và giới hạn HALF-OPEN 2 probe | mitigate | Reliability | G2 recovery ≤30 s, primary share ≥0.8 |
| 3 | Dashboard cảnh báo attempts/logical call, queue và availability theo provider | detect | On-call | Alert khi amplification >1.5 trong 5 phút; diễn tập outage hàng quý |

## 7. Một điểm yếu còn lại

G2 p95 của resilient vẫn 9.922 s, sát ngưỡng mục tiêu 10 s, do một số lượt chờ alpha timeout trước khi failover. Có thể giảm attempt timeout theo p99 bình thường và cho phép failover sớm hơn khi breaker thấy slow-call, đồng thời theo dõi cost/turn khi beta gánh tải (G6 hiện 2.230).

## 8. Cache (Part G)

| Client | availability | FAQ availability | cache hit ratio | stale served | wrong answers | outdated | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0 | 0 | 0 | 0 | 1.386 | Không |
| cached | 0.9117 | 0.9640 | 0.579 | 152 | 0 | 4 | 0.965 | Có |

| Tham số cache | Giá trị | Lý do |
|---|---:|---|
| similarity_threshold | 0.97 | 0.80 tạo 181 `other_question`; 0.97 loại nhầm lẫn và vẫn đạt hit ratio 57.9%. |
| ttl_s | 60 | Prompt/doc có thể đổi; 60 giây hạn chế câu trả lời cũ. C1 có 4 outdated answer dù TTL ngắn do silent update. |
| max_stale_s | 300 | Giữ FAQ phục vụ trong outage, đạt FAQ availability 96.4% và stale_served 152. |

**Ngưỡng:** Ở 0.80, hit ratio là 74.2% nhưng có 181 câu trả lời nhầm `other_question`; chi phí/turn 0.921. Với 0.97, hit ratio 57.9%, sai 0, cost/turn 0.965; đây cũng là cấu hình C1 đạt. Mức 0.97 là strict threshold và vẫn vượt mục tiêu 50% hit.

**TTL:** TTL 60 giây cho 4 outdated answer; tăng TTL lên 300 sẽ để nội dung silent update cũ sống lâu hơn, còn 30 giây giảm stale reuse nhưng làm giảm hit/cache availability. Webhook chỉ invalidate nội dung đã thông báo; silent update không phát webhook nên TTL là giới hạn duy nhất.

**Stale-if-error:** Với `max_stale_s=0`, stale answer không được dùng trong outage; hiện 152 stale response đã cứu FAQ. Câu hỏi cá nhân không được cache vì dữ liệu thay đổi và có nguy cơ rò rỉ giữa user; phân vùng theo user vẫn có thể giữ trạng thái tài chính/đơn hàng lỗi thời. Khi cache đang phục vụ tốt nhưng provider sập, alert theo provider error rate/health, breaker state, và tỷ lệ stale-served/cache-served song song với user availability để dashboard không che mất outage.
