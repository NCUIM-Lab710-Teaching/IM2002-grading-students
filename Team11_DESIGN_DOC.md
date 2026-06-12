# Section 2 — Normalisation Justification

## 2.1 Design Goal

The relational database design of TransitFlow follows Third Normal Form (3NF) as the main design principle. The goal is to reduce unnecessary data duplication, maintain data consistency, and avoid common database anomalies such as insert anomalies, update anomalies, and delete anomalies.

The schema separates the major entities of the system into different tables, including users, credentials, stations, station links, schedules, seat layouts, bookings, metro travel history, payments, feedback, and policy documents. This separation allows each table to focus on one main type of data and reduces the risk of mixing unrelated information in the same relation.

At the same time, TransitFlow keeps a small number of controlled denormalised fields in transaction-related tables, such as `amount_usd` and `stops_travelled`. These fields are stored directly in booking and travel history records to preserve the actual transaction state at the time of purchase.

---

## 2.2 First Normal Form (1NF)

First Normal Form requires each record to be uniquely identifiable and each field to store values in a structured and consistent way. In TransitFlow, all major tables have clear primary keys:

| Table | Primary Key | Purpose |
|---|---|---|
| `registered_users` | `user_id` | Stores user profile data |
| `user_credentials` | `credential_id` | Stores login and security credentials |
| `national_rail_stations` | `station_id` | Stores national rail station data |
| `metro_stations` | `station_id` | Stores metro station data |
| `national_rail_schedules` | `schedule_id` | Stores national rail schedule data |
| `metro_schedules` | `schedule_id` | Stores metro schedule data |
| `national_rail_bookings` | `booking_id` | Stores national rail booking records |
| `metro_travel_history` | `trip_id` | Stores metro travel records |
| `payments` | `payment_id` | Stores payment records |
| `feedback` | `feedback_id` | Stores user feedback |

Most fields store single values, such as `email`, `phone`, `travel_date`, `fare_class`, `seat_id`, `amount_usd`, `method`, `status`, and `paid_at`.

Some schedule-related tables use PostgreSQL structured types such as `TEXT[]` and `JSONB`. For example, `stops_in_order TEXT[]` stores the ordered list of stops in a schedule, while `travel_time_from_origin_min JSONB` stores the travel time from the origin station to each stop. `fare_classes JSONB` stores fare class information for national rail schedules, and `coaches JSONB` stores seat layout information.

From a strict theoretical relational model, these fields could be further decomposed into separate child tables such as `schedule_stops` or `seat_layout_details`. However, in this project, these values are usually accessed as one complete schedule or one complete seat layout. Therefore, storing them as structured attributes is a practical design choice that improves query simplicity and performance while still keeping the schema organised.

---

## 2.3 Second Normal Form (2NF)

Second Normal Form requires that all non-key attributes fully depend on the whole primary key. This is especially important for tables with composite primary keys.

Most tables in TransitFlow use a single-column primary key, such as `user_id`, `schedule_id`, `booking_id`, `trip_id`, and `payment_id`. Therefore, partial dependency is naturally avoided in these tables.

For station connection tables, TransitFlow uses composite primary keys:

| Table | Composite Primary Key | Non-key Attribute |
|---|---|---|
| `national_rail_links` | `(from_station_id, to_station_id, line)` | `travel_time_min` |
| `metro_links` | `(from_station_id, to_station_id, line)` | `travel_time_min` |

In these tables, `travel_time_min` depends on the full combination of the origin station, destination station, and line. Knowing only the origin station is not enough to determine the travel time, because one station can connect to multiple other stations. Knowing only the line is also not enough, because the same line contains multiple station-to-station segments. Therefore, the non-key attribute is fully dependent on the whole composite key, which satisfies 2NF.

---

## 2.4 Third Normal Form (3NF)

Third Normal Form requires that non-key attributes do not depend on other non-key attributes. In other words, each non-key field should directly describe the entity identified by the primary key.

