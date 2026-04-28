# Voice AI Email Assistant

A voice-powered email assistant with persistent memory. The agent can search a faux inbox, recall past conversations, look up contacts, and send emails on your behalf. An analytics dashboard at `/analytics` lets you inspect the agent's memory, tool calls, session history, and the full knowledge graph.

The voice UI is built on LiveKit's [agent-starter-react](https://github.com/livekit-examples/agent-starter-react) template and the agent on their [agent-starter-python](https://github.com/livekit-examples/agent-starter-python) template. Using these as a foundation meant the LiveKit speech interface (STT → LLM → TTS, room connection, transcript UI) was up and running quickly, allowing me to focus on the memory architecture instead.

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15, Tailwind CSS, LiveKit Agents UI |
| Agent | Python, LiveKit Agents 1.5.6, FastAPI |
| LLM | OpenAI gpt-4o-mini |
| STT | Deepgram nova-3 (via LiveKit inference) |
| TTS | Cartesia sonic-3 (via LiveKit inference) |
| VAD | Silero |
| Graph DB | Neo4j 5 |
| Vector DB | Qdrant |
| Relational DB | PostgreSQL 16 |
| Orchestration | Docker Compose |
| LiveKit server | LiveKit Cloud |

---

## Implementation

### Storage

Three storage layers are used in combination:

**PostgreSQL** stores session transcripts, session summaries, and the faux email inbox. 

**Neo4j** is the knowledge graph. Node and relationship types are not predefined; the extraction LLM decides them per email/session (e.g. `Person`, `Project`, `Organization`, `WORKS_ON`, `REPORTS_TO`). The only fixed node is `User`. Every entity merges by name across sessions so the same person mentioned in ten emails stays as a single node. Each node stores an email address, summary, aliases, and IDs of associated emails and events for fast cross-referencing.

**Qdrant** holds three collections:
- `emails` — embeddings of email bodies with payload metadata (sender, recipients, date, labels) for filtered semantic search
- `entities` — embeddings of entity names and summaries, used to resolve a fuzzy name query to a graph node (this is the GraphRAG pattern described in [Neo4j's GraphRAG guide](https://neo4j.com/blog/genai/what-is-graphrag/))
- `events` — embeddings of structured events extracted from session transcripts (decisions, preferences, commitments, etc.)

---

### Agent Tools

The agent has 7 tools:

| Tool | Description |
|---|---|
| `search_emails` | Semantic search over the `emails` in the vector DB. Optional `from_email`, `to_emails`, and date filters narrow the candidate set before the vector search, reducing latency. Returns the top results with ID, metadata, and a 160-character body preview. |
| `get_email` | Fetches the full body of a specific email by its ID. |
| `search_entities` | Embeds the query, finds the best-matching entity in the vector DB, then fetches the full node from Neo4j. Returns properties, aliases, email address, and optionally all direct relationships. |
| `get_entity` | Fetches a Neo4j node by its exact name. |
| `search_events` | Semantic search over events in the vector DB. Can filter by session ID or entity name list. Returns event descriptions, timestamps, and session IDs. |
| `get_session_transcript` | Retrieves the full transcript of a past session by ID from Postgres. |
| `send_email` | Composes and sends an email on behalf of the user. Validates recipient addresses, non-empty subject and body, then writes the email to Postgres and runs the email ingestion pipeline on it so the sent email is immediately reflected in the graph and vector DB. |

---

### Session Context

At the start of every session, the following is injected into the system prompt:

1. **Last 10 events** across all sessions including descriptions, timestamps, and entity names.
2. **Full transcript** of the most recent session (from Postgres)
3. **Summaries** of the 3 sessions before that (from Postgres)

The system prompt explicitly instructs the model to check this injected context before reaching for tools, and to only call a tool when the context is insufficient or the user needs more detail than what's summarized.

---

### Ingestion Pipeline

Two pipelines, both using GPT for extraction and OpenAI `text-embedding-3-small` for embeddings:

**Email ingestion** that I ran on the faux emails once, and it also runs for every email sent through  `send_email`. For each email it:
1. Calls GPT with the email body and the existing graph nodes/edges to extract entities and relationships as JSON
2. Upserts nodes and relationships into Neo4j
3. Embeds the email body and upserts the point into the vector DB with full metadata payload
4. Writes the email to Postgres

**Session ingestion** runs after a session ends. It:
1. Calls GPT to extract a summary, events, entities, and relationships from the transcript
2. Saves the transcript and summary to Postgres
3. Embeds and upserts events into the vector DB
4. Upserts any new entities/relationships into Neo4j
5. Updates entity embeddings in the vector DB for any new or modified nodes

---

## Analytics Dashboard (`/analytics`)

- **Sessions** — browse past sessions, view full transcripts and summaries, delete sessions
- **Graph** — interactive Neo4j entity graph with node detail panel (relationships, properties, associated memory)
- **Inbox** — faux email client
- **Events** — browse all extracted memory events across sessions
- **Tools** — call any agent tool manually
- **Settings** — reset sessions/events, wipe memory, wipe everything, re-run email ingestion, normalize email addresses across Postgres and Qdrant

---

## Known Limitations

**Model quality.** Ingestion uses `gpt-4o-mini` and embeddings use `text-embedding-3-small`, the smallest available tiers. This directly impacts the quality of semantic search and knowledge graph extraction. Retrieval relevance is lower than it would be with larger models, and the agent will occasionally surface the wrong result or miss a relevant one.

**Knowledge graph over-extraction.** The Neo4j extraction prompt instructs the LLM to pass existing nodes and edges to avoid duplication, but with small models this doesn't hold reliably. Running ingestion on 38 faux emails produced ~100 nodes and ~400 relationships which is far more than intended. Duplicate nodes, spurious relationships, and over-eager entity creation are visible in the graph. The prompt and deduplication logic need more iteration, but each ingestion run has an API cost so extensive testing wasn't feasible within the time constraint.

**Agent tool utilization.** The agent sometimes underutilizes tools for example reading the injected context and treating it as ground truth rather than calling `search_emails` for fresh results (or it will call either `search_emails` or `search_events` rather than both), or it passes wrong parameters. This is a prompting problem and could be improved with more testing, but time limits and token budget constraints (using LiveKit free tier and paid OpenAI API) limited how much prompt iteration was possible.
