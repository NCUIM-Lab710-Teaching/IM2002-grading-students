# Section 2 — 正規化合理性論述（Normalisation Justification）

## 2.1 設計目標（Design Goal）

TransitFlow 的 PostgreSQL 關係式資料庫以第三正規化（Third Normal Form, 3NF）作為主要設計原則。其設計目標是降低不必要的資料重複、維持資料一致性，並避免常見的資料庫異常問題，例如新增異常（Insert Anomaly）、修改異常（Update Anomaly）與刪除異常（Delete Anomaly）。

本系統將主要資料實體分離至不同資料表中，包括使用者、登入憑證、車站、站間連線、班次、座位配置、訂票紀錄、捷運旅程紀錄、付款紀錄、意見回饋與政策文件。透過這樣的分表設計，每一張資料表都能專注於一種主要資料類型，避免將不相關的資料混合在同一個 relation 中。

同時，TransitFlow 也在交易相關資料表中保留少量控制型反正規化欄位，例如 `amount_usd` 與 `stops_travelled`。這些欄位會直接儲存在訂票紀錄與旅程歷史中，用來保存乘客在購票或搭乘當下的實際交易狀態。

---

## 2.2 第一正規化（First Normal Form, 1NF）

第一正規化要求每一筆紀錄都能被唯一識別，且欄位內容應以一致且結構化的方式儲存。在 TransitFlow 中，所有主要資料表皆具備明確的主鍵：

| 資料表 | 主鍵 | 用途 |
|---|---|---|
| `registered_users` | `user_id` | 儲存使用者基本資料 |
| `user_credentials` | `credential_id` | 儲存登入與安全憑證 |
| `national_rail_stations` | `station_id` | 儲存國鐵車站資料 |
| `metro_stations` | `station_id` | 儲存捷運車站資料 |
| `national_rail_schedules` | `schedule_id` | 儲存國鐵班次資料 |
| `metro_schedules` | `schedule_id` | 儲存捷運班次資料 |
| `national_rail_bookings` | `booking_id` | 儲存國鐵訂票紀錄 |
| `metro_travel_history` | `trip_id` | 儲存捷運旅程紀錄 |
| `payments` | `payment_id` | 儲存付款紀錄 |
| `feedback` | `feedback_id` | 儲存使用者意見回饋 |

大部分欄位皆儲存單一值，例如 `email`、`phone`、`travel_date`、`fare_class`、`seat_id`、`amount_usd`、`method`、`status` 與 `paid_at`。

部分班次相關資料表使用 PostgreSQL 的結構化型別，例如 `TEXT[]` 與 `JSONB`。例如：

- `stops_in_order TEXT[]`：儲存該班次依序經過的車站。
- `travel_time_from_origin_min JSONB`：儲存從起點站到各站的累積行車時間。
- `fare_classes JSONB`：儲存國鐵不同艙等的票價設定。
- `coaches JSONB`：儲存國鐵座位配置資訊。

若從最嚴格的傳統關係模型來看，這些欄位可以進一步拆成子表，例如 `schedule_stops` 或 `seat_layout_details`。然而，在本專案中，這些資料通常會以完整班次或完整座位配置的形式被查詢與計算。因此，將其儲存為結構化屬性是一種實務上的設計選擇，可提升查詢簡潔性與執行效率，同時仍能維持資料結構的清楚性。

---

## 2.3 第二正規化（Second Normal Form, 2NF）

第二正規化要求所有非鍵欄位必須完全相依於整個主鍵，而不能只相依於複合主鍵的一部分。此要求對具有複合主鍵的資料表尤其重要。

TransitFlow 大部分資料表使用單一欄位作為主鍵，例如 `user_id`、`schedule_id`、`booking_id`、`trip_id` 與 `payment_id`。因此，這些資料表自然不會產生部分相依（Partial Dependency）的問題。

在站間連線資料表中，TransitFlow 使用複合主鍵來描述兩站之間的實體連線：

| 資料表 | 複合主鍵 | 非鍵欄位 |
|---|---|---|
| `national_rail_links` | `(from_station_id, to_station_id, line)` | `travel_time_min` |
| `metro_links` | `(from_station_id, to_station_id, line)` | `travel_time_min` |

在這兩張表中，`travel_time_min` 必須由起站、迄站與路線三者共同決定。只知道起站不足以決定行車時間，因為同一個車站可能連接多個不同方向；只知道路線也不足以決定行車時間，因為同一路線中包含多個站間區段。因此，`travel_time_min` 完全相依於整個複合主鍵，符合 2NF 的要求。

---

