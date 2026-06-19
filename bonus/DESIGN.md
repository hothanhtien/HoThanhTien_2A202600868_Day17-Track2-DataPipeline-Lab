# Thiết Kế Pipeline: Hệ Thống Phát Hiện Gian Lận Thời Gian Thực cho Ví Điện Tử

## Bài Toán Thực Tế

Một ví điện tử phổ biến tại Việt Nam (như MoMo, ZaloPay) xử lý hàng triệu giao dịch mỗi ngày. Tỷ lệ gian lận nhỏ (~0.1%) nhưng thiệt hại lớn vì giá trị giao dịch cao và tốc độ thanh toán cực nhanh (< 1 giây). Bài toán: **phát hiện giao dịch gian lận trong vòng < 200ms** trước khi tiền được chuyển, đồng thời không chặn oan giao dịch hợp lệ (tỷ lệ false positive < 0.5%).

**Ràng buộc cứng:**
- Độ trễ quyết định: < 200ms end-to-end
- Throughput: 50,000 giao dịch/giây vào giờ cao điểm Tết
- Dữ liệu nhạy cảm: phải tuân thủ Nghị định 13/2023/NĐ-CP về bảo vệ dữ liệu cá nhân
- Hạ tầng: on-premise hoặc Việt Nam-region cloud (Viettel IDC, VNG Cloud) vì yêu cầu lưu trữ dữ liệu trong nước

---

## Kiến Trúc Đề Xuất

```
[App Mobile/Web]
      |
      v
[Kafka / Redpanda]  <-- producer: mỗi giao dịch là 1 event
      |
      +--> [Flink Streaming Job] -----> [Redis Feature Store]
      |         |                             |
      |    (window agg                  (lookup real-time
      |     last 5/30 min)               features per user)
      |         |
      |         v
      |    [Fraud Scoring Service] <-- model phục vụ (ONNX runtime)
      |         |
      |    PASS / HOLD / BLOCK
      |         |
      v         v
[Bronze - raw events]   [Alert Queue -> Ops Team]
      |
      v
[Silver - enriched + labeled (sau review)]
      |
      v
[Gold - daily fraud summary + model retraining trigger]
      |
      v
[MLflow + DPO flywheel] --> retrain model hàng tuần
```

---

## Các Quyết Định Thiết Kế và Đánh Đổi

### 1. Kafka vs RabbitMQ cho event streaming

**Chọn Kafka (Redpanda on-premise).**

- **Kafka:** throughput cao (50K msg/s dễ dàng), replay được event để debug và retrain, partition-by-user-id đảm bảo thứ tự per-user.
- **RabbitMQ:** dễ cài hơn, nhưng không có native replay, throughput thấp hơn ở scale lớn.

Replay là bắt buộc ở đây: khi model mới được deploy, cần backfill score lại 30 ngày qua để đánh giá hiệu năng. RabbitMQ không làm được điều này.

### 2. Feature tính real-time vs batch

**Chọn hybrid:** features nhanh (count giao dịch 5 phút qua, velocity) tính streaming qua Flink; features chậm (lịch sử 90 ngày, hành vi theo mùa) tính batch hàng đêm, lưu vào Redis với TTL 24 giờ.

- **Chỉ dùng streaming:** đơn giản hơn nhưng cửa sổ thời gian ngắn, bỏ sót pattern dài hạn.
- **Chỉ dùng batch:** bỏ sót gian lận burst trong vài phút (kẻ tấn công chuyển tiền nhanh trước khi batch chạy).

Point-in-time correctness (**bài học Lab 17**) áp dụng ở đây: khi retraining, phải lấy feature value *tại thời điểm giao dịch xảy ra*, không phải giá trị hiện tại — tránh data leakage vào tập train.

### 3. Model serving: real-time API vs embedded scoring

**Chọn ONNX runtime embedded trong Flink job.**

- **Tách biệt scoring service:** linh hoạt scale độc lập, nhưng thêm 1 network hop ~20-50ms — vi phạm SLA 200ms khi cộng với latency DB và Kafka.
- **ONNX embedded:** model chạy trong cùng JVM với Flink, zero network, latency < 5ms cho inference. Trade-off: deploy model mới phải restart Flink job (downtime ~10s, chấp nhận được với blue-green).

### 4. Quyết định BLOCK vs HOLD

**Chọn 3 cấp: PASS / HOLD / BLOCK thay vì binary.**

BLOCK chỉ khi score > 0.95 (ngưỡng rất cao). HOLD (tạm giữ 60 giây, yêu cầu OTP xác nhận) khi 0.7 < score < 0.95. PASS khi score < 0.7.

Lý do: giao dịch bị chặn oan tại Việt Nam gây mất niềm tin nghiêm trọng hơn gian lận (người dùng chuyển khoản trong đêm trả tiền nhà, bị block là khủng hoảng). HOLD cho phép người dùng xác nhận mà không mất tiền.

### 5. Phương án bị loại: Rule-based only

Nhiều hệ thống ví điện tử Việt Nam hiện tại dùng rule cứng (chuyển > 50 triệu VNĐ phải OTP, v.v.). Rule-based không bắt được pattern mới của kẻ tấn công (chia nhỏ giao dịch, mạo danh hành vi hợp lệ). ML + flywheel cho phép model học pattern mới từ feedback của fraud team hàng tuần mà không cần code thêm rule.

---

## Flywheel Dữ Liệu (Kết Nối Với Lab 17)

Mỗi giao dịch được review bởi fraud team → label `fraud` / `legitimate` → vào Silver layer → tạo DPO-style preference pairs (giao dịch hợp lệ là "chosen", giao dịch gian lận cùng pattern là "rejected") → retrain model mỗi thứ Hai → model mới được A/B test 7 ngày trước khi full rollout.

Decontamination: cases đang trong quá trình điều tra pháp lý **không được dùng để train** cho đến khi có kết luận chính thức — tránh model học pattern chưa được xác nhận.

---

## Rủi Ro Triển Khai Tại Việt Nam

- **Dữ liệu phải ở trong nước:** toàn bộ pipeline chạy trên VNG Cloud hoặc Viettel IDC, không dùng AWS/GCP/Azure trực tiếp vì dữ liệu tài chính cá nhân.
- **Múi giờ GMT+7:** batch job chạy lúc 2AM-4AM (ít giao dịch nhất), không phải midnight UTC — nhiều pipeline quốc tế cài sai múi giờ và overlap với giờ cao điểm sáng.
- **Tết Nguyên Đán:** throughput tăng 10x trong 3 ngày — cần pre-scale Kafka và Flink cluster trước 1 tuần, không thể dùng auto-scaling vì latency spin-up > 5 phút.
