# Production Readiness Plan: Agentic Stock Research Assistant

This document outlines the engineering plan to transition the MVP Stock Research Assistant—an agentic AI application—into a highly robust, scalable, and secure production environment.

## Phase 1: Security & Architecture Hardening
Before exposing agentic capabilities to production traffic, the foundation must be secured against abuse and failures.

- [ ] **Strict Environment Validation**: Implement `zod` to validate all environment variables (e.g., `GEMINI_API_KEY`, `FINNHUB_API_KEY`, `PORT`, `NODE_ENV`) at startup.
- [ ] **Rate Limiting & Abuse Prevention**: 
  - Add IP-based rate limiting to the Express app (using `express-rate-limit`) to prevent API abuse that translates to costly Gemini and Google Search Grounding calls.
  - Implement circuit breakers for the `get_stock_price` route to prevent cascading failures if external APIs (Finnhub, Yahoo) go down.
- [ ] **CORS Configuration**: Restrict CORS on the Express server to only allow the authorized frontend domain.
- [ ] **Secrets Management**: Move secrets from local `.env` to a managed vault (e.g., Google Secret Manager, AWS Secrets Manager) in higher environments.

## Phase 2: Agentic Workflow & Reliability
LLMs are non-deterministic. Production agentic apps require strict guardrails, retries, and data validation.

- [ ] **Robust Output Validation**: Current `JSON.parse()` is risky. Implement `zod` to validate the Gemini output against the expected `ResearchReport` schema *before* rendering. If validation fails, trigger a self-correction retry or fallback.
- [ ] **Retry Mechanisms**: Add exponential backoff using a library like `p-retry` or `axios-retry` for both the custom API endpoints and the Gemini API boundaries to handle 429s (Too Many Requests) and 503s gracefully.
- [ ] **Prompt Engineering & Versioning**: Extract prompt strings from source code into a managed system or dedicated module. Maintain version control on prompts to test and rollback regressions cleanly.
- [ ] **Agent Observability**: Integrate an LLM observability tool (e.g., Langfuse, LangSmith, DataDog) to trace agent tool calls, measure token usage, track latency per step, and capture edge-case hallucinations.

## Phase 3: Financial Ops (FinOps) & Caching
Agentic search combined with API fetching can become expensive. Aggressive caching is necessary.

- [ ] **Redis Caching Layer**:
  - Cache stock data responses (`/api/stock/:ticker`) for ~15-60 seconds to reduce load on Finnhub/Yahoo logic.
  - Cache identical Gemini Search Grounding reports for ~5-15 minutes so identical popular ticker searches (e.g., TSLA, AAPL) serve from cache instantly, saving on token and grounding costs.
- [ ] **Token Optimization**: Review the JSON schema fed to Gemini to ensure it's as compact as possible. Minimize prompt length where feasible.
- [ ] **Cost Monitoring Alerts**: Set up budget alerts on the Google Cloud / Gemini platform to catch sudden spikes due to legitimate traffic scaling or DoS attacks.

## Phase 4: UX Polish & Error Handling
Agentic generation is asynchronous and multi-step. The UI must keep the user deeply informed.

- [ ] **Streaming Responses (Optional but Recommended)**: Instead of blocking until the entire 2-step process finishes, transition to a Server-Sent Events (SSE) or WebSocket integration. Show the user "Fetching Stock Data..." -> Render Stock Data -> "Searching News..." -> Render News.
- [ ] **Granular Edge Cases**:
  - Handle cases where grounding returns completely unrelated news simply because of ticker collisions.
  - Handle cases where the LLM decides *not* to call the tool because it thinks it already knows the answer (enforce tool calling config via `tool_choice: "any"` when required).
- [ ] **Accessibility (a11y)**: Ensure all Shadcn components, loading spinners, and dynamic DOM updates announce properly to screen readers (ARIA live regions). Add keyboard navigation.

## Phase 5: Testing & CI/CD Deployment
Testing agentic applications requires testing the determinism of non-deterministic systems.

- [ ] **Unit Testing (Code)**: Standard Jest/Vitest unit tests for the `/api/stock/:ticker` route, ensuring graceful fallback between Finnhub and Yahoo.
- [ ] **LLM Evaluation (Evals)**: Create a suite of test prompts (e.g., "AAPL", "INVALID_TICKER") and use an eval framework to automatically measure accuracy, hallucination rate, and JSON compliance.
- [ ] **End-to-End Testing (E2E)**: Implement Playwright or Cypress tests to walk through the entire UI flow, mocking the Gemini API response to ensure the frontend parses and renders the schema properly under various conditions.
- [ ] **Dockerization**: Create a multi-stage `Dockerfile` (build Vite frontend, copy to Express public folder, start Express).
- [ ] **CI/CD Pipeline**: Setup GitHub Actions / GitLab CI to run linting, unit tests, evals, and build before deploying automatically to Google Cloud Run or AWS ECS.
