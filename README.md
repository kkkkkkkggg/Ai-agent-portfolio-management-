# Ai-agent-portfolio-management-Custom GPT Agent: Portfolio Management
This document provides end-to-end implementation to launch your own PortfolioGPT—a Custom GPT
Agent for portfolio analysis and management, just like Hardik & Jatan’s demo. Follow each step to
configure data sources, build and deploy backend functions, and create the GPT in ChatGPT.
1. Prerequisites
Node.js (>=18) installed
A server or serverless environment (e.g., AWS Lambda, Vercel, Heroku)
OpenAI API access with GPT function-calling enabled
Market Data API key (e.g., Alpha Vantage)
2. Project Setup
# Create project
mkdir portfolio-agent && cd portfolio-agent
npm init -y
# Install dependencies
npm install express axios dotenv openai body-parser
Create a .env file:
PORT=3000
OPENAI_API_KEY=your_openai_api_key
ALPHA_VANTAGE_KEY=your_alpha_vantage_api_key
3. Backend Server: index.js
import express from 'express';
import bodyParser from 'body-parser';
import axios from 'axios';
import { Configuration, OpenAIApi } from 'openai';
import dotenv from 'dotenv';
dotenv.config();
const app = express();
app.use(bodyParser.json());
// Initialize OpenAI
const openai = new OpenAIApi(new Configuration({ apiKey:
• 
• 
• 
• process.env.OPENAI_API_KEY }));
// fetch_holdings: Example static holdings
app.post('/fetch_holdings', (req, res) => {
return res.json({
portfolio_id: req.body.portfolio_id,
holdings: [
{ symbol: 'RELIANCE.NS', shares: 50 },
{ symbol: 'TCS.NS', shares: 30 },
{ symbol: 'INFY.NS', shares: 20 }
]
});
});
// fetch_ohlcv: Alpha Vantage API
app.post('/fetch_ohlcv', async (req, res) => {
const { symbol, interval, output_size } = req.body;
const funcMap = { '1d': 'TIME_SERIES_DAILY', '1wk': 'TIME_SERIES_WEEKLY',
'1mo': 'TIME_SERIES_MONTHLY' };
const url = `https://www.alphavantage.co/query?function=$
{funcMap[interval]}&symbol=${symbol}&outputsize=${output_size}&apikey=$
{process.env.ALPHA_VANTAGE_KEY}`;
try {
const { data } = await axios.get(url);
res.json({ symbol, interval, data });
} catch (e) {
res.status(500).json({ error: e.message });
}
});
// calculate_portfolio_rating
app.post('/calculate_portfolio_rating', (req, res) => {
const { returns, volatility, diversification_index } = req.body;
// Simple scoring algorithm
const score = Math.max(0, Math.min(10,
5 + returns * 5 - volatility * 2 + diversification_index * 2
));
res.json({ score: Number(score.toFixed(1)) });
});
// GPT Function Router
app.post('/chat', async (req, res) => {
const { messages } = req.body;
try {
const response = await openai.createChatCompletion({
model: 'gpt-4o-mini',
messages,
functions: [
{
name: 'fetch_holdings',
description: 'Get current portfolio holdings',
parameters: {

type: 'object',

properties: { portfolio_id: { type: 'string' } },

required: ['portfolio_id']

}

},

{

name: 'fetch_ohlcv',

description: 'Fetch OHLCV bars for a ticker',

parameters: {

type: 'object',

properties: {

symbol: { type: 'string' },

interval: { type: 'string', enum: ['1d','1wk','1mo'] },

output_size: { type: 'string', enum: ['compact','full'] }

},

required: ['symbol','interval']

}

},

{

name: 'calculate_portfolio_rating',

description: 'Compute overall portfolio score',

parameters: {

type: 'object',

properties: {

returns: { type: 'number' },

volatility: { type: 'number' },

diversification_index: { type: 'number' }

},

required: ['returns','volatility','diversification_index']

}

}

],

function_call: 'auto'

});

const message = response.data.choices[0].message;

if (message.function_call) {

// Route to local function

const fnName = message.function_call.name;

const args = JSON.parse(message.function_call.arguments);

const result = await axios.post(`http://localhost:${process.env.PORT}/$

{fnName}`, args).then(r => r.data);

// Send result back to GPT for final reply

const final = await openai.createChatCompletion({

model: 'gpt-4o-mini',

messages: [

...messages,

message,

{ role: 'function', name: fnName, content: JSON.stringify(result) }

]
});
return res.json(final.data);
}
res.json(response.data);
} catch (err) {
res.status(500).json({ error: err.message });
}
});
// Start server
app.listen(process.env.PORT, () => console.log(`Server on port $
{process.env.PORT}`));
4. Deploy & Obtain Public URL
Vercel/Heroku: Deploy index.js , set environment variables.
Local Test: node index.js → http://localhost:3000/chat
5. Build the GPT in ChatGPT UI
Explore GPTs → Create a GPT
Info: Name: PortfolioGPT | Avatar: | Description: “Automated portfolio analysis &
ratings.”
System Prompt:
You are PortfolioGPT. Use functions to fetch holdings, price history, and compute
ratings. Ask clarifying questions if inputs are missing. Provide tables and clear
summaries.
Functions Tab: Paste the 3 JSON schemas from above.
API & Actions: Point each function to your deployed endpoint, e.g.:
fetch_holdings → https://your-domain.com/fetch_holdings
fetch_ohlcv → https://your-domain.com/fetch_ohlcv
calculate_portfolio_rating → https://your-domain.com/
calculate_portfolio_rating
Test with sample prompts:
What is my portfolio rating?
Scan my holdings for a bearish trend.
6. Share & Automate
Share: In ChatGPT UI, toggle your GPT to Private or Public and copy the invite link. Send this
link to your Inner Circle for instant access.
• 
• 
1. 
2. 
3. 
4. 
5. 
6. 
7. 
8. 
9. 
10. 
11. 
• Slack Integration:

