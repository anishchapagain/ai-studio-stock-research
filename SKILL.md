---
name: stock-research-assistant
description: >
  Full-stack Stock Research Assistant application blueprint. Use this skill whenever the user wants
  to build, extend, or debug a stock research app that combines real-time stock data (via function
  calling / tool use) with AI-powered news grounding (via Google Search or web search tools).
  Trigger this skill for any request involving: stock ticker lookup UIs, real-time market data
  pipelines, AI Studio or Claude-powered financial dashboards, server-side stock API integration,
  function calling schemas for financial data, or news grounding for company research. Also trigger
  when the user mentions terms like "get_stock_price function", "stock UI", "ticker symbol input",
  "market data assistant", or "financial research app".
---

# Stock Research Assistant — Master Skill

A complete reference for building, extending, and debugging a production-grade Stock Research
Assistant. Covers system design, architecture decisions, agent/function-calling patterns, UI
contracts, API integration, environment setup, and testing strategy.

---

## 1. Application Overview

| Property        | Value                                                                 |
|-----------------|-----------------------------------------------------------------------|
| App type        | Full-stack web application (server + client)                          |
| Primary AI      | Claude (Anthropic API) or Gemini (Google AI Studio)                  |
| Function call   | `get_stock_price` — custom server-side tool                          |
| Grounding       | Google Search grounding / web search tool (recent news)              |
| Stock data      | Real external API (Alpha Vantage, Polygon.io, Finnhub, Yahoo Finance) |
| Auth pattern    | API keys stored server-side only, never exposed to frontend           |

### Core User Flow

```
User enters ticker → Submit → [Loading state]
  → Server: validate ticker
  → Tool call: get_stock_price(ticker)
  → External stock API → structured price data
  → AI: Google Search grounding → 3 recent news articles
  → Combine results → render in UI
  → [Error state if any step fails]
```

---

## 2. Architecture

### 2.1 High-Level Layers

```
┌─────────────────────────────────────────────┐
│                  FRONTEND                   │
│  Input Box → Submit → Loading → Results UI  │
│  (HTML/CSS/JS or React — no API keys here)  │
└──────────────────┬──────────────────────────┘
                   │ POST /api/research  { ticker }
┌──────────────────▼──────────────────────────┐
│               SERVER / API LAYER            │
│  1. Validate ticker (regex + length check)  │
│  2. Call AI with function declarations      │
│  3. Handle tool_use → get_stock_price()     │
│  4. Call external stock market API          │
│  5. Return tool_result to AI                │
│  6. AI grounds response with Google Search  │
│  7. Parse & return structured JSON          │
└──────────────────┬──────────────────────────┘
                   │
      ┌────────────┴────────────┐
      ▼                         ▼
  External Stock API        Google Search
  (Alpha Vantage /          Grounding / Web
   Polygon / Finnhub)       Search Tool
```

### 2.2 Module Structure

```
stock-research-assistant/
├── server/
│   ├── index.js (or app.py)          # Express / FastAPI entry point
│   ├── routes/
│   │   └── research.js               # POST /api/research handler
│   ├── tools/
│   │   └── getStockPrice.js          # External stock API call
│   ├── ai/
│   │   └── claudeClient.js           # Anthropic API client + tool loop
│   └── utils/
│       └── validateTicker.js         # Input sanitisation
├── client/
│   ├── index.html
│   ├── style.css
│   └── app.js                        # Fetch + UI render logic
├── .env                              # API keys (never committed)
├── .env.example                      # Template committed to repo
└── README.md
```

---

## 3. Function Calling Schema

### 3.1 Tool Declaration (Anthropic SDK format)

```json
{
  "name": "get_stock_price",
  "description": "Retrieves real-time stock price and market data for a given ticker symbol from an external financial API. Call this whenever the user asks about a stock's current price, price change, or market status.",
  "input_schema": {
    "type": "object",
    "properties": {
      "ticker": {
        "type": "string",
        "description": "The stock ticker symbol, e.g. AAPL, GOOGL, MSFT. Must be uppercase."
      }
    },
    "required": ["ticker"]
  }
}
```

### 3.2 Tool Declaration (Google AI Studio / Gemini format)

```json
{
  "functionDeclarations": [
    {
      "name": "get_stock_price",
      "description": "Fetches real-time stock price and market data for a ticker symbol.",
      "parameters": {
        "type": "OBJECT",
        "properties": {
          "ticker": {
            "type": "STRING",
            "description": "Stock ticker symbol in uppercase, e.g. AAPL"
          }
        },
        "required": ["ticker"]
      }
    }
  ]
}
```

### 3.3 Expected Tool Output Shape

```json
{
  "company_name": "Apple Inc.",
  "ticker": "AAPL",
  "latest_price": 189.43,
  "price_change": 2.17,
  "percent_change": 1.16,
  "currency": "USD",
  "market_status": "open",
  "timestamp": "2025-05-09T14:32:00Z"
}
```

