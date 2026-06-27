# agentic-algotrade-risk-manager

A multi-agent stock-analysis system built on [n8n](https://n8n.io). A user sends a stock ticker to a **Telegram bot**; an AI agent orchestrates two specialist sub-agents — one for **technical analysis** (price action + indicators) and one for **news sentiment + earnings** — then returns a plain-language **Hebrew analysis and recommendation** back to Telegram.

> **Disclaimer:** This is an educational project. It is **not financial advice** and does not execute trades. Indicator readings and recommendations are model-generated and may be inaccurate or internally inconsistent. Do not make investment decisions based on its output.

---

## What it does

1. You message the bot a ticker (e.g. `AAPL`).
2. A **Tools Agent** extracts the symbol and calls two tools in parallel:
   - **Technical Analysis Tool** — fetches a TradingView chart + price history, RSI, MACD, and Bollinger Bands, and writes a technical read.
   - **Trends Analysis Tool** — pulls recent news sentiment and recent SEC 8-K (earnings) filings.
3. The agent combines both into a single structured result, decides a recommendation (buy / wait / sell), and writes it in Hebrew with each technical term briefly explained for non-experts.
4. The result is formatted and sent back to you in Telegram.

---

## Repository contents

| File | Workflow | Role |
|------|----------|------|
| `3_main_agent_telegram.json` | **3 - Stock Analysis Agent (Telegram)** | Entry point. Telegram in/out + the orchestrating AI Agent. Import **last**. |
| `1_technical_analysis_tool.json` | **1 - Technical Analysis Tool** | Sub-workflow called by the agent. Chart + indicators. |
| `2_trends_analysis_tool.json` | **2 - Trends Analysis Tool** | Sub-workflow called by the agent. News sentiment + earnings filings. |

The two tool workflows each start with an **Execute Workflow Trigger** so the agent can call them.

---

## External services & API keys

You will need free-tier accounts / keys for:

| Service | Used for | Where it goes in n8n |
|---------|----------|----------------------|
| **Groq** | LLM inference (OpenAI-compatible) | An "OpenAI" credential with Base URL `https://api.groq.com/openai/v1`, or the native Groq credential on the Chat Model nodes |
| **Telegram Bot** | Receiving the ticker and sending the reply | Telegram credential (bot token from @BotFather) |
| **Alpha Vantage** | News sentiment (`NEWS_SENTIMENT`) | Pasted into the `Set Variables` node (workflow 2) |
| **Twelve Data** | Price history, RSI, MACD, Bollinger | Pasted into the `Set Stock Symbol and API Key` node (workflow 1) |
| **chart-img.com** | TradingView chart image | Header-auth credential on `Get Chart URL` (workflow 1) |
| **SEC EDGAR** | 8-K earnings filings (no key) | Put a contact email in the `User-Agent` of `Get Earnings Filings` (workflow 2) — SEC requires this |

> API keys are **never** stored inside the workflow JSON. You create them as n8n credentials / paste them after import.

---

## Setup

**Order matters** — import the tools first so the agent can link to them.

1. **Import the two tool workflows first**, and **Save** each so it appears in your Workflows list:
   - `1_technical_analysis_tool.json`
   - `2_trends_analysis_tool.json`
2. **Import** `3_main_agent_telegram.json`.
3. **Create credentials** (n8n -> Credentials):
   - Groq (or an "OpenAI" credential pointed at `https://api.groq.com/openai/v1`)
   - Telegram (bot token)
   - chart-img (Header Auth: header `x-api-key`)
4. **Select credentials** on every model node and HTTP node in all three workflows.
5. **Paste the plain-text keys:** Twelve Data -> `Set Stock Symbol and API Key` (wf 1); Alpha Vantage -> `Set Variables` (wf 2); your email -> `Get Earnings Filings` User-Agent (wf 2).
6. **Relink the tools:** open workflow 3, open the **Technical Analysis Tool** and **Trends Analysis Tool** nodes, and select the actual imported workflows from the dropdown (n8n assigns new internal IDs on import, so the original references won't resolve).
7. **Activate** workflow 3 and message your bot a ticker.

---

## Configuration notes (learned the hard way)

These settings matter for the system to run reliably:

- **Model choice:** use a tool-calling-capable model for the agent. A current **Qwen3** model on Groq works well. The `gpt-oss` family caused tool-call format errors with the agent here.
- **Max tokens:** set the agent's Chat Model max tokens high enough (~3500) so the JSON response is not **truncated mid-field** — truncated JSON can't be parsed and the message fails.
- **Memory:** keep the agent's memory window very small (1) or disabled. A long window causes the agent to "bleed" a previous ticker's analysis into the next request.
- **Date:** today's date is injected into the agent prompt (`{{ $now.format('dd/MM/yyyy') }}`) because the model cannot know the current date and will otherwise invent one.
- **Output format:** the Structured Output Parser is **disabled**; the `Build Telegram Message` code node parses the agent's JSON defensively (strips code fences / leading whitespace, falls back to the first `{`...last `}`).
- **Rate limits:** Groq's free tier caps tokens-per-minute. Pace test messages, keep outputs concise, or upgrade tier.
- **Hebrew / RTL:** Telegram infers text direction per line, so each line leads with a Hebrew label or emoji to render right-to-left correctly.

---

## Architecture

```
Telegram (ticker)
      |
      v
  AI Agent --------------+
  (Tools Agent)          | calls in parallel
      |                  |
      |        +---------+----------+
      |        v                    v
      |  Technical Analysis   Trends Analysis
      |  (chart, RSI, MACD,   (news sentiment +
      |   Bollinger)           SEC 8-K earnings)
      |        +---------+----------+
      v                  | results
  combine + decide <-----+
      |
      v
  Build Telegram Message  ->  Send to Telegram (Hebrew analysis + recommendation)
```

---

## Course context

Built for **Track 1 — n8n-based Multi-Agent financial system**: at least three cooperating agents (sentiment, technical, and a decision-maker), connection to financial APIs, and an automatically generated report delivered to Telegram.

## Acknowledgements

Adapted and heavily modified from a community n8n "stock analysis with technical + news sentiment" template, re-architected to run on Groq, output to Telegram in Hebrew, add quantitative RSI, and pull SEC earnings filings.