Create a Slack App and enable Incoming Webhooks.

Add a Webhook URL and note it (e.g., https://hooks.slack.com/services/... ).

In your backend, add an endpoint /send_slack : 

app.post('/send_slack', async (req, res) => {

const { text } = req.body;

await axios.post(process.env.SLACK_WEBHOOK_URL, { text });

res.sendStatus(200);

});

Call Slack after generating your summary: 

const summary = final.data.choices[0].message.content;

await axios.post(process.env.SLACK_WEBHOOK_URL, { text: summary });

Email Alerts (GitHub Actions):

Create a file .github/workflows/weekly-summary.yml : 

name: Weekly Portfolio Summary

on:

schedule:

- cron: '0 9 * * 1' # Every Monday at 09:00 UTC

jobs:

summary:

runs-on: ubuntu-latest

steps:

- uses: actions/checkout@v3

- name: Call PortfolioGPT API

run: |

curl -X POST \

-H "Content-Type: application/json" \

-d '{"messages":[{"role":"user","content":"What is my 

portfolio rating and summary?"}]}' \

https://your-domain.com/chat > summary.txt

- name: Send Email

uses: dawidd6/action-send-mail@v3

with:

server_address: smtp.your-email.com

server_port: 587

username: ${{ secrets.EMAIL_USER }}

password: ${{ secrets.EMAIL_PASS }}

subject: 'Weekly Portfolio Summary'

• 

• 

• 

• 

• 

• 

• 
to: your.email@example.com

body: summary.txt

Add EMAIL_USER and EMAIL_PASS as GitHub Secrets.

7. Continuous Improvement & Versioning

Feature Branches: For each major feature (e.g., options analysis), create a new branch and

update your functions and prompts.

Version Tags: Tag releases (v1.0, v1.1) in Git and note changelogs.

Feedback Loop: Monitor user feedback from your Inner Circle; iterate on prompt phrasing and

function accuracy.

Your PortfolioGPT is now deployed, scheduled, and connected to your workflows!

Appendix: GPT Import Configuration

Below is a complete JSON manifest you can copy & import directly into the ChatGPT Custom GPT builder.

This will pin-to-pin replicate the agent:

... (same manifest as above) ...

How to import: In the GPT builder, click Import → paste this JSON → review and Create.

7. Deploying Your Backend Server

Option A: Vercel

Sign up at vercel.com and install Vercel CLI: npm i -g vercel

In your project root ( portfolio-agent ), run vercel and follow prompts:

Link to existing project or create new

Set Environment Variables ( OPENAI_API_KEY , ALPHA_VANTAGE_KEY , 

SLACK_WEBHOOK_URL )

Vercel will auto-detect the Node server and deploy. Note the provided URL: https://

portfolio-agent.vercel.app .

Option B: Heroku

Install Heroku CLI: npm i -g heroku

heroku login → heroku create portfolio-agent

git push heroku main

Set config vars: 

• 

• 

• 

• 

1. 

2. 

3. 

4. 

5. 

1. 

2. 

3. 

4. 
heroku config:set OPENAI_API_KEY=... ALPHA_VANTAGE_KEY=...

SLACK_WEBHOOK_URL=...

Your server is live at https://portfolio-agent.herokuapp.com .

Verify: curl https://<your-domain>/fetch_holdings -X POST -d 

'{"portfolio_id":"test"}'

8. Importing & Validating the GPT

Go to chat.openai.com → Explore GPTs → Create a GPT → Import

Paste the JSON manifest from the Appendix and click Review.

Confirm System Prompt, Functions, and API & Actions endpoints match your deployed URLs.

Click Create.

9. Testing & Troubleshooting

Test End-to-End

curl -X POST https://<your-domain>/chat \

-H "Content-Type: application/json" \

-d '{"messages":[{"role":"user","content":"What is my portfolio 

rating?"}]}'

Expect a JSON reply with choices[0].message.content containing the score and analysis.

In-UI Testing

Open your new GPT: Ask “What is my portfolio rating?”

Then ask “Scan my holdings for underperformers.”

Common Issues

Rate limits: Alpha Vantage free tier has 5 calls/min. Consider upgrading or caching.

CORS errors: Use serverless (Vercel/Heroku) to avoid browser CORS.

Missing env vars: Double-check names in Vercel/Heroku settings.

10. Next Steps & Customization

Add Fundamentals: Implement get_fundamentals function + schema.

Options Flow: Add fetch_option_chain and integrate OpenAI analysis.

UI Dashboard: Embed GPT in a React app with charts.

5. 

1. 

2. 

3. 

4. 

1. 

• 

• 

• 

• 

• 

• 

• 

• 

• 

• 

• 
11. Advanced Features Implementation

Elevate PortfolioGPT with deeper insights, performance, and UX:

11.1 Fundamentals Integration

Function Schema ( get_fundamentals ): 

{

 "name": "get_fundamentals",

 "description": "Fetch key fundamental metrics for a ticker.",

 "parameters": {

 "type": "object",

 "properties": {

 "symbol": {"type":"string"},

 "metrics": {"type":"array","items":

{"type":"string"},"description":"e.g. [\"PE_RATIO\", \"EPS\"]"}

 },

 "required":["symbol","metrics"]

 }

}

Backend Endpoint ( /get_fundamentals ):

Call a fundamentals API (e.g., Yahoo Finance via RapidAPI).

Return JSON: { symbol, fundamentals: { PE_RATIO: 24.5, EPS: 3.2, … } } .

Prompt Logic: In your system prompt, instruct PortfolioGPT to auto‑call get_fundamentals

whenever user asks for P/E, P/B, dividend yields, etc.

11.2 Options Flow Scanner

Function Schema ( fetch_option_chain ): 

{

 "name": "fetch_option_chain",

 "description": "Retrieve option chain data for a ticker and expiry.",

 "parameters": {

 "type": "object",

 "properties": {

 "symbol": {"type":"string"},

 "expiry": {"type":"string","format":"date"}

 },

 "required":["symbol","expiry"]

 }

}

Backend Endpoint ( /fetch_option_chain ):

Integrate with an options data provider (e.g., Tradier, Deribit).

Return strikes, bid/ask, open interest.

1. 

2. 

3. 

4. 

5. 

1. 

2. 

3. 

4. 
Analysis Logic: Add code to highlight high OI strikes, unusual volume/IV spikes.

11.3 Risk & Performance Caching

Redis/Memcached: Cache OHLCV and fundamentals for 10–15 minutes.

Cache Wrapper: 

async function cachedFetch(key, fetchFn) {

const cached = await redis.get(key);

if (cached) return JSON.parse(cached);

const data = await fetchFn();

await redis.set(key, JSON.stringify(data), 'EX', 900);

return data;

}

Use cachedFetch in /fetch_ohlcv and /get_fundamentals to reduce rate‑limit errors.

11.4 Enhanced Scoring & Analytics

Sharpe Ratio: Add calculate_sharpe(returns, volatility, risk_free_rate)

function.

Sector Diversification: Compute weight per sector; flag overweight/underweight sectors.

Custom Alerts: Allow users to set threshold triggers (e.g., P/E > 30, RSI > 70).

11.5 UI Dashboard Snippet (React + TradingView)

import React, { useState } from 'react';

import { widget } from 'tradingview-js';

import ChatInterface from './ChatInterface';

export default function PortfolioDashboard() {

const [symbol, setSymbol] = useState('RELIANCE.NS');

return (

<div className="grid grid-cols-2 gap-4 p-4">

<div id="tv_chart" className="col-span-1 h-96" />

<div className="col-span-1">

<ChatInterface />

</div>

</div>

);

}

// On mount:

widget({

symbol, interval: 'D', container_id: 'tv_chart', width: '100%', height: 400

});

11.6 Monitoring & Logging

Winston or Pino for structured logs.

5. 

• 

• 

• 

• 

• 

• 

• 

Health Check Endpoint: /health returns uptime & cache stats.

Alerting: Integrate with PagerDuty or Slack for 500-level errors.

By implementing these advanced features, PortfolioGPT will provide richer, faster, and more

robust analysis.

12. Final Validation & Go‑Live Checklist

Ensure every component is polished before full-scale launch:

End‑to‑End Testing:

Test Cases: Prepare 20 real prompts covering all functions. | # | Prompt | Expected Action |

|---|--------------------------------------------------|----------------------------------------------| | 1 | "What is my

portfolio rating?" | Calls fetch_holdings + calculate_portfolio_rating ; returns score

and summary. | | 2 | "Show me NIFTY50 daily OHLCV for last 30 days." | Calls 

fetch_ohlcv(symbol:"^NSEI",interval:"1d") ; returns chart data. | | 3 | "Give P/E and

EPS for TCS.NS." | Calls get_fundamentals(symbol:"TCS.NS",metrics:

["PE_RATIO","EPS"]) . | |...| ... | ... |