---

## 4. Server-Side Implementation

### 4.1 Tool Execution — `getStockPrice.js` (Node.js / Alpha Vantage)

```js
// server/tools/getStockPrice.js
import fetch from 'node-fetch';

export async function getStockPrice(ticker) {
  const apiKey = process.env.ALPHA_VANTAGE_API_KEY;
  const url = `https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol=${ticker}&apikey=${apiKey}`;

  const res = await fetch(url);
  if (!res.ok) throw new Error(`Stock API HTTP error: ${res.status}`);

  const data = await res.json();
  const quote = data['Global Quote'];

  if (!quote || !quote['05. price']) {
    throw new Error(`Invalid ticker or no data returned for ${ticker}`);
  }

  const price     = parseFloat(quote['05. price']);
  const change    = parseFloat(quote['09. change']);
  const changePct = parseFloat(quote['10. change percent'].replace('%', ''));

  return {
    company_name:   ticker,          // Alpha Vantage doesn't return name; enrich separately if needed
    ticker:         quote['01. symbol'],
    latest_price:   price,
    price_change:   change,
    percent_change: changePct,
    currency:       'USD',
    market_status:  getMarketStatus(),
    timestamp:      new Date().toISOString()
  };
}

function getMarketStatus() {
  const now = new Date();
  const et  = new Date(now.toLocaleString('en-US', { timeZone: 'America/New_York' }));
  const h   = et.getHours(), m = et.getMinutes(), day = et.getDay();
  const isWeekday  = day >= 1 && day <= 5;
  const afterOpen  = h > 9 || (h === 9 && m >= 30);
  const beforeClose = h < 16;
  return isWeekday && afterOpen && beforeClose ? 'open' : 'closed';
}
```

### 4.2 AI Agentic Loop — `claudeClient.js`

```js
// server/ai/claudeClient.js
import Anthropic from '@anthropic-ai/sdk';
import { getStockPrice } from '../tools/getStockPrice.js';

const client = new Anthropic();   // reads ANTHROPIC_API_KEY from env

const tools = [
  {
    name: 'get_stock_price',
    description: 'Fetches real-time stock price and market data for a given ticker symbol.',
    input_schema: {
      type: 'object',
      properties: {
        ticker: { type: 'string', description: 'Uppercase ticker symbol, e.g. AAPL' }
      },
      required: ['ticker']
    }
  }
];

export async function researchStock(ticker) {
  const messages = [
    {
      role: 'user',
      content: `Research the stock "${ticker}". First call get_stock_price to get live data, then use your knowledge and web context to find 3 recent relevant news articles about this company. Return a structured JSON response with stock data and news articles.`
    }
  ];

  // Agentic tool loop
  while (true) {
    const response = await client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 2048,
      tools,
      messages
    });

    if (response.stop_reason === 'tool_use') {
      const toolUseBlock = response.content.find(b => b.type === 'tool_use');
      const toolResult   = await getStockPrice(toolUseBlock.input.ticker);

      messages.push({ role: 'assistant', content: response.content });
      messages.push({
        role: 'user',
        content: [{
          type:        'tool_result',
          tool_use_id: toolUseBlock.id,
          content:     JSON.stringify(toolResult)
        }]
      });
      continue;
    }

    // stop_reason === 'end_turn'
    const textBlock = response.content.find(b => b.type === 'text');
    return textBlock.text;
  }
}
```

### 4.3 Route Handler — `research.js`

```js
// server/routes/research.js
import { Router }       from 'express';
import { researchStock } from '../ai/claudeClient.js';
import { validateTicker } from '../utils/validateTicker.js';

const router = Router();

router.post('/', async (req, res) => {
  const { ticker } = req.body;

  if (!ticker || !validateTicker(ticker)) {
    return res.status(400).json({ error: 'Invalid ticker symbol' });
  }

  try {
    const result = await researchStock(ticker.toUpperCase());
    res.json({ result });
  } catch (err) {
    console.error(err);
    if (err.message.includes('Invalid ticker')) {
      res.status(404).json({ error: 'Unable to retrieve stock data. Check the ticker symbol.' });
    } else {
      res.status(502).json({ error: 'Stock data service unavailable. Try again later.' });
    }
  }
});

export default router;
```

### 4.4 Ticker Validation — `validateTicker.js`

```js
// server/utils/validateTicker.js
export function validateTicker(ticker) {
  // 1–5 uppercase letters, optionally with . or - for foreign listings (e.g. BRK.B)
  return /^[A-Za-z]{1,5}([.\-][A-Za-z]{1,2})?$/.test(ticker.trim());
}
```

---

## 5. AI Prompt Design

### 5.1 System Prompt

