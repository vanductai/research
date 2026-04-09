Tôi đã đọc xong toàn bộ ~5500 dòng. Dưới đây là phân tích chi tiết.

---

# Phân Tích Kiến Trúc AI Agent BI System

## Tóm tắt nhanh

Tài liệu có chất lượng cao — nghiên cứu thị trường kỹ, quyết định kiến trúc có lý do rõ ràng. Tuy nhiên phát hiện **10 điểm mâu thuẫn** (3 nghiêm trọng) và **15 điểm có thể cải thiện**.

---

## I. Các Điểm Mâu Thuẫn (Contradictions)

### 🔴 Mâu thuẫn nghiêm trọng

---

#### M1. `mv_daily_store` dùng cột không tồn tại (§5.5 vs §5.2)

**Vấn đề:** Fact table `fct_sales_daily` chỉ có cột `customer_hll HLL` (HyperLogLog), không có `customer_count`. Nhưng MV `mv_daily_store` viết:
```sql
-- ❌ SAI — cột này không tồn tại trong fct_sales_daily
SUM(f.customer_count) AS customer_count
```
Trong khi `mv_monthly_region` lại dùng đúng:
```sql
-- ✅ ĐÚNG
HLL_UNION_AGG(f.customer_hll) AS customer_hll
```

**Hậu quả:** `mv_daily_store` sẽ fail khi deploy. Đây chính xác là loại lỗi "silent data corruption" mà §5.1.7 cảnh báo.

**Fix:** Đổi MV `mv_daily_store` thành:
```sql
HLL_UNION_AGG(f.customer_hll) AS customer_hll
```

---

#### M2. Embedding model không nhất quán với multilingual requirement (§2.3 vs §5.6.2.3)

**Vấn đề:** §2.3 (Multi-Language Strategy) khuyến nghị rõ ràng:
> `multilingual-e5-large` hoặc `paraphrase-multilingual-MiniLM-L12-v2` cho semantic search

Nhưng code implementation `QueryFingerprint` dùng:
```python
self.encoder = SentenceTransformer('all-MiniLM-L6-v2')  # English-only!
```

**Hậu quả:** Cache semantic matching sẽ hoạt động kém khi user hỏi tiếng Việt. "Doanh thu Q1 2026" và "Q1 2026 revenue" sẽ không match đúng.

---

#### M3. String interpolation trong RBAC SQL — SQL Injection risk (§7.3)

**Vấn đề:** `ChatRBACMiddleware.enforce()` inject region values bằng string interpolation:
```python
allowed = ', '.join(f"'{r}'" for r in user.allowed_regions)
rbac_filters.append(f"s.region IN ({allowed})")
```
Nếu region name chứa ký tự đơn quote (`'`) — ví dụ: `Côn Đảo's Store` — sẽ bị SQL injection. Permissions đến từ database nên rủi ro thấp nhưng không phải zero (compromised DB, stored XSS, v.v.)

**Fix:** Dùng parameterized approach: build `WHERE s.region = ANY($1::text[])` với array param.

---

### 🟡 Mâu thuẫn trung bình

---

#### M4. Tiêu đề "5 Lớp Bảo Vệ" nhưng có 6 lớp (§5.6.1.3)

Tiêu đề: `"5.6.1.3. Enforcement Mechanisms — 5 Lớp Bảo Vệ"` nhưng bảng liệt kê L1–L6 (6 layers). L6 (Retry Budget) được thêm nhưng tiêu đề chưa cập nhật.

---

#### M5. Query Proxy: PostgreSQL-style params dùng với StarRocks MySQL protocol (§6.2)

Code `buildRevenueTrendQuery()` dùng `$1, $2...` (PostgreSQL positional params):
```typescript
WHERE d.date_key BETWEEN $${paramIndex++} AND $${paramIndex++}
```
StarRocks kết nối qua **MySQL protocol** → dùng `?` placeholders. Code sẽ không execute được trên StarRocks.

---

#### M6. Cache hit rate target không nhất quán giữa các sections

| Section | Target |
|:--------|:-------|
| §5.6.4 KPIs | > 40% (tuần 1) → > 60% (tháng 3) |
| §6.6 Filter Performance | > 70% |
| §7.6 Security KPIs | > 70% |

