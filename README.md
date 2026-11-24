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













/**
 * ============================================================================
 * ENTERPRISE LLM API GATEWAY & SEMANTIC CACHE — ARCHITECTURE BLUEPRINT
 * ============================================================================
 *
 * 1. STEP-BY-STEP REQUEST FLOW
 * ----------------------------------------------------------------------------
 * [1] Interception & Auth:
 *     - Express receives HTTP POST (/v1/chat/completions).
 *     - Middleware validates JSON payload & extracts `Authorization: Bearer <key>`.
 *     - Invalid requests fail fast (400/401) before touching any database.
 *
 * [2] Pre-Flight Tokenization:
 *     - Runs raw prompt through Byte Pair Encoding (BPE) tokenizer locally.
 *     - Computes exact input token count before any upstream network call.
 *
 * [3] Distributed Rate Limiting:
 *     - Redis checks Requests Per Minute (RPM) & Tokens Per Minute (TPM) quotas.
 *     - Rejects over-quota traffic with HTTP 429 Too Many Requests in < 5ms.
 *
 * [4] Semantic Cache Lookup:
 *     - Converts prompt into a dense vector embedding & queries Qdrant.
 *     - CACHE HIT  (Cosine Similarity >= 0.92): Bypasses upstream LLM entirely;
 *       returns cached answer in < 50ms at $0.00 cost.
 *     - CACHE MISS (Cosine Similarity <  0.92): Forwards request to upstream LLM
 *       (e.g., LLaMA-3 via Groq or Ollama).
 *
 * [5] Real-Time Streaming (SSE):
 *     - Opens persistent `text/event-stream` connection on cache miss.
 *     - Pipes generated token chunks directly to client as they arrive.
 *
 * [6] Post-Response Hydration & Async Logging:
 *     - Counts completion tokens & calculates exact USD cost (or savings).
 *     - Stores new prompt embedding + response in Qdrant for future hits.
 *     - Fires non-blocking async write to MySQL so client latency is unaffected.
 *
 * ============================================================================
 * 2. CORE SUBSYSTEMS
 * ============================================================================
 *
 * [A] Authentication & Modular Express Architecture:
 *     - Strict separation of concerns: Routes -> Middleware -> Controllers -> Services.
 *     - Gatekeeper Middleware: Early-return validation (400/401) protects downstream layers.
 *     - Singleton Connections: `mysql2` pool, `ioredis`, and Qdrant client initialized
 *       once in `src/config/` and shared across the application lifecycle.
 *
 * [B] BPE Tokenizer & Dynamic Pricing Engine (`tokenizerService.js`):
 *     - Exact Accounting: Uses `js-tiktoken` to encode prompts into token IDs and
 *       accumulates streamed chunks to count completion tokens.
 *     - Pricing Formula:
 *       Total Cost = ((Input Tokens / 1,000,000) * P_in) + ((Output Tokens / 1,000,000) * P_out)
 *     - Cost-Saved Tracking: Logs hypothetical upstream cost as `cost_saved_usd` on cache hits.
 *
 * [C] Distributed Rate Limiter (`rateLimiter.js` + Redis):
 *     - Algorithm: Sliding Window Log using Redis Sorted Sets (`ZADD`, `ZREMRANGEBYSCORE`,
 *       `ZCARD`) or atomic Lua scripts to prevent boundary traffic spikes.
 *     - Dual-Dimension Quotas: Enforces both RPM (e.g., 20 req/min) and TPM
 *       (e.g., 10,000 tokens/min), returning `Retry-After` headers on HTTP 429.
 *
 * [D] Semantic Vector Cache (`cacheService.js` + Qdrant):
 *     - Vector Embeddings: Converts natural language prompts into high-dimensional vectors.
 *     - ANN Search: Qdrant uses HNSW graphs for sub-50ms vector retrieval.
 *     - Cosine Similarity: (A . B) / (||A|| * ||B||) with strict >= 0.92 threshold
 *       to prevent false positives between distinct queries.
 *     - TTL & Expiration: Metadata timestamps invalidate stale cached responses.
 *
 * [E] Upstream LLM Proxy & Server-Sent Events (`llmService.js`):
 *     - Headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`,
 *       `Connection: keep-alive`.
 *     - Stream Teeing: Simultaneously writes chunks to client HTTP response AND
 *       appends to an in-memory buffer for token counting & Qdrant hydration on `[DONE]`.
 *
 * [F] MySQL Telemetry & Analytics (`loggingService.js` + Dashboard):
 *     - Table `api_keys`:     id, user_name, key_hash, rpm_limit, tpm_limit, is_active, created_at
 *     - Table `request_logs`: id, api_key_id, model, prompt_tokens, completion_tokens,
 *                             cost_usd, cost_saved_usd, cache_hit, latency_ms, status_code, created_at
 *     - Endpoint `GET /api/analytics/summary`: SQL aggregations (`SUM`, `COUNT`, `AVG`,
 *       `GROUP BY`) powering React/Chart.js metrics (Cache Hit %, USD Saved, Latency).
 *
 * ============================================================================
 * 3. TECHNOLOGY STACK
 * ============================================================================
 * - Runtime & Framework : Node.js + Express.js (non-blocking event loop for SSE)
 * - Rate Limiting Store : Redis / `ioredis` (sub-ms atomic sliding-window state)
 * - Semantic Cache      : Qdrant (fast HNSW Cosine Similarity vector search)
 * - Relational Storage  : MySQL / `mysql2` (ACID key management & SQL aggregations)
 * - Tokenizer           : `js-tiktoken` (local BPE encoding with zero network overhead)
 * - Upstream LLM        : Groq API (LLaMA-3) / Ollama ($0 inference for dev/demo)
 * - Local Orchestration : Docker Compose (identical MySQL, Redis, Qdrant containers)
 *
 * ============================================================================
 * 4. RESILIENCE & FAULT TOLERANCE
 * ============================================================================
 * - Fail-Open Caching: If Qdrant crashes or times out (> 200ms), catch error and
 *   fall back to a normal LLM cache miss so users never see downtime.
 * - Non-Blocking Telemetry: MySQL writes run asynchronously after response dispatch.
 * - Client Disconnect Handling: Listen to `req.on('close')` and trigger `AbortController`
 *   to terminate upstream LLM generation immediately and avoid wasted token spend.
 * - Cache Isolation: Filter Qdrant queries by metadata (`model`, `tenant_id`) so
 *   cached responses never leak across models or tenants.
 *
 * ============================================================================
 * 5. RESUME SUMMARY
 * ============================================================================
 * Enterprise LLM API Gateway & Semantic Cache | Node.js, Express, Redis, Qdrant, MySQL, Docker
 * - Engineered an OpenAI-compatible HTTP proxy in Node.js with real-time SSE streaming
 *   and non-blocking async telemetry.
 * - Built a Qdrant Semantic Vector Cache (Cosine Similarity >= 0.92) serving equivalent
 *   prompts in < 50ms, slashing inference costs and latency.
 * - Implemented a distributed Redis Sliding-Window Rate Limiter enforcing dual RPM
 *   and BPE Token-Per-Minute (TPM) quotas.
 * - Designed a MySQL analytics pipeline and dashboard tracking cache hit ratios,
 *   per-key token usage, and real-time cost savings.
 * ============================================================================
 */