```
You are a professional stock research assistant. When given a ticker:
1. Always call get_stock_price first to retrieve live market data.
2. After receiving the data, search for 3 recent news articles about the company.
3. Return a single JSON object — no markdown, no preamble — with this exact shape:

{
  "stock": {
    "company_name": string,
    "ticker": string,
    "latest_price": number,
    "price_change": number,
    "percent_change": number,
    "currency": string,
    "market_status": "open" | "closed" | "pre-market" | "after-hours",
    "timestamp": ISO8601 string
  },
  "news": [
    {
      "title": string,
      "source": string,
      "published_at": string | null,
      "url": string,
      "snippet": string
    }
  ]
}

Return exactly 3 news articles. If you cannot find news, return an empty array and note it.
```

### 5.2 Google Search Grounding (AI Studio)

When using Google AI Studio, enable grounding like this in the API call:

```json
{
  "tools": [
    { "googleSearch": {} }
  ]
}
```

Combine with the function declaration tool in the same request by passing both in `tools[]`.

---

## 6. Frontend UI Contract

### 6.1 States

| State       | What to show                                                              |
|-------------|---------------------------------------------------------------------------|
| `idle`      | Input + button only                                                       |
| `loading`   | Spinner / skeleton cards. Disable submit button. Show "Researching…"      |
| `success`   | Stock card + news list                                                    |
| `error`     | Inline red error banner with human-readable message. Keep input editable. |

### 6.2 Error Messages (user-facing)

| Code / Cause                  | User Message                                               |
|-------------------------------|------------------------------------------------------------|
| Empty input                   | "Please enter a ticker symbol."                            |
| Invalid format                | "Invalid ticker symbol. Use 1–5 letters, e.g. AAPL."      |
| 404 from server               | "Unable to retrieve stock data. Check the ticker symbol."  |
| 502 / network error           | "Stock data service unavailable. Please try again later."  |
| News fetch failed             | "Stock data loaded, but news search failed."               |

### 6.3 Results Card Structure

```
┌────────────────────────────────────────────────┐
│  Apple Inc. (AAPL)                             │
│  $189.43   ▲ +$2.17  (+1.16%)                 │
│  Market: OPEN  •  Updated: May 9, 2025 2:32 PM │
└────────────────────────────────────────────────┘

Recent News
───────────
[1] Title of Article One
    Reuters · May 8, 2025
    Short snippet of the article explaining what it covers...
    [Read more →]

[2] Title of Article Two
    Bloomberg · May 7, 2025
    ...

[3] Title of Article Three
    CNBC · May 6, 2025
    ...
```

### 6.4 Client-Side Fetch Pattern

```js
// client/app.js
async function researchStock(ticker) {
  setLoading(true);
  clearError();

  try {
    const res = await fetch('/api/research', {
      method:  'POST',
      headers: { 'Content-Type': 'application/json' },
      body:    JSON.stringify({ ticker })
    });

    const data = await res.json();

    if (!res.ok) throw new Error(data.error || 'Unknown error');

    renderResults(data.result);
  } catch (err) {
    showError(err.message);
  } finally {
    setLoading(false);
  }
}
```

---

## 7. Environment Variables

### 7.1 Required Variables

| Variable                  | Description                                              |
|---------------------------|----------------------------------------------------------|
| `ANTHROPIC_API_KEY`       | Anthropic API key (for Claude tool use + grounding)      |
| `ALPHA_VANTAGE_API_KEY`   | Alpha Vantage free/paid key for real-time stock data     |
| `PORT`                    | Server port (default: 3000)                              |
| `NODE_ENV`                | `development` or `production`                            |

### 7.2 `.env.example`

```env
# Anthropic (Claude) — https://console.anthropic.com
ANTHROPIC_API_KEY=sk-ant-...

# Stock market data — https://www.alphavantage.co/support/#api-key
# Free tier: 25 requests/day. Paid plans for higher limits.
ALPHA_VANTAGE_API_KEY=your_key_here

# Alternatively, use Polygon.io:
# POLYGON_API_KEY=your_key_here
# Or Finnhub:
# FINNHUB_API_KEY=your_key_here

PORT=3000
NODE_ENV=development
```

### 7.3 Alternative Stock APIs

| Provider        | Free Tier           | Real-time | Notes                              |
|-----------------|---------------------|-----------|------------------------------------|
| Alpha Vantage   | 25 req/day          | ~15min delay (free) / real-time (paid) | Good for prototyping |
| Polygon.io      | Unlimited (delayed) | Yes (paid) | Best developer experience          |
| Finnhub         | 60 req/min          | Yes        | Solid free real-time option        |
| Yahoo Finance   | Unofficial          | Yes        | No official API; use yfinance lib  |

---

## 8. Error Handling Matrix