Không rõ đây là cùng 1 chỉ số hay đo hai loại cache khác nhau (Redis dashboard cache vs. Result Artifact cache). Cần phân biệt rõ.

---

#### M7. Phase 3 Roadmap trùng lặp với Phase 0 (§8)

| Phase 0 | Phase 3 |
|:--------|:--------|
| "Configure CDC: MongoDB + MySQL Binlog" | "Database connectors — MongoDB, MySQL via Airbyte CDC" |

Đây là duplicate task. Phase 3 nên tập trung vào semantic layer, không phải setup CDC đã làm từ Phase 0.

---

#### M8. Deployment table mô tả sai vai trò FastAPI (§9.6 vs §4.2.1)

§9.6 ghi: `FastAPI Backend → "Agent orchestration + Query Proxy"`  
Nhưng §4.2.1 định nghĩa rõ: **Query Proxy thuộc Next.js API Routes**, FastAPI chỉ có Agent orchestration.

---

### 🟢 Mâu thuẫn nhỏ

#### M9. Section number 5.7 nhưng subsections đánh số 5.6.x

§5.7 header là "Data Integrity & Result Reusability Architecture" nhưng subsections dùng `5.6.1`, `5.6.2`, `5.6.3`, `5.6.4` — toàn bộ section này thực ra là §5.6 bị đánh lại header thành §5.7.

---

#### M10. Code syntax error trong WrenAIClient (§4.4.3)

```python
response = await httpx.post(f"{self.base_url}/v1/ask", json={
    "question": question,
    "# Wren AI tự động dùng MDL..."  # ← string comment làm dict key
})
```
Comment bị đặt làm key trong JSON dictionary — code không hợp lệ về mặt logic (sẽ gửi key lạ tới API).

---

## II. Các Điểm Có Thể Cải Thiện

### 🏗️ Kiến trúc & Thiết kế

---

#### I1. Thiếu pre-warming strategy cho K8s Pod Sandbox (§2.2, §4.3)

Khi nhiều users đồng thời yêu cầu Python analysis → pod cold start mỗi lần (1–3s scheduling + container start). Với 300–500 users, spike traffic có thể gây latency > 30s.

**Đề xuất:** Thêm **pod pool pre-warming** — giữ sẵn N idle pods (ví dụ: 3–5 pods), scale theo demand. Tương tự AWS Lambda warm start strategy:
```yaml
# K8s: Minimum replica pods (warm)
minReplicas: 3
maxReplicas: 20
```

---

#### I2. Thiếu strategy xử lý schema drift từ production MongoDB (§5.1)

Pipeline: `MongoDB → Airbyte → dbt → StarRocks → Wren AI MDL`. Nếu production MongoDB thêm/đổi field → dbt raw models lệch schema → dbt compile fail → toàn pipeline stop.

**Đề xuất:** Thêm step **Schema Drift Detection** trong Dagster:
1. Airbyte detect schema change → emit event
2. Dagster pause pipeline → alert data team
3. Sau khi dbt model và MDL được update → resume

---

#### I3. Không rõ lifecycle của ChromaDB → Qdrant migration (§4.2)

Document nói "ChromaDB (dev) → Qdrant (prod)" nhưng không define:
- Threshold nào trigger switch (data size? query volume? ChromaDB bottleneck metrics?)
- Migration steps cho existing embeddings
- Rollback plan nếu Qdrant có issue

**Đề xuất:** Define migration trigger (ví dụ: > 500K vectors hoặc p95 search > 200ms) và viết migration runbook.

---

#### I4. Circular dependency giữa Next.js và FastAPI (§4.2.1)

- Next.js forward chat requests → FastAPI (`/api/chat/*`)
- FastAPI Agent gọi Next.js Query Proxy để reuse RBAC + cache

Nếu cả 2 service down đồng thời, hoặc Next.js slow → FastAPI timeout → Next.js nhận error → retry → loop. Không có mô tả circuit breaker cho inter-service communication này.

**Đề xuất:** Thêm circuit breaker (ví dụ: `httpx` retry config + timeout) và fallback: nếu Next.js Query Proxy unreachable, FastAPI execute query trực tiếp với RBAC tự handle (không gọi Next.js).

---