In `registered_users`, fields such as `full_name`, `email`, `phone`, `date_of_birth`, `registered_at`, `is_active`, and `deleted_at` directly describe the user identified by `user_id`. Login credentials are not stored in the same table. Instead, password and security question data are separated into `user_credentials`, which stores `password_hash`, `secret_question`, and `secret_answer_hash`.

This separation improves both normalisation and security. User profile data and authentication data can be updated independently. For example, changing a user phone number or email does not require modifying password-related fields.

The same design principle is applied to schedules and bookings. `national_rail_schedules` stores reusable schedule information such as line, service type, direction, station order, travel time, and fare classes. Actual user transactions are stored in `national_rail_bookings`. A booking references a schedule through `schedule_id` instead of duplicating all schedule details in every booking record.

Similarly, metro travel records are stored in `metro_travel_history`, while reusable metro schedule data are stored in `metro_schedules`. This avoids repeating schedule information in every trip record.

Payment information is also stored separately in `payments`. This allows booking status and payment status to be managed independently. For example, a booking may be `confirmed` or `cancelled`, while a payment may be `paid` or `refunded`. In the current schema, `payments.booking_id` is used as a logical reference to a booking record, although it is not explicitly declared as a foreign key.

---

## 2.5 Referential Integrity

The schema uses primary keys and foreign keys to maintain relationships between major entities.

For national rail bookings, `national_rail_bookings.user_id` references `registered_users.user_id`, `schedule_id` references `national_rail_schedules.schedule_id`, and both `origin_station_id` and `destination_station_id` reference `national_rail_stations.station_id`. This ensures that each booking is associated with a valid user, schedule, and pair of stations.

For metro travel history, `metro_travel_history.user_id` references `registered_users.user_id`, `schedule_id` references `metro_schedules.schedule_id`, and both origin and destination station IDs reference `metro_stations.station_id`. The `day_pass_ref` field also references `metro_travel_history.trip_id`, allowing day pass related trips to be linked to an existing metro travel record.

For station connectivity, `national_rail_links` references `national_rail_stations`, while `metro_links` references `metro_stations`. This ensures that station-to-station links cannot point to non-existent stations.

The `feedback` table stores user feedback with a foreign key to `registered_users.user_id`. This allows feedback records to be associated with valid users.

---

## 2.6 Controlled Denormalisation

Although the schema mainly follows 3NF, TransitFlow intentionally stores some calculated values in transaction tables. The most important examples are:

| Table | Denormalised Field | Reason |
|---|---|---|
| `national_rail_bookings` | `amount_usd` | Stores the actual fare charged at booking time |
| `national_rail_bookings` | `stops_travelled` | Stores the actual number of stops travelled |
| `metro_travel_history` | `amount_usd` | Stores the actual metro fare or pass amount |
| `metro_travel_history` | `stops_travelled` | Stores the actual number of metro stops travelled |

These fields could theoretically be recalculated from schedule data, station order, and fare rules. However, recalculating them every time would create a risk for historical transaction records.

Transit fares, discounts, service rules, and fare classes may change over time. If an old booking were recalculated using the newest fare rule, the result might not match the amount that the passenger actually paid. To avoid this problem, TransitFlow stores `amount_usd` and `stops_travelled` directly in the transaction record at the time of booking or travel.

This is a controlled denormalisation decision. It improves the reliability of historical transaction records and supports financial auditing, because the system can preserve the original charged amount even if fare rules are changed later.

---

## 2.7 Summary

Overall, TransitFlow’s relational schema follows the main principles of 3NF by separating user profiles, credentials, stations, station links, schedules, bookings, metro travel history, payments, and feedback into different tables. This reduces redundancy, improves data consistency, and prevents unrelated data from being stored together.

The schema also uses foreign keys to maintain referential integrity for users, stations, schedules, bookings, and travel records. At the same time, controlled denormalisation is applied to transaction records through fields such as `amount_usd` and `stops_travelled`. This design preserves historical fare information and ensures that past bookings remain accurate even if future schedules or fare policies change.

Therefore, the TransitFlow relational design balances normalisation, data integrity, query efficiency, and practical transaction requirements.


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
