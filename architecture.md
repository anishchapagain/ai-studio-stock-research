# Production Architecture & Development Requirements
**Project:** Agentic Stock Research Assistant

This document outlines the core product-specific development requirements and system architecture necessary to elevate the Agentic Stock Research Assistant from an MVP to a secure, enterprise-grade production application.

---

## 1. Core Development Requirements (Product-Specific)

### 1.1 Agentic Orchestration & Server-Side Execution
*   **Security & Credential Masking:** In a true production environment, **no AI API keys (e.g., Gemini) or financial data API keys (e.g., Finnhub) should ever touch the client.** The multi-step agentic workflow (Initial Prompt -> Tool Call -> Function Execution -> Search Grounding -> JSON Parsing) must be completely migrated to the backend Express server. The frontend should only make a single `GET /api/research/:ticker` request.
*   **Structured Outputs Constraints:** The LLM's non-deterministic nature means generating JSON can be failure-prone. The application must enforce strict `responseSchema` validation (using libraries like `Zod`) on the backend before delivering the payload to the frontend.

### 1.2 Data Resilience & API Fallbacks
*   **Graceful Degradation:** Financial APIs are prone to rate-limiting and downtime. The system must natively support fallback mechanisms (e.g., failing over to Yahoo Finance if Finnhub hits a 429 Limit Exceeded status).
*   **Timeout & Circuit Breakers:** Implement circuit breakers on external API boundaries to prevent cascading failures in the orchestration layer if third-party APIs become unresponsive.

### 1.3 State & Latency Management
*   **Asynchronous UX:** The dual-pass AI workflow (Stock resolution -> Google Grounded News generation) takes several seconds. The UI must utilize progressive loading states or Server-Sent Events (SSE) to stream intermediate updates (e.g., "Stock data fetched...", "Searching news...") to the user rather than leaving them waiting on a blank spinner.
*   **Query Sanitization:** Ticker symbols must be validated, sanitized, and explicitly restricted (e.g., alphabetic, exact length boundaries) to prevent prompt injection or broken tool calls.

---

## 2. Production System Architecture

A robust, enterprise-grade architecture for this agentic application separates concerns across distinct layers to optimize security, scalability, and cost/FinOps tracking.

### 2.1 The Client Layer (Presentation)
*   **Tech Stack:** React 19, Tailwind CSS, Shadcn UI.
*   **Responsibilities:** Captures input, performs client-side ticker format validation, and handles application state. Must render comprehensive error states (Network Error, Invalid Ticker, Rate Limits) gracefully.

### 2.2 The API Gateway & Security Layer
*   **Tech Stack:** Node.js, Express, `express-rate-limit`, `helmet`, `cors`.
*   **Responsibilities:** Inspects incoming connection requests. Enforces IP-based rate limiting (preventing Denial of Wallet attacks on the LLM layer). Validates User Auth tokens (if implemented) and routes traffic to the Orchestration API.

### 2.3 The Agent Orchestration Layer (Backend Core)
*   **Tech Stack:** Node.js, `@google/genai` (Server-side implementation).
*   **Responsibilities:** Acts as the brain of the application.
    1.  Receives the sanitized ticker symbol from the unified endpoint.
    2.  Builds the context and invokes the Gemini model, requesting the `get_stock_price` tool call.
    3.  Executes the physical `fetch` to financial APIs natively on the server.
    4.  Passes the response back into the context window and triggers the second Gemini call utilizing Google Search Grounding.
    5.  Receives the response, validates the final JSON structure, and returns it to the Client.

### 2.4 The Data & Caching Layer
*   **Tech Stack:** Redis (or Memcached/Upstash).
*   **Responsibilities:** Caching is critical for cost management (FinOps). 
    *   **Financial Data:** Cache standard API responses for 15-60 seconds.
    *   **LLM Responses:** Cache identical Research Reports for standard tickers (like `AAPL`) for up to 5-15 minutes using semantic or exact-match caching. If two users query `AAPL` within 5 minutes, the second user receives the cached report, entirely bypassing token costs and latency.