#### I5. Race condition trong cache invalidation + pre-warming (§5.6.2.4)

```python
# Step 1: Mark stale
UPDATE result_artifacts SET is_stale = TRUE WHERE ...

# Step 2: Background refresh popular
await self.background_refresh(artifact)
```

Giữa Step 1 và Step 2: một request đến trong window này sẽ nhận "stale" artifact và serve stale result. Với 300–500 concurrent users, window này có thể gây nhiều stale serves.

**Đề xuất:** Dùng Redis distributed lock (SETNX) để đảm bảo:
1. Đánh dấu stale + launch background refresh là atomic
2. Subsequent requests block trên lock hoặc serve stale với explicit "refreshing" badge

---

### 🔐 Bảo mật

---

#### I6. Thiếu rate limiting cho AI Chat Path (§7)

Section 7 mô tả security cho Dashboard nhưng AI Chat (UC2) không có rate limiting rõ ràng. Một user có thể gửi 1000 queries → exhaust LLM budget + K8s Pod Sandbox.

§9.5 đề cập "Rate limiting & quotas" nhưng chỉ là Phase 4 task — quá trễ.

**Đề xuất:** Implement rate limiting từ Phase 1:
- Per-user: tối đa 60 queries/giờ (configurable)
- Per-session: tối đa 20 complex Python queries/giờ
- Global: budget alert khi LLM cost vượt threshold/ngày

---

#### I7. RBAC trong ChatRBACMiddleware giả định query luôn JOIN dim_store (§7.3)

```python
if 'fct_sales_daily' in tables or 'fct_revenue' in tables:
    if user.role == 'regional_manager':
        rbac_filters.append(f"s.region IN ({allowed})")  # assume JOIN dim_store AS s
```

Nếu LLM-generated SQL không alias `dim_store` as `s`, hoặc query không JOIN `dim_store` (ví dụ query chỉ trên `fct_monthly_summary`), RBAC filter `s.region IN (...)` sẽ fail với "column s.region not found".

**Đề xuất:** Parse SQL thực sự (dùng `sqlglot` hoặc `sqlparse`) thay vì string matching để tìm đúng alias và inject filter chính xác.

---

### 📊 Data Platform

---

#### I8. MDL governance thiếu ownership rõ ràng (§4.4.4)

| Trigger | Action | Automation Level |
|:--------|:-------|:----------------|
| Business metric thay đổi | Cập nhật MDL | 🔴 Manual |

Nhưng "manual" bởi ai? Data team? Product team? Nếu "Doanh thu" định nghĩa thay đổi (gross → net) mà không có approval process → inconsistent KPIs cho toàn tổ chức.

**Đề xuất:** Define MDL Change Control Process:
1. RFC (Request for Change) → PR trên Git
2. Review bởi: Data Team Lead + Finance/domain owner
3. Staging deploy → test NL queries accuracy
4. Production deploy + notify tất cả dashboard owners

---

#### I9. User expectation gap về data freshness (§5.1.4 vs §5.8)

Freshness SLA: Orders ≤ 15 phút. Nhưng trong §5.8, timeline end-to-end là 8–10 phút từ khi order được tạo đến khi data sẵn sàng.

Nếu user thấy badge "Data cập nhật 3 phút trước" và hỏi AI về đơn hàng vừa tạo 4 phút trước → AI trả lời "không có đơn hàng nào" → user confused.

**Đề xuất:** 
- Hiển thị rõ hơn: "Data có hiệu lực đến [timestamp]. Đơn hàng tạo sau [timestamp] chưa được phản ánh."
- Khi user hỏi về data "vừa xong" (trong 15 phút), AI cần proactively cảnh báo về freshness window.

---

### 🗺️ Implementation Roadmap

---

#### I10. RBAC/Auth được schedule quá muộn (Phase 4) (§8)

Phases 1–3 đã có: Chat interface, Python sandbox, data access, dashboard builder — tất cả đều cần auth. Không có auth trong Phase 1–3 nghĩa là:
- Development không test realistic security flow
- Risk phát hiện auth integration breaking changes muộn

**Đề xuất:** Move "Setup Better Auth với JWT" sang Phase 1 (cùng với foundation setup). Có thể dùng simple API key cho internal dev đến khi full Better Auth setup xong, nhưng RBAC framework phải có từ sớm.

