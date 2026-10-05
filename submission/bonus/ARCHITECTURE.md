# Architecture Brief: High-Throughput LLM Observability Lakehouse (1B Requests/Day)

**Học viên thực hiện:** Nguyễn Quốc Tuấn (MSSV: 2A202602910)  
**Topic:** A — LLM Observability ở quy mô 1 tỷ requests/ngày  
**Mã môn học:** K4-Track02-Day18 Lakehouse Lab

---

## 1. Executive Summary & Constraints

### 1.1 Bối cảnh & Yêu cầu bài toán
Hệ thống Gateway API cho Foundation Model phục vụ **1.000.000.000 requests/ngày** (~11.574 req/s trung bình, peak ~25.000 req/s). Mỗi payload request/response trung bình ~5 KB, tương đương:
- **Tốc độ sinh dữ liệu thô:** $5\text{ TB/ngày}$ ($150\text{ TB/tháng}$).
- **Yêu cầu 1 (Dashboard):** SLA dashboard hiển thị cost & latency theo `tenant_id`, refresh mỗi **5 phút** ($p95 < 1\text{s}$).
- **Yêu cầu 2 (Retention):** Giữ full prompt/response trong **7 ngày** phục vụ incident debugging; sau 7 ngày chỉ lưu dữ liệu tổng hợp (aggregates) trong **1 năm**.
- **Yêu cầu 3 (Privacy):** Khử định danh / mã hóa PII (Personally Identifiable Information) ngay tại tầng ingest trước khi bất kỳ ai đọc.
- **Yêu cầu 4 (Ngân sách FinOps):** Tổng chi phí lưu trữ storage $\le \mathbf{\$5.000\text{/tháng}}$.

---

## 2. Kiến trúc tổng thể (Architecture Diagram & Medallion Layout)