### 2.5 The Observability & FinOps Layer
*   **Tech Stack:** LangSmith / DataDog / Google Cloud Logging.
*   **Responsibilities:** Real-time monitoring of agent behavior.
    *   Tracking exact token usage per request.
    *   Tracing the duration of each individual LLM inference and tool execution.
    *   Tracking "hallucination" exceptions (e.g., when the LLM searches the news but hallucinates the JSON structure).
    
---
### Summary Diagram (Conceptual)
`[Client UI]` --(HTTP GET)--> `[Express Rate Limiter]` --> `[Agent Orchestrator]`
                                                                 |
                                 +-------------------------------+-----------------------------------+
                                 |                               |                                   |
                         `[Gemini 3.1 Pro]`            `[Finnhub / Yahoo API]`              `[Redis Cache]`
                         (Search Grounding)             (Real-time Stock Data)              (Cost Optimization)

---

## 3. Generic Approach: Advanced System Design for Agentic Apps

When designing AI-first, agent-driven applications at scale, engineers must treat LLMs as highly capable but fundamentally volatile network dependencies. The following advanced concepts form the generalized system design blueprint for these architectures:

### 3.1 Idempotency & Caching Strategies
*   **Semantic Caching:** Instead of strictly matching query strings (e.g., "AAPL news" vs "news for AAPL"), use an embedding-based vector cache (like RedisVL) to serve identical intent from cache. This drastically cuts LLM token costs and reduces latency to milliseconds.
*   **Tiered Caching:**
    *   *L1 (In-Memory)*: Transient app state (e.g., fast lookups, rate limit counters).
    *   *L2 (Distributed Cache)*: Redis for agent memory, HTTP responses, and synthesized LLM reports.
    *   *L3 (Cold Storage)*: Database for audit logs and historical analysis.

### 3.2 Resilience: Error Handling & Circuit Breakers
*   **Circuit Breaker Pattern:** Agentic workflows often sequence multiple APIs. If a downstream provider (e.g., an LLM or Data API) starts failing or taking 30s to respond, the circuit breaker "trips" and fast-fails new requests. This prevents the application from exhausting server threads while waiting on dead dependencies.
*   **Jittered Exponential Backoff (Time-Sleep):** When encountering a `429 Too Many Requests` or `5xx Server Error`, simple retries cause "thundering herd" problems. Implement retry loops with exponential backoff and randomized jitter (e.g., sleep 1s, then 2.3s, then 4.8s) to gracefully recover without overwhelming the revived service.
*   **Fallback Routing:** Multi-model routing is critical. If the primary model becomes unavailable, gracefully routing the request to a secondary provider (if configured) ensures high availability.

### 3.3 Rate Limiting, Throttling & Queueing
*   **Token-Bucket Architecture:** Implement sophisticated rate limiting that differentiates between authenticated tiers and anonymous traffic, safeguarding backend resources while ensuring fair usage.
*   **Asynchronous Queues:** For deep research tasks that take 10+ seconds, synchronous HTTP connections often timeout. Utilize a message broker (e.g., BullMQ, Kafka, RabbitMQ) for background processing. Workers independently pick up tasks, emitting progression state updates to clients via WebSockets or Server-Sent Events (SSE).

### 3.4 Observability & Deterministic Sandboxes
*   **Agent DAG Tracing:** Standard APM (Application Performance Monitoring) isn't enough. You must track the full Directed Acyclic Graph (DAG) of the agent's workflow: token usage per step, prompt text, tool input/output, and final synthesis.
*   **Guardrails & Output Sandboxing:** LLM responses must be treated as untrusted user input. Run the generated text through deterministic sandboxes (e.g., `Zod` parsing) and apply specialized guardrails (heuristic or smaller model checks) to intercept hallucinations, syntax failures, or policy violations before returning payload to the client.