---

#### I11. File upload pipeline không được define rõ (§8.2 vs §4.2)

Phase 1 task: "File upload (CSV/Excel)" nhưng không có:
- Next.js hay FastAPI xử lý upload?
- Schema inference pipeline (pandas? dbt?)
- Làm thế nào file data integrate vào Wren AI semantic layer?
- File cleanup policy (sau 24h? user-controlled?)

**Đề xuất:** Define file upload pipeline rõ ràng: `Browser → Next.js → MinIO → FastAPI (schema inference) → Temporary StarRocks table → Wren AI MDL patch (session-scoped)`.

---

### ⚙️ Operations

---

#### I12. StarRocks backup strategy thiếu (§9.8)

| StarRocks | Không backup riêng — re-sync từ source via Airbyte |

Với 2TB+ data, re-sync từ MongoDB có thể mất **2–4 giờ**. Trong thời gian này, AI Agent BI hoàn toàn không hoạt động.

**Đề xuất:** Thêm StarRocks snapshot backup:
- StarRocks hỗ trợ `BACKUP ... TO REPOSITORY` native
- Snapshot daily → MinIO → incremental restore < 30 phút
- Giảm RTO từ 2–4 giờ → < 1 giờ

---

#### I13. Dashboard/Report schema migration strategy không có (§6.0.4)

Dashboards được lưu dưới dạng JSONB với `queryConfig.queryTemplate` chứa SQL template và column references. Nếu dbt model rename column (`net_revenue` → `revenue_net`), tất cả saved dashboards sẽ query fail silently.

**Đề xuất:**
1. Thêm versioning vào dashboard schema: `dashboardSchemaVersion: "1.0"`
2. Khi dbt column rename → Dagster trigger dashboard migration job
3. Alert nếu migration fail → dashboard owner được notify

---

#### I14. Session persistence cho LangGraph Agent không rõ (§4.3)

User đang chat multi-turn, đóng tab, mở lại → session có được restore không? LangGraph hỗ trợ checkpointing nhưng không có mô tả:
- Session TTL bao lâu?
- Lưu checkpoint ở đâu (PostgreSQL? Redis?)
- Cross-device session sharing?

**Đề xuất:** Define session persistence strategy:
- Lưu LangGraph checkpoints vào PostgreSQL `chat_sessions`
- TTL: 7 ngày (configurable per user)
- Session resume: user open chat → fetch last checkpoint → context restored

---

#### I15. PostgreSQL resource spec không nhất quán (§9.3.2 vs §9.6)

| Table | RAM spec |
|:------|:---------|
| §9.3.2 K8s Resource Allocation | 8GB RAM |
| §9.6 Deployment Architecture | 4GB RAM |

Cần thống nhất. PostgreSQL lưu audit logs (high write volume) + metadata + sessions → 4GB có thể tight khi scale.

---

## III. Tổng Hợp Ưu Tiên

| Mức độ | Vấn đề | Section |
|:-------|:-------|:--------|
| 🔴 **Blocker** | MV dùng cột không tồn tại (`customer_count`) | §5.5 |
| 🔴 **Blocker** | StarRocks param style `$1,$2` sai (nên dùng `?`) | §6.2 |
| 🔴 **Security** | SQL injection trong RBAC filter injection | §7.3 |
| 🟠 **High** | Embedding model tiếng Anh cho dữ liệu tiếng Việt | §5.6.2.3 |
| 🟠 **High** | Thiếu rate limiting cho AI Chat | §7 |
| 🟠 **High** | RBAC assume alias `s` luôn tồn tại trong SQL | §7.3 |
| 🟡 **Medium** | Auth Phase 4 — quá muộn | §8 |
| 🟡 **Medium** | StarRocks không có backup native | §9.8 |
| 🟡 **Medium** | K8s Pod pool pre-warming | §2.2 |
| 🟡 **Medium** | Session persistence LangGraph chưa define | §4.3 |
| 🟢 **Low** | Numbering section 5.7/5.6.x | §5.7 |
| 🟢 **Low** | Code comment làm dict key trong WrenAIClient | §4.4.3 |
| 🟢 **Low** | PostgreSQL RAM spec mâu thuẫn 8GB vs 4GB | §9.3.2/§9.6 |