## 2.4 第三正規化（Third Normal Form, 3NF）

第三正規化要求非鍵欄位之間不應存在遞移相依（Transitive Dependency）。換言之，每一個非鍵欄位都應該直接描述該表主鍵所代表的實體，而不應透過另一個非鍵欄位間接決定。

在 `registered_users` 中，`full_name`、`email`、`phone`、`date_of_birth`、`registered_at`、`is_active` 與 `deleted_at` 皆直接描述由 `user_id` 所代表的使用者。登入憑證並未直接儲存在同一張表中，而是拆分至 `user_credentials`。該表保存 `password_hash`、`secret_question` 與 `secret_answer_hash` 等登入與安全相關資料。

這種分離設計同時提升正規化程度與安全性。使用者基本資料與驗證資料可以獨立更新，例如修改使用者電話或電子郵件時，不需要同時修改密碼雜湊或安全問題相關欄位。

相同的設計原則也應用於班次與訂票資料。`national_rail_schedules` 儲存可重複使用的班次資料，例如路線、服務類型、方向、站序、行車時間與票價艙等。實際使用者交易則儲存在 `national_rail_bookings` 中。訂票紀錄透過 `schedule_id` 參照班次，而不是在每一筆訂單中重複儲存完整班次資訊。

同樣地，捷運旅程紀錄儲存在 `metro_travel_history`，可重複使用的捷運班次資料則儲存在 `metro_schedules`。這樣可以避免在每一筆旅程紀錄中重複儲存班次資料。

付款資訊則獨立儲存在 `payments` 表中，使訂票狀態與付款狀態可以分開管理。例如，訂票紀錄的狀態可能是 `confirmed` 或 `cancelled`，而付款紀錄的狀態可能是 `paid` 或 `refunded`。在目前的 schema 中，`payments.booking_id` 作為付款紀錄對應訂票紀錄的邏輯參照欄位，但未明確宣告為外鍵。

---

## 2.5 參照完整性（Referential Integrity）

TransitFlow 的 schema 使用主鍵與外鍵維持主要資料實體之間的關係。

在國鐵訂票資料中，`national_rail_bookings.user_id` 參照 `registered_users.user_id`，`schedule_id` 參照 `national_rail_schedules.schedule_id`，而 `origin_station_id` 與 `destination_station_id` 則參照 `national_rail_stations.station_id`。這能確保每一筆國鐵訂票紀錄都對應到有效的使用者、班次與起訖車站。

在捷運旅程資料中，`metro_travel_history.user_id` 參照 `registered_users.user_id`，`schedule_id` 參照 `metro_schedules.schedule_id`，而起訖站欄位則參照 `metro_stations.station_id`。此外，`day_pass_ref` 也會參照 `metro_travel_history.trip_id`，用來支援與日票相關的旅程紀錄。

在站間連線資料中，`national_rail_links` 參照 `national_rail_stations`，`metro_links` 則參照 `metro_stations`。這能確保站間連線不會指向不存在的車站。

`feedback` 表中的 `user_id` 也參照 `registered_users.user_id`，使意見回饋紀錄能對應到有效使用者。

---

## 2.6 控制型反正規化（Controlled Denormalisation）

雖然 TransitFlow 的 schema 主要遵循 3NF，但系統刻意在交易資料表中保存部分計算結果。主要例子如下：

| 資料表 | 反正規化欄位 | 保留原因 |
|---|---|---|
| `national_rail_bookings` | `amount_usd` | 保存訂票當下實際收取的票價 |
| `national_rail_bookings` | `stops_travelled` | 保存該筆訂票實際搭乘站數 |
| `metro_travel_history` | `amount_usd` | 保存捷運票價或票券金額 |
| `metro_travel_history` | `stops_travelled` | 保存捷運實際搭乘站數 |

這些欄位理論上可以透過班次資料、站序與票價規則重新計算。然而，若每次查詢舊訂單時都重新計算，會對歷史交易紀錄造成風險。

交通票價、折扣、服務規則與票價艙等可能會隨時間變更。如果舊訂單使用最新票價規則重新計算，結果可能會與乘客當時實際支付的金額不同。為避免此問題，TransitFlow 在訂票或旅程發生當下，直接將 `amount_usd` 與 `stops_travelled` 寫入交易紀錄。

這是一種控制型反正規化設計。其目的不是任意重複資料，而是保存歷史交易正確性，並支援財務紀錄與後續查詢。即使未來票價規則改變，系統仍能保留過去實際收取的金額。

---

## 2.7 小結（Summary）

