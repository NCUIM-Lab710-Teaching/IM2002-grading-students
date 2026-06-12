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

# Section 5 — AI Tool Usage Evidence

## Example 1 — Database Schema Design

### Context

We were designing the PostgreSQL database schema for TransitFlow. One design challenge was deciding how user authentication data should be stored.

### Prompt

```text
Should password hashes be stored in the same table as user profile information, or should they be separated into a dedicated credentials table?
```

### Outcome

The AI suggested separating authentication data into a dedicated `user_credentials` table linked to `registered_users` through `user_id`.

After reviewing the recommendation, we adopted the design because it improves security, reduces redundancy, and supports Third Normal Form (3NF).

---

## Example 2 — Available Seat Query Development

### Context

We were implementing the `query_available_seats()` function for national rail bookings. The seat layouts were stored as JSONB objects and booked seats needed to be excluded.

### Prompt

```text
Write a PostgreSQL query that returns available seats from a JSONB seat layout while excluding seats that have already been booked.
```

### Outcome

The AI provided an initial approach using PostgreSQL JSONB functions.

The generated solution helped us understand how to traverse nested JSON structures and filter occupied seats. We later adapted the logic and integrated it into the final implementation of `query_available_seats()`.

---

## Example 3 — Booking Flow Debugging

### Context

During development, the booking workflow was failing when users entered booking requests through the Gradio interface.

### Prompt

```text
The booking command is not triggering the booking function. Review the intent detection logic and identify possible causes.
```

### Outcome

The AI suggested checking the booking trigger keywords, login validation logic, and fallback routing conditions.

Using these suggestions, we identified issues in the intent-detection logic and updated the trigger conditions. After the correction, booking requests could successfully reach the booking workflow.

---

## Example 4 — AI Error and Correction

### Context

We used AI assistance to review our schema and identify foreign key relationships.

### Prompt

```text
Review the TransitFlow schema and identify all primary key and foreign key relationships.
```

### Outcome

The AI initially assumed that `payments.booking_id` and `feedback.booking_id` already had foreign key constraints defined.

However, after manually reviewing the schema, we discovered that these foreign keys had not actually been declared in the SQL schema.

We corrected the design by adding the missing foreign key relationships and updating the ER diagram accordingly.

This experience demonstrated the importance of verifying AI-generated suggestions against the actual implementation before accepting them.