---

## IV. Các Điểm Bổ Sung (Phát hiện thêm bởi Antigravity)

Ngoài các phân tích ở trên, hệ thống còn thiếu một số tiểu tiết kiến trúc rất quan trọng có thể gây lỗi nghiêm trọng khi triển khai thực tế. Dưới đây là 4 điểm cần đặc biệt chú ý:

### 🔴 Lỗi Logic Nghiêm Trọng

#### A1. Lỗ hổng Entity Guard trong Semantic Cache gây sai lệch dữ liệu (§5.6.2.3)

**Vấn đề:** Trong `QueryFingerprint.find_matching_artifact()`, đoạn code kiểm tra Entity Guard hiện chỉ so sánh `metric` và `time_range`:
```python
if (cached_entities.metric == entities.metric and
    cached_entities.time_range == entities.time_range):
    return CacheResult(...)
```
Entity Guard này **bỏ quên việc kiểm tra `filters` và `granularity`**. 
**Hậu quả:** Kéo theo thảm họa về data integrity. Nếu User A hỏi *"Doanh thu Q1 tại Miền Bắc"*, và User B hỏi *"Doanh thu Q1 tại Miền Nam"*, Semantic Matcher sẽ coi chúng là giống hệt nhau (do cùng metric "Doanh thu" và time "Q1") và cho ra cùng một kết quả truy vấn.
**Fix:** Bắt buộc bổ sung phép so sánh đối với `filters` và `granularity`:
```python
if (cached_entities.metric == entities.metric and
    cached_entities.time_range == entities.time_range and
    cached_entities.filters == entities.filters and
    cached_entities.granularity == entities.granularity):
```

### 🟠 Cải Tiến Hệ Thống & Bảo Mật

#### A2. Lỗ hổng Sandbox Timeout (Infinite Loop) trong K8s execution (§5.6)

**Vấn đề:** Khi LLM sinh ra Python code chạy trong K8s Pod Sandbox, hệ thống chưa đề cập tới việc kiểm soát giới hạn thời gian chạy (Runtime Hard Timeout).
Nếu LLM sinh ra vòng lặp vô hạn `while True: pass`, Sandbox Pod sẽ bị treo không hồi kết, dần dần cạn kiệt K8s cluster resources của 300+ user.
**Đề xuất:** Sandbox Proxy phải có Time Controller: Truyền SIGTERM sau 10s và SIGKILL sau 15s nếu pod chưa hoàn thành. Cập nhật `SandboxSecurityConfig` với `MAX_EXECUTION_TIME_SEC = 15`.

#### A3. Quá tải Token (Context Bloat) khi Retry Fallback (§5.6.1.3)

**Vấn đề:** Tại Attempt 2 (Self-correct), mã nguồn truyền thẳng biến `last_error` về phía LLM. 
Traceback của Stack trace (từ Python Pandas/Scikit) hay SQL DB Engine Error có thể dài hàng chục ngàn token. Việc nhồi nhét này sẽ dẫn đến Timeout từ LLM API, hoặc chiếm hết Context window gây hallucination.
**Đề xuất:** Cần triển khai hàm `truncate_and_summarize_error(last_error)` giữ lại duy nhất 3-5 dòng Error name (e.g., `SyntaxError`, `ColumnNotFound`) trước khi nối vào Prompt để feedback cho LLM.

#### A4. Nghẽn cổ chai Context với Wren AI Schema (§4.4)

**Vấn đề:** Siêu thị có hàng trăm bảng (300+ tables) và hàng nghìn metrics/dimensions. Khi AI Agent gọi Wren AI để xác định Entity hoặc Generate SQL, liệu toàn bộ YAML config có được đẩy vào prompt của LLM một lúc hay không? Câu trả lời là nếu nạp tất cả, Context Limit sẽ bị phá vỡ và chi phí đội lên gấp nhiều lần.
**Đề xuất:** Wren AI cần được cấu hình chạy trong chế độ **RAG-based Semantic Retrieval**. Tức là phân mảnh MDL thành các Vector để khi có câu hỏi về "Sales", chỉ các Context về `fct_sales` và `dim_product` mới được đưa vào ngữ cảnh của Agent. Cần nhắc rõ việc này trong thiết kế.