整體而言，TransitFlow 的 PostgreSQL schema 透過分表設計符合 3NF 的主要精神。系統將使用者基本資料、登入憑證、車站、站間連線、班次、座位配置、訂票紀錄、捷運旅程紀錄、付款紀錄與意見回饋分開管理，降低資料重複並提升資料一致性。

系統也透過外鍵維持使用者、車站、班次、訂票與旅程紀錄之間的參照完整性。同時，針對交易紀錄，系統保留 `amount_usd` 與 `stops_travelled` 等控制型反正規化欄位，以保存歷史票價與實際搭乘狀態。

因此，TransitFlow 的關係式資料庫設計在正規化、資料完整性、查詢效率與實務交易需求之間取得平衡，符合雙軌道交通查詢與訂票系統的設計目標。
# Section 4 — Vector / RAG Design

## 4.1 Embedded Content and Semantic Search

TransitFlow uses a Retrieval-Augmented Generation (RAG) architecture to provide grounded answers for help-desk and policy-related questions. The system stores policy documents in the `policy_documents` table and converts them into vector embeddings for semantic retrieval.

The embedded content includes documents such as:

- Ticket booking policies
- Refund and cancellation rules
- Passenger conduct guidelines
- Travel and payment regulations

Each document is transformed into a high-dimensional vector representation using an embedding model. Instead of matching keywords directly, the system searches for documents with similar semantic meaning.

TransitFlow uses **cosine similarity** for vector search. Cosine similarity compares the angle between two embedding vectors rather than their magnitude.

This is important because documents may differ in length while still expressing similar concepts. In the embedding space, semantically related texts tend to point in similar directions, so cosine similarity can retrieve relevant documents even when they do not share the same keywords.

The method is therefore well suited for semantic search and question answering.

---

## 4.2 RAG Pipeline

The TransitFlow RAG workflow consists of the following stages:

```text
User Question
      ↓
Query Embedding
      ↓
Cosine Similarity Search (pgvector)
      ↓
Retrieved Policy Documents
      ↓
Prompt Construction
      ↓
LLM Response
```

### Step 1: User Query

The user submits a natural-language question, for example:

> How do I request a refund?

### Step 2: Query Embedding

The question is converted into an embedding vector using the same embedding model that was used to generate the document embeddings.

### Step 3: Similarity Search

PostgreSQL with the pgvector extension performs a cosine-similarity search over the `policy_documents.embedding` column.

### Step 4: Document Retrieval

The system retrieves the most relevant policy documents based on similarity scores.

### Step 5: Prompt Construction

The retrieved documents are inserted into the prompt as contextual information.

### Step 6: LLM Response

The language model generates a final answer grounded in the retrieved documents, reducing hallucinations and improving factual consistency.

---

## 4.3 Embedding Dimension Choice

The current implementation uses **Ollama's `nomic-embed-text` model**, which produces **768-dimensional embeddings**.

Accordingly, the database schema defines:

```sql
embedding vector(768)
```

The schema comments also document an alternative configuration:

| Provider | Embedding Dimension |
|-----------|-----------|
| Ollama (`nomic-embed-text`) | 768 |
| Gemini (`gemini-embedding-001`) | 3072 |

The embedding dimension is part of the database schema and vector index definition. Therefore, all vectors stored within the same column must have identical dimensionality.

---

## 4.4 Provider Change After Seeding

A practical consideration is what happens if the embedding provider is changed after the database has already been seeded.

Suppose the system was initially seeded using Ollama embeddings (768 dimensions). If the provider is later switched to Gemini embeddings (3072 dimensions), the new vectors will no longer match the existing schema and vector index.

This creates a **dimension mismatch** problem.

### Consequences

1. New embeddings cannot be stored in a `vector(768)` column.
2. Existing vector indexes become unusable.
3. Similarity search operations fail because vectors with different dimensions cannot be compared directly.

### Recovery Procedure

To resolve this issue:

1. Update the schema to the new dimension:

```sql
embedding vector(3072)
```

2. Drop and recreate the vector index.
3. Regenerate embeddings for all policy documents.
4. Re-seed the database using the new embedding provider.

Therefore, the embedding provider should ideally be finalized before large-scale seeding and indexing are performed.

---

## 4.5 Summary

TransitFlow's RAG design embeds policy documents into vector representations and uses cosine similarity search to retrieve semantically related information.

The retrieved documents are injected into the LLM prompt to produce grounded answers. This improves answer quality, reduces hallucinations, and ensures that responses are based on the stored policy knowledge.

The current implementation uses 768-dimensional Ollama embeddings. If a different embedding provider is adopted in the future, the vector schema and index must be rebuilt to match the new embedding dimensionality.