```
[Clients / API Gateways] (~25K req/s peak)
         │
         ▼
[Kafka / Redpanda Ingest Buffer] (Partitioned by hash(tenant_id), 3-day retention)
         │
         ▼
[Streaming Flink / Spark Ingestion Workers]
         ├─── Step 1: PII Tokenization & Presidio Redaction (Salted Hash for IDs)
         └─── Step 2: Micro-batch append (1-min batches)
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       MEDALLION STORAGE LAYER (S3 / Delta)                  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ BRONZE LAYER (Raw Sanitized Event Stream)                             │  │
│  │ - Format: Delta Lake (Parquet + Snappy)                               │  │
│  │ - Partition: day(timestamp)                                           │  │
│  │ - Clustered / Z-ORDER: tenant_id, request_id                          │  │
│  │ - Lifecycle: S3 Standard (0-7 days) -> S3 Glacier Instant (8-30 days) │  │
│  │              -> S3 Lifecycle Hard Delete after 30 days                │  │
│  │ - Volume: ~35 TB (giữ 7 ngày active)                                  │  │
│  └───────────────────────────────────┬───────────────────────────────────┘  │
│                                      │                                      │
│               Continuous 5-minute Micro-batch (Streaming)                   │
│                                      ▼                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ SILVER LAYER (Normalized & Deduplicated Telemetry)                    │  │
│  │ - Fields: request_id, tenant_id, model, tokens_in, tokens_out,        │  │
│  │           latency_ms, cost_usd, status_code, timestamp                │  │
│  │   (Prompt/Response payload moved to external Cold Blob via pointer)   │  │
│  │ - Partition: day(timestamp)                                           │  │
│  │ - Z-ORDER BY: tenant_id, timestamp                                    │  │
│  │ - Volume: 150 GB/ngày (~4.5 TB/tháng, retention 90 ngày)             │  │
│  └───────────────────────────────────┬───────────────────────────────────┘  │
│                                      │                                      │
│               Scheduled Aggregation Rollup (Hourly / Daily)                 │
│                                      ▼                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ GOLD LAYER (Aggregated Metrics & FinOps Datamart)                     │  │
│  │ - Granularity: 5-minute rollup per tenant_id, model                   │  │
│  │ - Metrics: request_count, p50_latency, p95_latency, p99_latency,     │  │
│  │            total_tokens, total_cost_usd, error_rate                   │  │
│  │ - Volume: ~1.5 GB/tháng (Lưu 1 năm = 18 GB)                           │  │
│  │ - Query Engine: Trino / DuckDB phục vụ Grafana & Tenant Dashboard     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Năm quyết định kiến trúc cốt lõi & Đánh giá phương án (Key Decisions & Alternatives)

### Quyết định 1: Định dạng bảng lưu trữ (Table Format) — Chọn Delta Lake
- **Lựa chọn:** Delta Lake (delta-rs / Delta Spark).
- **Phương án thay thế 1 (Apache Iceberg):** Iceberg rất mạnh về catalog độc lập và hidden partitioning, nhưng tại thời điểm thiết kế, hệ sinh thái streaming write micro-batch tốc độ cao (sub-minute commits với append latency nhỏ) của Delta Lake chín muồi hơn, hỗ trợ Z-order đa chiều trực tiếp trong write job.
- **Phương án thay thế 2 (Apache Hudi):** Hudi hỗ trợ Merge-On-Read (MOR) tốt cho streaming, nhưng độ phức tạp vận hành cao và metadata footprint lớn hơn Delta khi số lượng data file tăng cao.
- **Đánh đổi (Trade-off):** Chấp nhận cơ chế log compaction định kỳ của Delta để đổi lấy tốc độ streaming append tối ưu và tính tương thích cao với DuckDB/Trino.

### Quyết định 2: Tách biệt Payload thô và Metadata (Inline Blobs vs. Pointer Architecture)
- **Lựa chọn:** Pointer Architecture cho Silver. Prompt và completion lớn (>2 KB) được lưu nén riêng biệt tại Bronze, Silver chỉ chứa các metadata số (token count, latency, cost) kèm con trỏ URL/UUID tới Bronze.
- **Phương án thay thế 1 (Inline Blobs trong toàn bộ pipeline):** Lưu toàn bộ prompt/response trong Silver. Bị loại vì dẫn đến hiện tượng **Random-access I/O amplification** ($\ge 5\times$) khi dashboard chỉ đọc `tenant_id`, `latency` và `cost`.
- **Phương án thay thế 2 (Lưu prompt trong Elasticsearch/OpenSearch):** Chi phí cluster search cho 5 TB/ngày quá đắt (vượt trần $5.000/tháng).
- **Đánh đổi:** Tăng thêm 1 bước join khi cần review incident sâu, nhưng giảm $90\%$ dung lượng query cho $99\%$ các dashboard thông thường.

### Quyết định 3: Khử trùng lặp và Bảo vệ dữ liệu cá nhân (PII Tokenization at Ingestion)
- **Lựa chọn:** Khử định danh PII ngay tại tầng Streaming Consumer trước khi ghi vào Bronze, sử dụng Salted Deterministic Hash (HMAC-SHA256 với secret quay vòng theo tháng) cho user IDs và regex/NER masking cho email/SĐT/thẻ tín dụng.
- **Phương án thay thế 1 (Redact định kỳ bằng batch job trên Bronze):** Bị loại vì tạo ra cửa sổ vi phạm tuân thủ (compliance window) kéo dài hàng giờ khi dữ liệu thô chưa redact vẫn nằm trên storage.
- **Phương án thay thế 2 (Client tự redact ở SDK):** Không kiểm soát được nếu client gửi trực tiếp hoặc bypass SDK.
- **Đánh đổi:** Tăng CPU tiêu thụ của worker streaming lên ~12%, nhưng đảm bảo dữ liệu ghi xuống Bronze hoàn toàn "clean-by-default".

### Quyết định 4: Chiến lược gom cụm dữ liệu (Clustering / Z-ORDER)
- **Lựa chọn:** Partition theo `day(timestamp)`, trong từng partition áp dụng **Z-ORDER BY `(tenant_id, timestamp)`**.
- **Phương án thay thế 1 (Partition trực tiếp theo `tenant_id`):** Gây ra **High-cardinality Partitioning Anti-Pattern** (với 10.000 tenants, hệ số phân tán file tăng vọt, metadata quá tải).
- **Phương án thay thế 2 (Chỉ sort theo timestamp):** Bị loại vì truy vấn lọc theo từng tenant sẽ phải quét toàn bộ partition ngày ($0\%$ file skipping).
- **Đánh đổi:** Cần định kỳ chạy compaction & Z-order job mỗi 6 giờ, nhưng cho phép query của tenant skip được $\ge 85\%$ số file Parquet.

### Quyết định 5: Chiến lược Tiering & Retention FinOps
- **Lựa chọn:**
  - Bronze: Lưu tại S3 Standard trong 7 ngày, sau đó S3 Lifecycle tự động chuyển sang S3 Glacier Instant Retrieval cho ngày 8–30, và Hard Delete sau ngày thứ 30.
  - Silver: Giữ 90 ngày.
  - Gold: Giữ 365 ngày (1 năm) trên S3 Standard.
- **Phương án thay thế 1 (Giữ toàn bộ Bronze 1 năm trên S3 Standard):** Chi phí lưu trữ 1.800 TB sẽ lên tới $\approx \$41.400\text{/tháng}$ (phá vỡ hoàn toàn ngân sách \$5.000).
- **Phương án thay thế 2 (Xóa Bronze ngay sau khi tổng hợp Silver):** Bị loại vì không đáp ứng yêu cầu incident review 7 ngày của khách hàng enterprise.

---

## 4. Phép tính Dung lượng & Ngân sách FinOps chi tiết

### 4.1 Bảng tính dung lượng lưu trữ (Storage Sizing)
- **Dữ liệu thô:** $1\times 10^9 \text{ req/ngày} \times 5\text{ KB/req} = 5.000\text{ GB} = 5\text{ TB/ngày}$.
- **Hệ số nén Parquet + Snappy:** Giảm dung lượng còn $\approx 35\%$ kích thước ban đầu $\rightarrow 1.75\text{ TB/ngày}$.

| Tầng dữ liệu | Thời gian giữ (Retention) | Dung lượng trên đĩa | Storage Class | Đơn giá / GB / tháng | Chi phí ước tính / tháng |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Bronze (Hot)** | 7 ngày | $7 \times 1.75\text{ TB} = 12.25\text{ TB}$ | S3 Standard | \$0.023 | \$281.75 |
| **Bronze (Warm)** | 23 ngày kế tiếp | $23 \times 1.75\text{ TB} = 40.25\text{ TB}$ | S3 Glacier Instant | \$0.004 | \$161.00 |
| **Silver (Metadata)** | 90 ngày | $90 \times 60\text{ GB} = 5.4\text{ TB}$ | S3 Standard | \$0.023 | \$124.20 |
| **Gold (Aggregates)** | 365 ngày | $18\text{ GB}$ (1 năm) | S3 Standard | \$0.023 | \$0.41 |
| **Delta Transaction Logs**| Checkpoint 10 versions | ~150 GB | S3 Standard | \$0.023 | \$3.45 |
| **Dự phòng I/O Requests** | PUT/GET requests & Lifecycle | — | — | — | ~\$350.00 |
| **TỔNG CHI PHÍ LƯU TRỮ**| | **~58 TB tổng cộng** | | | **~\$920.81 / tháng** |

> **Kết luận FinOps:** Tổng chi phí lưu trữ là **~\$921/tháng**, thấp hơn rất nhiều so với trần ngân sách **\$5.000/tháng**. Phần ngân sách còn lại (~$4.000) được dành cho cụm compute stream-worker (Flink/Kinesis) và compute phục vụ query.

---

## 5. Failure Modes, Giám sát & Kế hoạch Khôi phục (Failure Modes & Recovery)

| Failure Mode | Cơ chế phát hiện (Detection) | Tác động hệ thống | Kế hoạch khắc phục & Rollback |
| :--- | :--- | :--- | :--- |
| **1. Small Files Explosion do Peak Traffic** | CloudWatch Metric: `num_files_per_day > 50,000` hoặc query latency tăng đột biến. | Metadata Delta log phình to, latency dashboard suy giảm từ 1s lên 15s. | Kích hoạt Auto-Compaction job khẩn cấp: gọi `dt.optimize.compact(target_size=128MB)` gom nhóm các file nhỏ, sau đó chạy `Z-ORDER(tenant_id)`. |
| **2. PII Tokenizer Salt Leak hoặc Lỗi Regex** | Scanner kiểm thử canary quét thấy pattern email/SĐT chưa che trong Silver. | Rủi ro rò rỉ dữ liệu cá nhân vi phạm tuân thủ bảo mật. | 1. Dừng streaming ingest.<br>2. Xoay vòng Salt key.<br>3. Chạy `dt.restore(target_version)` lùi về commit trước khi lỗi xảy ra.<br>4. Re-play dữ liệu từ Kafka buffer với tokenizer bản vá. |
| **3. Uncommitted Orphan Files do Worker Crash** | Giám sát kích thước bucket S3 so với tổng dung lượng tính từ active Delta commits (độ lệch > 5%). | Lãng phí dung lượng lưu trữ rác trên S3, chi phí âm thầm tăng cao. | Chạy định kỳ **Job 4 Orphan Cleanup** (dựa trên thuật toán so sánh hiệu tập hợp giữa danh sách file trên S3 và `dt.file_uris()`), loại bỏ các file uncommitted sau thời gian an toàn 24 giờ. |

---

## 6. Kế hoạch triển khai MVP trong 1 tuần (One-Week MVP Plan)

- **Ngày 1–2 (Ingestion Slice):**
  - Dựng Kafka topic mẫu với 10 triệu requests/ngày (1% scale).
  - Triển khai pipeline streaming write vào Delta Lake với PII Masking UDF.
- **Ngày 3–4 (Medallion & Z-Order Tuning):**
  - Cấu hình Silver deduplication và Gold 5-minute rollup.
  - Đo đạc file-pruning ratio khi query lọc theo `tenant_id` trước và sau Z-order (Tiêu chí: pruning ratio $\ge 10\times$).
- **Ngày 5 (Maintenance & Verification):**
  - Thiết lập script compaction, vacuum và orphan cleanup tự động.
  - Xác thực khả năng phục hồi RESTORE khi inject batch lỗi.
- **Tiêu chí nghiệm thu (Acceptance Criteria):**
  1. Dashboard lọc theo tenant phản hồi $p95 < 800\text{ ms}$.
  2. Không có bất kỳ plain-text PII nào lọt vào Bronze/Silver.
  3. Chi phí storage ngoại suy ở 1B req/ngày không vượt quá \$1.200/tháng.