---

## VI. Điểm Bổ Sung Lần 3 (Rà soát toàn văn bi-research.md — Antigravity)

Lần rà soát này đọc lại toàn bộ 5655 dòng `bi-research.md` một lượt mới, đối chiếu kỹ hơn giữa các section, phát hiện thêm **7 điểm** chưa được ghi nhận.

---

### 🔴 Lỗi Logic / Blocker

#### A5. `fct_revenue` dùng `COUNT(DISTINCT customer_name)` — sai HLL pattern (§5.6 Data Journey)

**Vấn đề:** Tại chặng 6️⃣ (dbt Marts Layer), mô hình `fct_revenue.sql` viết:
```sql
COUNT(DISTINCT customer_name) AS unique_customers
```
Đây **mâu thuẫn trực tiếp** với quyết định thiết kế đã được nhấn mạnh ở §5.2: `customer_count` phải là semi-additive và bắt buộc dùng HLL (`HLL_UNION_AGG`) để tránh double-counting khi rollup qua nhiều dimension.

**Hậu quả:** Nếu `fct_revenue` vẫn dùng `COUNT(DISTINCT customer_name)`, thì khi aggregation qua nhiều store/region/tháng → số khách bị đếm trùng. Đây là bug data integrity nghiêm trọng.

**Fix:** Đổi sang:
```sql
HLL_RAW_AGG(customer_id) AS customer_hll  -- trong dbt model
-- Trong query: HLL_UNION_AGG(customer_hll)
```
Hoặc xác nhận rõ: `fct_revenue` là mart cấp cao (đã group by `order_date, city, customer_segment`) — ở grain này `COUNT(DISTINCT)` có thể chấp nhận được nếu không dùng để rollup thêm. Nếu vậy phải document rõ grain và cảnh báo về giới hạn rollup.

---

#### A6. Reconciliation Pipeline thiếu xử lý `source_db = "production_pg"` nhưng source thực là MongoDB (§5.1.7)

**Vấn đề:** Code `reconcile_data()` trong Dagster viết:
```python
"source_db": "production_pg"  # ← nhưng source chính là MongoDB!
```
Toàn bộ document xác nhận MongoDB là primary source (~95%, 2TB+). Chỉ có ~5% là MySQL legacy, không có PostgreSQL nào là OLTP source.

**Hậu quả:** Reconciliation sẽ kết nối sai database, phép so sánh sẽ fail hoặc so sánh data sai. Mục đích phát hiện pipeline drift hoàn toàn bị vô hiệu hóa.

**Fix:** Đổi `source_db` sang `"production_mongodb"` và dùng MongoDB aggregation pipeline cho `source_query` (không phải SQL), hoặc dùng Airbyte-synced `_airbyte_raw` table trong StarRocks làm source của truth thay vì trực tiếp query MongoDB production.

---

### 🟠 Cải Tiến Kiến Trúc Quan Trọng

#### A7. Không có strategy xử lý Stale-While-Revalidate cho Dashboard Viewer (§6.2)

**Vấn đề:** Cache TTL được đặt 5 phút. Khi cache expire:
- Request đầu tiên vào → cache miss → query StarRocks (`100-300ms`) → response → re-cache
- Trong `100-300ms` đó, tất cả concurrent requests cũng cache miss → gây **"cache stampede"** (tất cả request hit StarRocks cùng lúc).

Khi 300–500 users đồng loạt load dashboard sau 5 phút → StarRocks bị spike load đột ngột.

**Đề xuất:** Implement **Stale-While-Revalidate (SWR)** pattern tại tầng Redis:
1. Đặt 2 TTL: `DATA_TTL = 5 phút` (khi nào stale) và `GRACE_TTL = 10 phút` (khi nào expired hoàn toàn)
2. Request đến trong `5–10 phút`: serve stale data ngay + trigger 1 background revalidation
3. Dùng Redis `SETNX` lock để đảm bảo chỉ 1 background revalidation chạy
4. Kết quả: User luôn nhận response `<50ms`, StarRocks chỉ bị query 1 lần mỗi 5 phút

---

