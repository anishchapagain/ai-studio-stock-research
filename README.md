# Stock Research Assistant

A full-stack, AI-powered stock research application built with **Google AI Studio**, **Gemini**, **React**, and **Express**. It allows users to query real-time stock data and instantly gathers related recent news articles using Google Search grounding.

## 🚀 Features

- **Real-Time Stock Data:** Fetches up-to-date pricing, price movement, and percentage changes securely from the Finnhub API.
- **AI-Powered News Summarization:** Uses Gemini 3.1 Pro combined with Google Search Grounding to find and summarize the top 3 most relevant and recent news articles for the queried stock ticker.
- **Agentic AI Workflow:** Gemini acts as an agent, first calling a custom server-side tool (`get_stock_price`) to retrieve real-time market data, then executing a Search Grounded query to find contextual news, before returning a strictly formatted JSON response.
- **Modern UI:** Clean, responsive, and accessible interface built with Tailwind CSS and Shadcn UI components.
- **Full-Stack Architecture:** Express backend serving a React SPA via Vite middleware, ensuring secure handling of API keys.
- **Robust Error Handling:** Client-side validation for API keys and graceful fallbacks/error states.

## 🛠️ Tech Stack

- **Frontend:** React 19, Vite, Tailwind CSS v4, Shadcn UI, Lucide React
- **Backend:** Node.js, Express, TSX
- **AI Integration:** `@google/genai` (Gemini SDK), Google Search Grounding
- **APIs:** Finnhub Stock API

## ⚙️ Setup & Configuration

This project requires API keys to function properly. 

1. **Gemini API Key:** Required for the AI orchestration and search grounding.
2. **Finnhub API Key:** Required for fetching real-time stock quotes. (Get a free API key at [Finnhub.io](https://finnhub.io/))

### Environment Variables

Create a `.env` file in the root directory (based on `.env.example`) and add your keys:

```env
GEMINI_API_KEY="your_gemini_api_key_here"
FINNHUB_API_KEY="your_finnhub_api_key_here"
```

## 🚀 Running Locally

Install dependencies:
```bash
npm install
```

Start the development server:
```bash
npm run dev
```

The application will be running on `http://localhost:3000`.

## 📦 Build for Production

Compile the application for production deployment:
```bash
npm run build
```

Start the production server:
```bash
npm start
```

## 🧠 How it Works

1. **User Input:** The user enters a stock ticker (e.g., `AAPL`).
2. **First AI Pass:** The frontend asks Gemini to research the ticker using the `get_stock_price` tool.
3. **Server-Side API Call:** Gemini outputs a function call. The frontend intercepts this and executes a secure GET request to the Express backend (`/api/stock/AAPL`).
4. **Backend Processing:** The backend queries the Finnhub API, structures the data, and returns it.
5. **Second AI Pass:** The frontend sends the structured financial data back to Gemini and requests exactly 3 recent news articles using the Google Search tool.
6. **Final Output:** Gemini processes the context and returns a final strictly-typed JSON object containing the stock data and news, which the frontend renders.

## 📝 License

Apache 2.0