Error Simulation: Send invalid symbol ( symbol:"INVALID" ), verify graceful error message.

User Acceptance Testing (Inner Circle):

Share the GPT link with 5–10 users.

Provide a Feedback Form (Google Forms) with ratings on clarity (1–5), speed, accuracy.

Collect at least 10 feedback points; group by theme; iterate prompt phrasing or function logic.

Performance Benchmarking:

Setup: Use autocannon or simple curl + timestamps in a script.

Script Example (Node.js): 

import autocannon from 'autocannon';

const instance = autocannon({

url: 'https://your-domain.com/chat',

method: 'POST',

headers: {'Content-Type':'application/json'},

body: JSON.stringify({messages:[{role:'user',content:'What is my 

portfolio rating?'}]}),

connections: 10,

duration: 20

});

autocannon.track(instance);

Targets:

fetch_holdings : < 500 ms

• 

• 

1. 

2. 

3. 

4. 

5. 

6. 

7. 

8. 

9. 

10. 

11. 

◦ 
fetch_ohlcv : < 1.5 s

Full chat flow: < 2 s

Report: Save results to benchmark-report.md .

Security Review:

Env Variables: Confirm no API keys in source code.

HTTPS: Ensure TLS certificate valid.

Dependency Audit: Run npm audit ; address any critical vulnerabilities.

CORS: Restrict origins for web UI if applicable.

Documentation & Training:

README.md (root of repo): 

# PortfolioGPT Agent

## Quick Start

1. Clone the repo: `git clone ...`

2. Install dependencies: `npm install`

3. Set env vars in `.env`:

 ```ini

OPENAI_API_KEY=...

ALPHA_VANTAGE_KEY=...

SLACK_WEBHOOK_URL=...

 ```

4. Deploy to Vercel: `vercel`

5. Import the GPT manifest in ChatGPT UI (see `manifest.json`).

6. Test in ChatGPT: ask "What is my portfolio rating?"

## Troubleshooting

- Rate limit errors: Upgrade Alpha Vantage or add caching.

- CORS issues: Ensure backend URL matches ChatGPT action URL.

- Slow responses: Check logs and caching layer.

Quick‑Start Guide: PDF or markdown summary with sample prompts & screenshots.

Demo Video Script:

Intro (0:00–0:30): Explain agent purpose.

Setup (0:30–1:00): Show code, env vars, deployment.

Import (1:00–1:30): Demonstrate GPT import in UI.

Live Demo (1:30–2:30): Ask 3 sample prompts, highlight function calls and outputs.

Wrap‑Up (2:30–3:00): Next steps and feedback link.

Go‑Live Plan:

Announce launch date to stakeholders.

Monitor logs, feedback channel (#portfoliogpt-feedback).

◦ 

◦ 

12. 

13. 

14. 

15. 

16. 

17. 

18. 

19. 

20. 

21. 

1. 

2. 

3. 

4. 

5. 

22. 

23. 

24. 
Plan a post‑launch review after one week.

13. Ongoing Maintenance & Roadmap

Weekly: Check API quotas, review error logs.

Monthly: Release minor improvements (e.g., add new metrics).

Quarterly: Security audit, rotate API keys.

Future Roadmap:

Multi‑portfolio support

News sentiment integration

Tax optimization module

All done: These final docs, tests, and plans ensure a smooth, secure, and user‑friendly launch of

PortfolioGPT.

14. Generated Artifacts

Below are ready-to-use files you can copy:

14.1 README.md

# PortfolioGPT Agent

## Quick Start

1. **Clone the repo**

 ```bash

git clone https://github.com/yourusername/portfolio-agent.git

cd portfolio-agent

 ```

2. **Install dependencies**

 ```bash

npm install

 ```

3. **Set environment variables** in a `.env` file:

 ```ini

OPENAI_API_KEY=your_openai_api_key

ALPHA_VANTAGE_KEY=your_alpha_vantage_api_key

SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...

 ```

4. **Deploy**

- **Vercel**: `vercel`

- **Heroku**: `git push heroku main`

25. 

1. 

2. 

3. 

4. 

5. 

6. 

7. 

5. **Import GPT Manifest** in ChatGPT UI (see `manifest.json`).

6. **Test** in ChatGPT: ask "What is my portfolio rating?"

## Troubleshooting

- **Rate limit errors**: Upgrade Alpha Vantage or add caching.

- **CORS issues**: Verify backend URL matches ChatGPT action endpoint.

- **Slow responses**: Check logs or increase cache TTL.

## License

MIT

14.2 manifest.json

{

"name": "PortfolioGPT",

"description": "Automated portfolio analysis and rating with alerts.",

"avatar_url": "https://i.imgur.com/3XKZtQx.png",

"model": "gpt-4o-mini",

"system_prompt": "You are PortfolioGPT... missing details...",

"functions": [ /* fetch_holdings, fetch_ohlcv, calculate_portfolio_rating, 

get_fundamentals, fetch_option_chain */ ],

"api_actions": {

"fetch_holdings": {"url":"https://<your-domain>/

fetch_holdings","method":"POST"},

"fetch_ohlcv": {"url":"https://<your-domain>/

fetch_ohlcv","method":"POST"},

"calculate_portfolio_rating": {"url":"https://<your-domain>/

calculate_portfolio_rating","method":"POST"},

"get_fundamentals": {"url":"https://<your-domain>/

get_fundamentals","method":"POST"},

"fetch_option_chain": {"url":"https://<your-domain>/

fetch_option_chain","method":"POST"}

},

"sample_prompts": [

{"role":"user","content":"What is my portfolio rating?"},

{"role":"user","content":"Give P/E and EPS for TCS.NS"},

{"role":"user","content":"Scan for high OI options on INFY.NS expiry 

2025-07-31"}

]

}
14.3 Feedback Form Template

Use this Google Form structure to collect Inner Circle feedback:

Clarity of outputs (1–5)

Response speed (1–5)

Accuracy of analysis (1–5)

Ease of use (1–5)

Additional comments (paragraph)

Create at: https://docs.google.com/forms/d/e/your-form-id/viewform