#### A8. Graphic Walker integration với RBAC chưa được define (§6.10)

**Vấn đề:** Tài liệu giới thiệu Graphic Walker (Layer 5 — optional) và mô tả server-side computation qua `/api/gw-query`. Tuy nhiên, `translateGWSpecToSQL()` được nhắc tới nhưng **không được implement hay document**:
```typescript
const sql = translateGWSpecToSQL(spec, permissions);  // ← hàm này là gì?
```

GW gửi **aggregate specification** thay vì SQL — bao gồm `measures`, `dimensions`, `filters`. Hàm translate này cần:
1. Map GW spec fields → StarRocks column names
2. Apply RBAC row-level filters (region/store restrictions)
3. Validate không truy cập sensitive tables

Nếu thiếu implementation, Graphic Walker sẽ bypass RBAC và expose raw data.

**Đề xuất:** Document rõ spec translation logic hoặc dùng Graphic Walker's built-in `IDataQueryPayload` → convert sang StarRocks-compatible SQL với RBAC filter injection theo cùng pattern như `ChatRBACMiddleware`.

---

### 🟡 Cải Tiến Chất Lượng

#### A9. Cost estimation dùng tên model lỗi thời (§9.3.1)

**Vấn đề:** Bảng cost estimation ghi `"Claude Sonnet 4"` với giá `$3/1M input`. Tên model thực tế của Anthropic là `claude-sonnet-4-5` hoặc `claude-3-5-sonnet` — "Claude Sonnet 4" không phải tên chính xác, gây khó tra cứu pricing khi implement.

Tương tự, **Gemini 2.5 Flash** có tier giá khác nhau (`thinking` vs `non-thinking` mode) — document chưa nêu rõ context.

**Đề xuất:**
- Chuẩn hóa tên model theo API model ID chính thức (ví dụ: `claude-sonnet-4-5`, `gpt-4o`, `gemini-2.5-flash-preview`)
- Note rõ pricing có thể thay đổi — link tới trang pricing chính thức thay vì hard-code giá

---

#### A10. BlockNote pre-1.0 risk mitigation thiếu version pinning strategy (§6.0.5)

**Vấn đề:** Document đã thêm `IReportEditor` abstraction layer rất tốt. Tuy nhiên trong comment code:
```typescript
// Pinned to specific version: @blocknote/react@0.47.x
```
Không có hướng dẫn cụ thể để:
1. **Lock version** trong `package.json` (dùng `~0.47.0` hay `0.47.x` exact?)
2. **Monitor** cho breaking changes (renovate bot? manual check?)
3. **Test suite** để detect khi nào update BlockNote vô tình break `IReportEditor` contract
4. **Changelog monitoring** — ai theo dõi BlockNote releases?

**Đề xuất:** Thêm vào §6.0.5 Fallback Plan:
- Pin chính xác: `"@blocknote/react": "0.47.x"` (patch-level updates OK, minor = review first)
- Renovate Bot config: auto-update chỉ patch, minor/major cần manual approval
- Contract test: 5 integration tests cho `IReportEditor` methods — chạy trong CI sau mỗi dep update

---

#### A11. Thiếu MDL versioning strategy khi deploy Wren AI MDL lên production (§4.4.4)

**Vấn đề:** MDL làm "single source of truth" cho toàn bộ metric definitions. Hiện tại §4.4.4 mô tả MDL maintenance workflow:
- MDL thay đổi → restart Wren AI Service để reload

Nhưng không có:
1. **MDL versioning**: Làm thế nào để rollback MDL về version cũ nếu deploy mới gây lỗi?
2. **Zero-downtime reload**: Restart Wren AI = gián đoạn service. Không có blue/green deploy.
3. **Validation trước deploy**: AI Agent có test MDL mới trước khi switch production hay không?

**Hậu quả:** Một MDL deploy sai có thể làm hỏng NL→SQL cho toàn bộ user, không có rollback nhanh.

**Đề xuất:**
1. Lưu MDL dưới dạng Git-versioned YAML (đã được mention) + thêm Git tag cho mỗi production deploy
2. **Pre-deploy validation**: Chạy bộ test NL queries canonical (50 câu chuẩn) trên MDL mới trước khi deploy production
3. **Blue/Green MDL deploy**: Wren AI support multiple MDL profiles → giữ `MDL_v1` (production) và `MDL_v2` (staging), switch traffic khi validated
4. **Rollback script**: `git checkout <prev-tag> -- mdl.yaml && kubectl rollout restart wren-ai`

