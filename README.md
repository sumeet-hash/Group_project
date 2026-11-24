# Group_project
This git hub is dedicated to our group project
# Enterprise LLM API Gateway & Semantic Cache

A high-concurrency, OpenAI-compatible HTTP proxy server built with **Node.js** and **Express**. It sits between client applications and upstream Large Language Model (LLM) providers to reduce API inference costs, lower response latency, enforce token-aware rate limits, and track real-time usage analytics.

---

## 🎯 Problem Statement

Organizations integrating LLMs into production face four core infrastructure challenges:
1. **Uncontrolled API Costs:** Standard exact-match caches miss when users ask semantically identical questions phrased differently (e.g., *"How do I reset my password?"* vs. *"I forgot my password, how do I change it?"*), forcing redundant LLM calls.
2. **High Latency:** Full LLM generations take 2–10 seconds unless streamed token-by-token or served from a sub-50ms cache.
3. **Quota Exhaustion:** Standard HTTP rate limiters count *requests* rather than *tokens*, allowing a single massive prompt to drain an API budget.
4. **Lack of Observability:** Direct client-to-LLM calls provide no centralized visibility into per-user token spend, latency, or cache efficiency.

This gateway solves all four problems behind a drop-in `POST /v1/chat/completions` endpoint.

---

## 🏗️ System Architecture & Request Lifecycle

Every incoming request passes through a non-blocking, fail-fast pipeline:

```text
  [ Client / Postman / Dashboard ]
                 │
                 ▼  POST /v1/chat/completions (Authorization: Bearer <API_KEY>)
┌────────────────────────────────────────────────────────────────────────────┐
│                     NODE.JS / EXPRESS API GATEWAY                          │
│                                                                            │
│  1. Auth & Validation Middleware ──► Rejects invalid keys/payloads (401)   │
│                 │                                                          │
│  2. Tokenizer & Cost Engine ───────► Counts exact BPE input tokens         │
│                 │                                                          │
│  3. Redis Rate Limiter ────────────► Checks RPM & TPM limits (429 if over) │
│                 │                                                          │
│  4. Semantic Vector Cache Check ───► Embeds prompt ──► Queries Qdrant      │
│          │                  │                                              │
│     [CACHE HIT]        [CACHE MISS]                                        │
│   (Similarity ≥ 0.92)       │                                              │
│          │                  ▼                                              │
│          │         5. Upstream LLM Proxy (Groq / Ollama)                   │
│          │            Streams chunks via Server-Sent Events (SSE)          │
│          │                  │                                              │
│          ▼                  ▼                                              │
│     Return Cached      Stream Live Tokens to Client                        │
│     Answer (<50ms)          │                                              │
│          │                  ▼                                              │
│          │         6. Cache Hydration (Save embedding + answer to Qdrant)  │
│          └──────────────────┴──────────────────────────────────────┐       │
└────────────────────────────────────────────────────────────────────┼───────┘
                                                                     ▼
                                                      7. Async Telemetry Logger
                                                         Writes metrics to MySQL








📁 Project Structure
Plaintext
llm-api-gateway/
├── docker-compose.yml          # Spins up MySQL, Redis, and Qdrant locally
├── .env.example                # Environment variable template
├── package.json
└── src/
    ├── server.js               # Express entry point
    ├── config/                 # Singleton DB connections (mysql.js, redis.js, qdrant.js)
    ├── routes/                 # API route definitions (gatewayRoutes.js, analyticsRoutes.js)
    ├── middleware/             # API key auth & Redis rate limiter
    ├── controllers/            # Request pipeline orchestration
    └── services/               # Modular core business logic
        ├── tokenizerService.js # BPE token counting & USD cost estimation
        ├── cacheService.js     # Vector embedding & Qdrant similarity search
        ├── llmService.js       # Upstream LLM proxy & SSE stream handler
        └── loggingService.js   # Non-blocking MySQL telemetry writer













Step-by-Step Request FlowInterception & Auth: Express receives the HTTP POST request. Middleware validates the JSON payload and extracts the API key from the Authorization: Bearer <key> header. Invalid requests are rejected immediately before touching any database.Pre-Flight Tokenization: The gateway runs the raw prompt through a Byte Pair Encoding (BPE) tokenizer to compute the exact input token count before any network call is made.Distributed Rate Limiting: Redis checks whether the API key has exceeded its Requests Per Minute (RPM) or Tokens Per Minute (TPM) budget. If exceeded, the gateway returns HTTP 429 Too Many Requests in under 5 milliseconds.Semantic Cache Lookup: The prompt is converted into a dense numerical vector (embedding) and queried against the Qdrant vector database.On a Cache Hit: If a previously answered prompt has a Cosine Similarity score $\ge 0.92$, the gateway bypasses the upstream LLM entirely and returns the cached response in $<50\text{ms}$ at $\$0.00$ cost.On a Cache Miss: The gateway forwards the request to the upstream LLM (e.g., LLaMA-3 via Groq or Ollama).Real-Time Streaming (SSE): Instead of buffering the entire AI response, the gateway opens a Server-Sent Events (text/event-stream) connection and pipes each generated token chunk directly to the client as it arrives.Post-Response Hydration & Async Logging: Once the stream closes, the gateway counts the output tokens, computes the exact USD cost (or savings), stores the new prompt-response pair in Qdrant for future semantic hits, and fires an asynchronous, non-blocking write to MySQL so the client response is never delayed.3. Deep Dive into Core SubsystemsA. Authentication & Modular Express ArchitectureThe server strictly separates concerns into Routes, Middleware, Controllers, and Services to keep the codebase maintainable and conflict-free for parallel team development.Gatekeeper Middleware: Request validation and API key verification happen at the middleware layer using early-return patterns (400 Bad Request / 401 Unauthorized), protecting downstream services from malformed payloads.Singleton Connections: Database clients (mysql2 connection pools, ioredis, and Qdrant clients) are initialized once inside src/config/ and shared across the entire application lifecycle rather than opening new TCP connections per request.B. BPE Tokenizer & Dynamic Pricing EngineLLMs do not read raw characters or whole words; they process sub-word chunks called tokens using Byte Pair Encoding (BPE).Exact Accounting: Using js-tiktoken, tokenizerService.js encodes incoming prompts into token IDs to determine exact input length. For streamed responses, it accumulates the output chunks to calculate exact completion tokens.Pricing Matrix: A model-specific pricing table maps token counts to USD costs:$$\text{Total Cost} = \left(\frac{\text{Input Tokens}}{1,000,000} \times P_{\text{in}}\right) + \left(\frac{\text{Output Tokens}}{1,000,000} \times P_{\text{out}}\right)$$Cost-Saved Tracking: On a semantic cache hit, the engine still calculates what the request would have cost upstream and records that figure as cost_saved_usd.C. Distributed Rate Limiter (Redis)Standard in-memory JavaScript objects fail as a rate limiter if you run multiple server instances. Storing rate-limit state in Redis makes the gateway horizontally scalable.Algorithm (Sliding Window Log / Token Bucket): Instead of naive fixed-window counters (which allow traffic spikes at minute boundaries), the gateway uses Redis Sorted Sets (ZADD, ZREMRANGEBYSCORE, ZCARD) or atomic Lua scripts to enforce smooth rolling windows.Dual-Dimension Limiting: Enforces both Request Limits (e.g., 20 requests/min) and Token Quotas (e.g., 10,000 tokens/min), returning standard Retry-After headers when triggered.D. Semantic Vector Cache (Qdrant / ChromaDB)Traditional key-value caches like Redis require 100% character-for-character string equality. Natural language almost never repeats identically.Vector Embeddings: cacheService.js converts incoming prompts into high-dimensional floating-point vectors that capture semantic meaning.Approximate Nearest Neighbor (ANN) Search: Qdrant indexes these vectors using HNSW (Hierarchical Navigable Small World) graphs to search millions of past prompts in milliseconds.Cosine Similarity Thresholding: The similarity between the incoming prompt vector $\mathbf{A}$ and a cached vector $\mathbf{B}$ is measured via:$$\text{Cosine Similarity} = \frac{\mathbf{A} \cdot \mathbf{B}}{\Vert{}\mathbf{A}\Vert{} \Vert{}\mathbf{B}\Vert{}}$$A strict threshold (e.g., 0.92–0.95) ensures the cache only fires when two questions genuinely mean the same thing, avoiding false positives (e.g., "Write code in Python" vs. "Write code in Java").TTL & Expiration: Cached entries carry metadata timestamps so stale answers can be expired or invalidated automatically.E. Upstream LLM Proxy & Server-Sent Events (SSE)Standard REST APIs force the client to wait in silence until the entire JSON payload is ready.Persistent HTTP Streaming: When forwarding a cache miss, the gateway sets Content-Type: text/event-stream, Cache-Control: no-cache, and Connection: keep-alive.Teeing the Stream: As chunks arrive from Groq or Ollama, Node.js simultaneously writes each chunk to the client HTTP response and appends it to a local buffer. Once the upstream provider emits [DONE], the gateway uses the buffered string to count output tokens and populate the semantic cache—all without blocking the Node.js event loop.F. MySQL Telemetry & Analytics DashboardEvery request—whether a cache hit, cache miss, or rate-limited rejection—is logged asynchronously to MySQL for business intelligence.Core Relational Tables:api_keys: Stores id, user_name, key_hash, rpm_limit, tpm_limit, is_active, created_at.request_logs: Stores id, api_key_id, model, prompt_tokens, completion_tokens, cost_usd, cost_saved_usd, cache_hit (BOOLEAN), latency_ms, status_code, created_at.Analytics API & UI: Exposes GET /api/analytics/summary using SQL aggregations (SUM, COUNT, AVG, GROUP BY) to power a clean React/Chart.js dashboard displaying:Cache Hit Ratio (%)Total USD Spent vs. Total USD SavedAverage Latency (Cache Hit vs. Cache Miss)Top Consumers & Rate-Limit Violations4. Technology Stack & Engineering JustificationLayerTechnologyWhy This Choice?Runtime & FrameworkNode.js + Express.jsSingle-threaded, event-driven non-blocking I/O excels at proxying concurrent HTTP streams and SSE connections without thread overhead.Rate Limiting StoreRedis (ioredis)Sub-millisecond in-memory reads/writes with native TTL and atomic operations for distributed sliding-window enforcement.Semantic CacheQdrant (or ChromaDB)Purpose-built vector database supporting fast HNSW Cosine Similarity search over high-dimensional prompt embeddings.Relational AnalyticsMySQL (mysql2)ACID-compliant structured storage optimized for relational API key management and SQL financial aggregations.Tokenizationjs-tiktokenFast, local Byte Pair Encoding (BPE) implementation requiring zero external API calls to count tokens.Upstream LLMGroq API (LLaMA-3) / OllamaFree-tier / local open-weights inference so the team can develop, load-test, and demo the entire system at $0 cost.Local InfrastructureDocker ComposeGuarantees identical MySQL, Redis, and Qdrant versions across every teammate's laptop with a single docker compose up command.5. Resilience, Fault Tolerance & Engineering Edge CasesTo elevate this from a college project to a production-grade backend system, the architecture explicitly handles failure modes:Fail-Open Caching (Graceful Degradation): If Qdrant crashes or times out (>200ms), the gateway catches the error, logs a warning, and treats the request as a normal cache miss—forwarding it to the LLM so the user never experiences downtime.Non-Blocking Telemetry: Writing to MySQL is decoupled from the HTTP response path. If MySQL experiences a brief spike in write latency, the client still receives their AI response immediately.Client Disconnect Handling: If a user closes their browser tab mid-stream, Express listens for the req.on('close') event and aborts the upstream AbortController signal to immediately stop generating (and paying for) unused LLM tokens.Cache Isolation: System prompts or model parameters (e.g., model: llama-3 vs. gpt-4) are included as metadata filters in Qdrant so a cached response from one model or tenant is never leaked to another.6. Resume & Interview ImpactWhen presenting this project on your resume or in a technical interview, frame it as AI Infrastructure & Distributed Backend Engineering:Enterprise LLM API Gateway & Semantic Cache | Node.js, Express, Redis, Qdrant, MySQL, DockerEngineered an OpenAI-compatible HTTP proxy server in Node.js supporting real-time Server-Sent Events (SSE) streaming and non-blocking async telemetry.Built a Semantic Vector Cache using Qdrant and Cosine Similarity ($\ge 0.92$) to serve semantically equivalent prompts in $<50\text{ms}$, reducing upstream LLM inference costs and latency.Implemented a distributed Redis Sliding-Window Rate Limiter enforcing dual Request-Per-Minute (RPM) and Byte-Pair-Encoding (BPE) Token-Per-Minute (TPM) quotas.Designed a MySQL analytics pipeline and dashboard tracking cache hit ratios, per-key token consumption, and real-time cost savings across containerized microservices.
