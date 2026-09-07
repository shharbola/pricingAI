# LumanAI

*AI-assisted pricing for lighting and home products.*

A pricing proof of concept for a lighting and home retailer selling in
Germany, France, and Switzerland. It takes two years of daily sales history,
fits a demand model per product, and recommends a price. A small set of AI
agents then read the model output and turn it into a decision a category
manager can act on.

Four parts:

1. Charts of the sales data, so you can see the price-demand relationship, not
   just take the model's word for it.
2. A demand model per product that estimates price elasticity and picks the
   profit-maximising price.
3. Three AI agents (analyst, auditor, strategist) that interpret the result.
4. An evaluation script that checks the agents against the data before anyone
   trusts them.

## Screens

- **Trend** - daily price and units for a product, so price cuts and demand
  spikes line up on the same timeline.
- **Price vs demand** - weekly scatter showing the downward slope the model fits.
- **Recommended price** - current vs recommended, the profit-vs-price curve with
  the optimum marked, and the margin and profit impact.
- **AI team** - the analyst's proposal, the auditor's pushback, and the
  strategist's final call with a confidence score.
- **Output checks** - pass/fail from the evaluation script.

## Layout

```
backend/
  app/
    data.py       load + clean the CSV, aggregations for the charts
    ml.py         Poisson demand model + price optimiser
    llm.py        one provider-agnostic LLM call (Gemini) + offline fallback
    agents.py     the three-agent pipeline and its prompts
    evaluate.py   checks on agent output (also a CLI)
    main.py       FastAPI endpoints
  tests/          pytest, runs offline
  data/           pricing_sales_daily.csv
frontend/         React + Vite + Recharts
docs/DECISIONS.md notes on the modelling choices
```

## Running locally

Backend:

```bash
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

Frontend, in another terminal:

```bash
cd frontend
npm install
npm run dev          # http://localhost:5173, proxies /api to :8000
```

It runs with no API key: the agents fall back to a rule-based mode so the
pipeline works offline. For real LLM output, copy `.env.example` to `.env`, add
a Gemini key (free tier is plenty), and restart the backend.

Or the whole thing in Docker:

```bash
cp .env.example .env      # add your key (optional)
docker compose up --build # frontend :8080, backend :8000
```

## Tests and checks

```bash
cd backend
python -m app.evaluate    # pass/fail table, non-zero exit on failure
pytest -q
```

## Deploying to Cloud Run

Backend first, so you have its URL for the frontend build.

```bash
REGION=europe-west1

gcloud run deploy luman-ai-api \
  --source backend --region $REGION --allow-unauthenticated \
  --set-env-vars GEMINI_API_KEY=$GEMINI_API_KEY

# copy the URL it prints, e.g. https://luman-ai-api-xxxx.run.app

gcloud run deploy luman-ai-web \
  --source frontend --region $REGION --allow-unauthenticated \
  --build-arg VITE_API_BASE=https://luman-ai-api-xxxx.run.app
```

Then set `ALLOWED_ORIGINS` on the backend to the frontend URL and redeploy, so
CORS is limited to your own site. The key stays in Cloud Run's environment and
never goes in the repo.

## The model in one paragraph

Demand is a daily unit count, low and zero on about a third of rows. A log-log
regression can't use those zeros and they carry most of the price signal, so it
underestimates elasticity. Instead the model is a Poisson regression on the raw
counts, one per product, with market and month as fixed effects and ad spend as
a control. The coefficient on log(price) is the elasticity. The optimiser then
searches a bounded price grid for the point that maximises
`(price - cost) x predicted demand`, never below cost and within 30% of today's
price. There's more detail in `docs/DECISIONS.md`.
