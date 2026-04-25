# SwitchIT — Technical Architecture Overview

SwitchIT is a two-sided career marketplace connecting job candidates with HR recruiters. It consists of two independently deployable backend services, a shared infrastructure layer, and a React frontend, with an embedded AI companion agent and a RAG (Retrieval-Augmented Generation) pipeline for contextual career guidance.

---

## Table of Contents

- [System Architecture](#system-architecture)
- [Service Overview](#service-overview)
- [AI Companion Agent (Darshan)](#ai-companion-agent-darshan)
- [RAG Pipeline](#rag-pipeline)
- [Candidate Platform — switcher-app-be](#candidate-platform--switcher-app-be)
- [Recruiter Platform — scout-backend](#recruiter-platform--scout-backend)
- [Frontend — app-frontend-website](#frontend--app-frontend-website)
- [Infrastructure — infra-server](#infrastructure--infra-server)
- [Databases](#databases)
- [Key Architectural Patterns](#key-architectural-patterns)
- [Environment Variables](#environment-variables)

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Browser                           │
│                  React 18 + Vite (port 5173)                    │
└──────────────────────┬──────────────────────┬───────────────────┘
                       │                      │
           ┌───────────▼────────┐  ┌──────────▼───────────┐
           │  switcher-app-be   │  │    scout-backend      │
           │  Fastify (3000)    │  │    Express (8080)     │
           │  Candidate API     │  │    HR/Recruiter API   │
           └───────────┬────────┘  └──────────┬────────────┘
                       │                      │
           ┌───────────▼──────────────────────▼────────────┐
           │              infra-server (Docker)             │
           │  ┌─────────────┐ ┌────────┐ ┌──────────────┐  │
           │  │ PostgreSQL  │ │ Redis  │ │Elasticsearch │  │
           │  │ 15+pgvector │ │   7    │ │   9.2.4      │  │
           │  │ port 5432   │ │  6379  │ │   port 9200  │  │
           │  └─────────────┘ └────────┘ └──────────────┘  │
           └────────────────────────────────────────────────┘
```

External AI/ML services:

| Service | Usage |
|---------|-------|
| Groq API (`llama-3.3-70b-versatile`) | Companion agent chat inference |
| OpenAI (`text-embedding-3-small`) | RAG document embeddings |
| OpenAI Vision API | Resume PDF parsing |
| Supabase Realtime | WebSocket push for live chat updates |

---

## Service Overview

| Service | Framework | Port | Database | Purpose |
|---------|-----------|------|----------|---------|
| `switcher-app-be` | Fastify 5 | 3000 | `switchit-db` | Candidate profiles, social feed, jobs, AI companion |
| `scout-backend` | Express 4 | 8080 | `scout-db` | HR authentication, candidate search, job management |
| `app-frontend-website` | React 18 + Vite | 5173 | — | Candidate-facing web UI |
| `infra-server` | Docker Compose | — | — | PostgreSQL, Redis, Elasticsearch |

---

## AI Companion Agent (Darshan)

### Overview

Darshan is a context-aware career mentorship agent embedded in the candidate platform. It uses a tool-augmented LLM inference loop to ground responses in the user's live profile data rather than relying solely on static knowledge.

**Personality**: Modelled as a 25-year-old tech recruiter — casual, direct, short responses (max 300 output tokens). The system prompt enforces tool-first behaviour: the agent must call a tool before answering any question that involves user-specific data.

### LLM Configuration

```
Provider:    Groq (OpenAI-compatible REST endpoint)
Model:       llama-3.3-70b-versatile
Endpoint:    https://api.groq.com/openai/v1
Max Tokens:  300
Auth:        GROQ_API_KEY
```

### Inference Loop

Darshan follows a two-pass agentic loop:

```
User Message
     │
     ▼
[Pass 1] LLM call with tools defined
     │
     ├── No tool call → stream final response
     │
     └── Tool call returned
              │
              ▼
        Execute tool (DB query)
              │
              ▼
        Inject tool result into messages array
              │
              ▼
        [Pass 2] LLM call with tool result in context
              │
              ▼
        Stream final response
```

### Tool Definitions

All tools are defined in `switcher-app-be/controller/companionController.js` and resolved against live PostgreSQL data.

| Tool | Description | Data Sources |
|------|-------------|--------------|
| `get_user_profile` | Fetches user's full career snapshot — headline, bio, industry, seniority, skills, experience, education | `profiles`, `experience`, `education` tables in `switchit-db` |
| `get_user_applied_jobs` | Returns the last 10 job applications with statuses | `applications` (switchit-db) JOIN `jobs`, `companies` (scout-db) — cross-database join at the application layer |
| `get_user_feed` | Retrieves top 5 recommended posts from the social feed | `Post.findRecommended(userId)` ranking algorithm |

### Conversation Memory

- Conversation history stored in `companion.conversations` and `companion.messages` (PostgreSQL, isolated schema).
- Last 20 messages passed as context on each inference call.
- Users can issue a hard reset (`DELETE /api/companion/clear`), which archives the active session and creates a new one.
- Multi-part replies are delimited by `||FOLLOWUP||`, split client-side into separate chat bubbles.

### Context Caching

On each new session, Darshan pre-fetches and caches the user's extracted career profile into `companion.user_context_cache`. This snapshot (name, headline, industry, role, experience range, location, skills summary) is embedded directly into the system prompt, avoiding repeated joins on the main profile tables per chat turn.

### Proactive Messaging (Companion Cron)

`companionCron.js` registers a daily cron trigger at 10:00 UTC. On each run it:
1. Queries all users with status `open_to_opportunities`
2. For each user, calls Groq to generate a personalised check-in message using their cached profile
3. Persists the message to the conversation store marked `proactive: true`
4. The frontend surfaces these as unsolicited messages with a distinct visual treatment

---

## RAG Pipeline

### Architecture

```
Knowledge Documents (.txt / .md)
         │
         ▼ ingest-rag.js
  Paragraph-level Chunking (~500 tokens)
         │
         ▼ OpenAI text-embedding-3-small
  1536-dimensional dense vectors
         │
         ▼ pgvector INSERT
  companion.knowledge_base (PostgreSQL)
         │
         │          At query time:
         ▼
  Embed user query → cosine similarity search via HNSW index
         │
         ▼ top-5 chunks (similarity threshold ≥ 0.5)
  Injected as context into LLM prompt
```

### Vector Store

RAG storage uses the `pgvector` PostgreSQL extension with HNSW indexing for approximate nearest-neighbour (ANN) retrieval.

**Schema — `companion.knowledge_base`**:

```sql
CREATE TABLE companion.knowledge_base (
    id           SERIAL PRIMARY KEY,
    title        TEXT,
    content      TEXT,
    chunk_index  INTEGER,
    embedding    vector(1536),              -- OpenAI text-embedding-3-small
    metadata     JSONB,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_companion_kb_embedding
    ON companion.knowledge_base
    USING hnsw (embedding vector_cosine_ops);  -- HNSW for ANN retrieval
```

**Why HNSW?** HNSW (Hierarchical Navigable Small World) provides sub-linear query time for ANN search, making it significantly faster than brute-force exact search at the cost of additional index build time and memory — an acceptable trade-off for a read-heavy, low-write knowledge base.

### Embedding Model

| Parameter | Value |
|-----------|-------|
| Model | `text-embedding-3-small` |
| Dimensions | 1536 |
| Similarity metric | Cosine similarity via pgvector `<=>` operator |
| Provider | OpenAI API |

### Document Ingestion

Run via `node scripts/ingest-rag.js`. The pipeline:

1. Scans `switcher-app-be/knowledge_base/` for `.txt` and `.md` files
2. Splits each file into paragraph-level chunks (loose chunking — not token-counted)
3. Calls OpenAI embeddings API per chunk
4. Upserts `(title, content, chunk_index, embedding, metadata)` into `companion.knowledge_base`

**Current knowledge documents**:
- `interview_guide.txt` — Interview preparation strategies
- `resume_tips.txt` — Resume optimisation techniques

### Retrieval

`searchKnowledgeBase(queryEmbedding, threshold = 0.5)` returns up to 5 chunks ordered by cosine similarity using the `<=>` distance operator:

```sql
SELECT content, 1 - (embedding <=> $1::vector) AS similarity
FROM companion.knowledge_base
WHERE 1 - (embedding <=> $1::vector) >= $2
ORDER BY similarity DESC
LIMIT 5;
```

**Integration status**: The retrieval function and knowledge base schema are production-ready. The `knowledge_search` tool definition exists in the codebase but is currently commented out of the active tool roster pending prompt engineering for context injection into the companion's response generation.

---

## Candidate Platform — switcher-app-be

**Framework**: Fastify 5  
**Port**: 3000  
**Auth**: JWT issued as HTTP-only cookie; enforced via `authenticateToken` middleware  
**File uploads**: `@fastify/multipart` → AWS S3  

### API Routes

#### Authentication `/api/auth`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/signup` | Register with email/password, trigger OTP verification email |
| POST | `/verify-otp` | Confirm email OTP to activate account |
| POST | `/login` | Credential login, set JWT HTTP-only cookie |
| POST | `/logout` | Clear JWT cookie |
| POST | `/google` | Google OAuth token exchange |
| POST | `/forgot-password` | Initiate password reset flow |
| POST | `/reset-password` | Consume reset token, update credentials |
| GET | `/validate-token` | Verify active session + return profile completion state |

#### Profile `/api/profile`

| Method | Path | Description |
|--------|------|-------------|
| PUT | `/` | Update profile (multipart: picture, full_name, employment status, etc.) |
| DELETE | `/` | Delete account |
| POST | `/send-otp` | Send OTP to company email for workplace verification |
| POST | `/verify-otp` | Verify company email OTP |

Career timeline follows the same CRUD pattern for `/api/experience` and `/api/education`.

#### Social Feed `/api/posts`

Paginated infinite-scroll feed with recommendation ranking via `Post.findRecommended(userId)`. Supports text, image, video, and poll post types with likes and threaded comments.

#### Jobs & Applications

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/jobs` | Fetch available listings |
| GET | `/api/jobs/:job_uuid` | Job detail view |
| POST | `/api/applications/apply` | Submit application |
| GET | `/api/applications` | User's application history |

#### AI Companion `/api/companion`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/chat` | Send user message, execute tool calls, return LLM response |
| GET | `/history` | Fetch last 50 messages in active session |
| DELETE | `/clear` | Archive current session, create new conversation |

#### Resume Parsing `/api/resume-parse`

Accepts PDF upload, converts to base64, submits to OpenAI Vision API, returns structured JSON (name, skills, experience, education). Used in the candidate onboarding flow to pre-populate profile fields.

---

## Recruiter Platform — scout-backend

**Framework**: Express 4  
**Port**: 8080  
**Auth**: Session-based HR authentication via `hrAuthMiddleware`  
**Search**: Elasticsearch primary with PostgreSQL fallback  

### Candidate Search `/api/search/candidates`

The core recruiter feature. Accepts a rich filter set and constructs an Elasticsearch bool query:

```
must:     fuzzy multi-field text match (searchText across role, skills, bio)
filter:   term/range filters (industry, roleType, ctcRange, noticePeriod, etc.)
must_not: match current_company to exclude recruiter's own employees
```

Supported filter parameters: `searchText`, `industry`, `roleType`, `experienceRange`, `ctcRange`, `noticePeriod`, `employmentType`, `currentLocation`, `companies`, `skills`, `gender`.

Results are ranked by Elasticsearch relevance score and paginated via `page` / `pageSize`. If Elasticsearch is unavailable the service falls back to equivalent parameterised PostgreSQL queries without surfacing the degradation to the client.

### Job Description Parser

`POST /api/recruiter/jd-parse` accepts raw job description text and calls OpenAI to extract structured fields (title, required skills, experience range, salary band, role type). Used by HR to auto-populate job draft forms from unstructured JD copy.

### Agent Booking `/api/bookAgent`

Manages scheduling of candidate interviews via SwitchIT interview agents. Exposes endpoints for listing available agents, creating bookings, querying scheduled interviews, and managing the interview queue.

---

## Frontend — app-frontend-website

**Framework**: React 18 + TypeScript  
**Build tool**: Vite 5  
**UI**: shadcn/ui + Radix UI primitives + Tailwind CSS  
**State**: Redux Toolkit + Redux Persist (auth), React Query (server-side cache)  
**Routing**: React Router v6  

### Key Pages

| Route | Page |
|-------|------|
| `/` | Home / Jobs feed |
| `/feed` | Social feed |
| `/profile` | User profile |
| `/complete-profile` | Onboarding wizard |
| `/opportunities` | Job listings |
| `/messaging` | Direct messaging |
| `/insights` | Analytics dashboard |
| `/coins`, `/cpoints` | Gamification (wallet, coin transactions) |
| `/learning` | Educational content |

### AI Companion Chat Widget

`AICompanionChat.tsx` renders as an expandable floating widget pinned to the main layout.

- Subscribes to Supabase Realtime for live message delivery (WebSocket)
- Splits `||FOLLOWUP||` delimited responses into individual chat bubbles for a natural conversational feel
- Renders inline markdown (bold text, line breaks) in message content
- Typing indicator with 30-second auto-dismiss timeout
- Unread badge counter persisted in local state
- Full-screen expand mode for mobile viewports

### Auth Architecture

- JWT stored in HTTP-only cookie (XSS-resistant; not accessible via `document.cookie`)
- `ProtectedRoute` / `PublicRoute` higher-order components enforce authenticated and unauthenticated route guards
- `AuthContext` exposes session state and decoded user metadata
- Google OAuth via `@react-oauth/google`

---

## Infrastructure — infra-server

All backing services are defined in a single Docker Compose file with environment-specific overrides (`docker-compose.local.yml`, `docker-compose.staging.yml`).

### Services

#### PostgreSQL 15 + pgvector

- Hosts both `scout-db` and `switchit-db` in a single instance
- `pgvector` extension pre-installed and enabled for vector column support
- Initialisation scripts in `init-scripts/`:
  - `01-create-databases.sh` — Creates both databases
  - `02-init-scout-db.sql` — HR schema (companies, jobs, skills, agents, applications)
  - `03-init-switchit-db.sql` — Candidate schema + `companion` schema with `knowledge_base`
- Healthcheck via `pg_isready`

#### Redis 7

- OTP caching with TTL (keys namespaced `scout:otp:*`, `switchit:otp:*`)
- Session token cache
- Rate limit counters consumed by Fastify `@fastify/rate-limit`

#### Elasticsearch 9.2.4

- Single-node cluster, no TLS (`xpack.security.enabled: false`)
- Index `profiles1`: candidate profile documents synced from `switchit-db`
- Multi-field fuzzy search with `AUTO` edit distance fuzziness plus range and term filters

### Operational Scripts

```bash
./scripts/start.sh       # Start all containers, block until healthchecks pass
./scripts/stop.sh        # Stop containers (preserve volumes)
./scripts/cleanup.sh     # Destroy containers, volumes, and networks
./scripts/verify.sh      # Run health checks against all three services
./scripts/deploy-aws.sh  # SSH deploy to EC2 instance
```

---

## Databases

### switchit-db (Candidate platform)

| Table | Purpose |
|-------|---------|
| `users` | Auth accounts (email, provider, verification status) |
| `profiles` | Career profile (headline, bio, industry, salary expectations, employment status) |
| `experience` | Work history entries |
| `education` | Academic history entries |
| `posts` | Social feed entries (text, image, video, poll) |
| `comments`, `likes`, `follows` | Social graph interactions |
| `coins` (wallet, transactions, streaks) | Gamification system |
| `notifications` | Email/SMS notification log |
| `initiative_waitlist` | Early access signups (yea, 90club, freelancer initiatives) |

#### Companion Schema (`companion.*`)

| Table | Purpose |
|-------|---------|
| `companion.conversations` | Chat session records (active / archived) |
| `companion.messages` | Full message history per session |
| `companion.user_context_cache` | Pre-extracted profile snapshots for LLM system prompt injection |
| `companion.knowledge_base` | RAG document chunks + 1536-dim vector embeddings |

### scout-db (Recruiter platform)

| Table | Purpose |
|-------|---------|
| `companies` | Company profiles |
| `jobs` | Job postings with structured JD fields |
| `skills`, `industries`, `locations`, `job_roles` | Reference / taxonomy tables |
| `agents` | SwitchIT human interview agents |
| `applications` | Job applications (FK to `switchit-db.users` resolved at the application layer) |
| `saved_lists` | Recruiter candidate shortlists |

### Redis

| Key Pattern | Usage |
|-------------|-------|
| `scout:otp:<email>` | HR OTP with TTL |
| `switchit:otp:<email>` | Candidate OTP with TTL |
| Rate limiter keys | Per-IP counters for API rate limiting |

### Elasticsearch Index: `profiles1`

Documents represent candidate profiles synced from `switchit-db`. Indexed fields: `current_job_role`, `current_location`, `industry`, `skills[]`, `experience_years`, `ctc`, `employment_status`, `gender`. Queried via multi-field fuzzy full-text search combined with bool filter clauses.

---

## Key Architectural Patterns

### 1. Tool-Augmented LLM Inference (Agentic Loop)

Rather than stuffing all user data into a static system prompt, Darshan uses OpenAI-compatible function calling. The LLM decides at inference time which tools to invoke, receives structured JSON results injected into the messages array, and then generates a data-grounded response. This enables dynamic retrieval on demand without inflating the context window with data the model may not need.

### 2. Cross-Database Joins at the Application Layer

`scout-db` and `switchit-db` are physically separate PostgreSQL databases on the same host. Queries spanning both databases (e.g., fetching a candidate's applications with job and company details) are resolved as two independent parameterised queries and merged in the service layer. This decouples the two platforms and allows independent schema evolution or future sharding.

### 3. Elasticsearch + PostgreSQL Fallback Search

The candidate search service constructs and executes an Elasticsearch bool query first. If the ES node is unavailable, the identical filter logic is re-executed as a parameterised PostgreSQL query. Both code paths return the same response schema, making Elasticsearch an optional performance enhancement rather than a hard runtime dependency.

### 4. pgvector HNSW for Vector Similarity Search

Embedding retrieval uses HNSW indexing (`vector_cosine_ops`) within PostgreSQL via the `pgvector` extension, eliminating a separate vector database dependency. The `<=>` cosine distance operator enables approximate nearest-neighbour lookups against the 1536-dimensional embedding space with sub-linear query time.

### 5. Proactive Agent Engagement via Scheduled Cron

The companion agent is not purely reactive. A server-side cron job identifies candidates flagged `open_to_opportunities`, generates personalised check-in messages via the Groq LLM, and persists them to the message store marked `proactive: true`. The frontend surfaces these without any user-initiated trigger, increasing session re-engagement.

### 6. Context Caching for LLM Personalisation

Rather than executing multi-table joins on every chat turn to construct the system prompt, Darshan materialises a concise user context snapshot into `companion.user_context_cache` at session start. This cache is invalidated on profile update, ensuring the LLM always has fresh but efficiently retrieved personalisation data without per-turn DB overhead.

### 7. Paragraph-Level Chunking for RAG Ingestion

Documents are chunked at paragraph boundaries rather than by fixed token count. This preserves semantic coherence per chunk at the cost of variable chunk sizes. The trade-off is intentional: for a small, high-quality knowledge base, semantic fidelity per chunk matters more than uniform density, and the HNSW index handles variable-length retrieval efficiently.

### 8. Supabase Realtime for Chat Push Delivery

Rather than polling or maintaining a persistent WebSocket server, the frontend subscribes to Supabase Realtime channels for companion message delivery. This offloads connection lifecycle management and horizontal scaling of WebSocket connections to an external service, allowing the Fastify backend to remain stateless.

### 9. Dual-Platform Role-Based Authentication

HR routes are protected by `hrAuthMiddleware` which validates sessions against `scout-db`. Candidate routes use `authenticateToken` which validates JWTs against `switchit-db`. The two auth contexts are fully isolated — an HR session cannot be used to access candidate-scoped endpoints and vice versa.

### 10. Resume Parsing via Multimodal Vision API

PDF resumes are converted to base64-encoded images and submitted to OpenAI's Vision API rather than using PDF parsing libraries. This approach handles scanned documents, non-standard layouts, and mixed-format PDFs without fragile text extraction heuristics, returning structured JSON directly usable for profile pre-population.

---

## Environment Variables

### switcher-app-be

```env
PORT=3000
DB_HOST=local-postgres
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=
DB_NAME=switchit-db
JWT_SECRET=
GROQ_API_KEY=                  # Companion LLM inference (Groq)
OPENAI_API_KEY=                # RAG embeddings + resume parsing (OpenAI)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_BUCKET_NAME=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
EMAIL_USER=
EMAIL_PASS=
```

### scout-backend

```env
PORT=8080
DB_HOST=local-postgres
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=
DB_NAME=scout-db
JWT_SECRET=
ELASTIC_ENDPOINT=http://local-elasticsearch:9200
REDIS_URL=redis://local-redis:6379
EMAIL_USER=
EMAIL_PASS=
```

### app-frontend-website

```env
VITE_BASE_URL=http://localhost:3000/api
VITE_SCOUT_BASE_URL=http://localhost:8080/api
VITE_GOOGLE_CLIENT_ID=
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
```