---

## VII. Tổng Hợp Ưu Tiên (Cập Nhật Đầy Đủ)

| Mức độ | Vấn đề | Section | Lần phát hiện |
|:-------|:-------|:--------|:--------------|
| 🔴 **Blocker** | MV dùng cột không tồn tại (`customer_count`) | §5.5 | I |
| 🔴 **Blocker** | StarRocks param style `$1,$2` sai (nên dùng `?`) | §6.2 | I |
| 🔴 **Blocker** | `reconcile_data` kết nối sai `production_pg` thay vì MongoDB | §5.1.7 | III |
| 🔴 **Security** | SQL injection trong RBAC filter injection | §7.3 | I |
| 🔴 **Logic** | `fct_revenue` dùng `COUNT(DISTINCT)` thay vì HLL — lỗi double-count | §5.6 | III |
| 🔴 **Logic** | Entity Guard bỏ qua `filters` và `granularity` trong Semantic Cache | §5.6.2.3 | II |
| 🟠 **High** | Embedding model tiếng Anh cho dữ liệu tiếng Việt | §5.6.2.3 | I |
| 🟠 **High** | Thiếu rate limiting cho AI Chat | §7 | I |
| 🟠 **High** | RBAC assume alias `s` luôn tồn tại trong SQL | §7.3 | I |
| 🟠 **High** | Sandbox Timeout không được enforce — infinite loop risk | §5.6 | II |
| 🟠 **High** | Graphic Walker integration thiếu RBAC + `translateGWSpecToSQL` | §6.10 | III |
| 🟠 **High** | Cache stampede khi TTL expire — thiếu SWR pattern | §6.2 | III |
| 🟡 **Medium** | Auth Phase 4 — quá muộn | §8 | I |
| 🟡 **Medium** | StarRocks không có backup native | §9.8 | I |
| 🟡 **Medium** | K8s Pod pool pre-warming | §2.2 | I |
| 🟡 **Medium** | Session persistence LangGraph chưa define | §4.3 | I |
| 🟡 **Medium** | Context bloat khi Retry — cần `truncate_and_summarize_error()` | §5.6.1.3 | II |
| 🟡 **Medium** | Wren AI cần RAG-based Semantic Retrieval (không load full MDL) | §4.4 | II |
| 🟡 **Medium** | MDL versioning + zero-downtime deploy + rollback thiếu | §4.4.4 | III |
| 🟡 **Medium** | BlockNote version pinning + contract test + monitoring strategy | §6.0.5 | III |
| 🟢 **Low** | Numbering section 5.7/5.6.x | §5.7 | I |
| 🟢 **Low** | Code comment làm dict key trong WrenAIClient | §4.4.3 | I |
| 🟢 **Low** | PostgreSQL RAM spec mâu thuẫn 8GB vs 4GB | §9.3.2/§9.6 | I |
| 🟢 **Low** | Tên model LLM lỗi thời trong cost estimation | §9.3.1 | III |

---

## V. Version Tracking

| Version | Date | Author | Description |
|:---|:---|:---|:---|
| 1.0.0 | 2026-04-09 | Claude Code Core | Khởi tạo tài liệu phân tích sâu, tìm ra 10 mâu thuẫn và 15 điểm cải tiến. |
| 1.1.0 | 2026-04-09 | Antigravity | Rà soát chéo (Cross-review), bổ sung 4 lỗ hổng nghiêm trọng (Cache Logic, Sandbox Timeout, LLM Context Bloat, Schema RAG) và cập nhật Version Tracking. |
| 1.2.0 | 2026-04-09 | Antigravity | Rà soát toàn văn lần 3 (đọc lại toàn bộ 5655 dòng bi-research.md), bổ sung 7 điểm mới (A5–A11): fct_revenue HLL bug, reconciliation source DB sai, SWR cache stampede, GW RBAC gap, MDL versioning, BlockNote pinning, LLM model naming. Cập nhật bảng tổng hợp ưu tiên đầy đủ. |
