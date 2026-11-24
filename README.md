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