| Layer      | Failure Scenario                    | Handling Strategy                              |
|------------|-------------------------------------|------------------------------------------------|
| Frontend   | Empty input on submit               | Client-side validation, no request made        |
| Frontend   | Network failure                     | catch → showError("Service unavailable")       |
| Server     | Invalid ticker format               | 400 response before any API call               |
| Server     | Stock API 4xx                       | Throw with "Invalid ticker" → 404 to client    |
| Server     | Stock API 5xx / timeout             | Throw generic → 502 to client                  |
| Server     | Claude API failure                  | 502 + log full error server-side only          |
| AI loop    | Tool called with wrong ticker       | Validation inside getStockPrice(), rethrow     |
| AI loop    | Infinite tool loop                  | Max iteration counter (cap at 5 iterations)    |

---

## 9. Testing Checklist

### 9.1 Smoke Tests (manual)

- [ ] Enter `AAPL` → live price loads, 3 news articles appear
- [ ] Enter `GOOGL` → live price loads, 3 news articles appear
- [ ] Enter `MSFT` → live price loads, 3 news articles appear
- [ ] Submit empty input → "Please enter a ticker symbol." shown, no API call made
- [ ] Enter `XXXXXX` (invalid) → "Invalid ticker symbol" error shown
- [ ] Enter `FAKEX` (valid format, no data) → "Unable to retrieve stock data" shown
- [ ] Resize to 375px width → UI still readable and functional (mobile)
- [ ] While loading → submit button disabled, spinner visible
- [ ] After error → input still editable, user can retry

### 9.2 Function Calling Verification

To confirm the tool is being called (not hallucinated):

```js
// Add logging in the tool loop:
console.log('[tool_use]', toolUseBlock.input);
console.log('[tool_result]', toolResult);
```

Expected log for `AAPL`:
```
[tool_use] { ticker: 'AAPL' }
[tool_result] { company_name: 'AAPL', ticker: 'AAPL', latest_price: 189.43, ... }
```

### 9.3 Grounding Verification

Check that the 3 returned news articles:
- Have a `url` that is a real, clickable HTTP link
- Have a `published_at` within the past 7–30 days
- Have titles that mention the company name or ticker

---

## 10. Security Checklist

- [ ] `.env` is listed in `.gitignore` — never committed
- [ ] API keys are only read from `process.env` on the server
- [ ] Frontend JS contains zero API keys
- [ ] Ticker input is sanitised server-side before passing to external APIs
- [ ] Server errors are logged internally; only generic messages returned to client
- [ ] Rate limiting middleware applied to `/api/research` (e.g. express-rate-limit)
- [ ] CORS configured to allow only your frontend origin in production

---

## 11. Extensibility Hooks

| Feature                     | Where to add                                            |
|-----------------------------|--------------------------------------------------------|
| Multiple ticker comparison  | Accept `tickers[]` array in request body               |
| Historical price chart      | Add `get_price_history(ticker, range)` tool            |
| Analyst ratings             | Add `get_analyst_ratings(ticker)` tool                 |
| Portfolio tracking          | Add user auth + database layer                         |
| WebSocket live prices       | Replace polling with WS connection to stock API        |
| Caching                     | Redis cache on `get_stock_price` results (TTL: 60s)    |

---

## 12. Quick Start Commands

```bash
# Clone and install
git clone <repo>
cd stock-research-assistant
npm install

# Set up environment
cp .env.example .env
# Edit .env with your real API keys

# Run development server
npm run dev    # nodemon server/index.js

# Run production
npm start

# Test endpoints directly
curl -X POST http://localhost:3000/api/research \
  -H "Content-Type: application/json" \
  -d '{"ticker":"AAPL"}'
```

---

## 13. Key Design Decisions & Rationale

| Decision                              | Rationale                                                                 |
|---------------------------------------|---------------------------------------------------------------------------|
| Server-side tool execution            | API keys never reach the client; prevents credential leakage              |
| Agentic while-loop for tool calls     | Handles multi-turn tool use correctly; AI can call tools multiple times   |
| Separate `validateTicker` utility     | Single place to update validation rules; testable in isolation            |
| Return raw JSON from AI, parse client | Decouples AI output format from UI rendering logic                        |
| 3 news articles (fixed count)         | Balances information density vs. UI clutter; explicitly prompted          |
| Market status derived server-side     | More reliable than AI guessing; uses actual NYSE hours                    |
| `.env.example` committed to repo      | Onboarding guide without exposing secrets                                 |

---

## Reference Files

For larger implementations, break out into:

- `references/alpha-vantage-api.md` — Full Alpha Vantage endpoint reference
- `references/polygon-api.md` — Polygon.io endpoint reference  
- `references/ai-studio-grounding.md` — Google AI Studio grounding setup
- `references/anthropic-tool-use.md` — Anthropic tool use loop patterns
- `assets/loading-skeleton.css` — CSS skeleton animation classes
