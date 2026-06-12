# n8n FAQ Chatbot with Semantic Search

AI-powered FAQ chatbot built in n8n. Answers user questions from a Postgres FAQ database using pgvector semantic search. Unknown questions are answered by an LLM, saved back to the database, and embedded automatically for future matching (self-learning loop).

## Architecture

**Parent workflow — `FAQ_Chatbot_Parent_FIXED.json`**
1. Chat trigger receives user message
2. Question normalized (e.g. "SAP" → "SAP B1")
3. Question embedded via OpenAI `text-embedding-3-small`
4. pgvector cosine similarity search against `faqs` table
5. If match found (similarity ≥ 0.5) → answer rewritten in friendly tone by LLM
6. If no match → LLM generates new answer, replies to user, saves Q&A to DB, and triggers the child workflow

**Child workflow — `Train_Agent_Child_FIXED.json`**
1. Triggered by parent after a new FAQ is saved
2. Selects all rows where `learned = false`
3. Generates embeddings for each Q&A pair
4. Stores vector in DB and marks row `learned = true`

## Requirements

- n8n (self-hosted or cloud)
- PostgreSQL with the `pgvector` extension
- OpenAI API key (chat model + embeddings)

## Database Setup

Run once:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS faqs (
  id SERIAL PRIMARY KEY,
  question TEXT NOT NULL,
  answer TEXT NOT NULL,
  embedding vector(1536),
  learned BOOLEAN DEFAULT false
);
```

If the table already exists:

```sql
ALTER TABLE faqs ADD COLUMN IF NOT EXISTS embedding vector(1536);
ALTER TABLE faqs ADD COLUMN IF NOT EXISTS learned BOOLEAN DEFAULT false;
UPDATE faqs SET learned = false WHERE embedding IS NULL;
```

## Installation

1. Import both JSON files into n8n (Workflows → Import from File)
2. Create credentials in n8n:
   - **Postgres** — attach to all Postgres nodes
   - **OpenAI** — attach to the chat model nodes, plus the two HTTP Request embedding nodes ("Embed User Question" in parent, "Generate Embedding" in child)
3. In the parent workflow, point the "Train New FAQs" Execute Workflow node at the imported child workflow
4. Run the child workflow manually once to embed any existing FAQs
5. Activate the parent workflow

## Configuration

- **Similarity threshold**: set to `0.5` in the "Check if FAQ Exists" IF node. Raise (0.6–0.7) if wrong matches appear; lower if too many questions fall through to AI generation.
- **Embedding model**: `text-embedding-3-small` (1536 dimensions). If you change the model, update the `vector(1536)` column dimension to match.
- **Memory**: conversation memory is keyed on the chat `sessionId` with a 20-message window.

## Security Notes

- No API keys are stored in the workflow JSON — all secrets live in n8n credentials.
- All SQL uses parameterized queries (`$1`, `$2`) — no string interpolation.

## License

MIT
