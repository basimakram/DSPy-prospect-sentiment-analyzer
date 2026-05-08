# DSPy Prospect Sentiment Analyzer

A FastAPI service powered by **DSPy** that analyzes sales prospect sentiment from email threads. Uses `ChainOfThought` modules with offline optimization (`BootstrapFewShot` / `MIPROv2`) for systematic prompt improvement.

## Quick Start

```bash
cd prospect-sentiment-analyzer
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env       # add your OPENAI_API_KEY
```

### Run the API

```bash
uvicorn app.main:app --reload --port 8000
```

### Example Request

```bash
curl -s -X POST http://localhost:8000/analyze-sentiment \
  -H 'content-type: application/json' \
  -d '{
    "prospect_name": "Sara Kim",
    "thread": [
      {"sender":"agent","from":"alex@acme.com","body":"Hi Sara, would our platform be a fit?"},
      {"sender":"prospect","from":"sara@helio.io","body":"Maybe — send a one-pager."},
      {"sender":"agent","from":"alex@acme.com","body":"Attached. Happy to walk through on a call."},
      {"sender":"prospect","from":"sara@helio.io","body":"Yes, lets do Friday afternoon."}
    ]
  }' | jq .
```

Returns `overall_sentiment`, `sentiment_trend`, `key_signals` (with verbatim quotes), `objections_or_concerns`, `recommended_next_action`, and more. Full schema in [`app/schemas.py`](app/schemas.py).

---

## Tests

```bash
pytest -q                          # 21 tests, no API key needed
RUN_LIVE_TESTS=1 pytest -q         # includes live LLM smoke test
```

## Eval & Optimization

```bash
python -m eval.run_eval                        # baseline evaluation
python -m eval.run_eval --variant zero-shot
python -m eval.run_eval --compiled compiled/analyzer.json

python -m eval.optimize                        # BootstrapFewShot + MIPROv2
python -m eval.optimize --skip-mipro           # bootstrap only (faster)
```

The compiled artifact (`compiled/analyzer.json`) is loaded at API startup.

---

## Project Structure

```
├── app/
│   ├── main.py                  FastAPI app + /analyze-sentiment endpoint
│   ├── schemas.py               Pydantic request/response models
│   ├── dspy_config.py           DSPy LM config, loads compiled artifact
│   └── pipeline/
│       ├── preprocessing.py     Thread stats, formatting, quote stripping
│       ├── signatures.py        DSPy Signatures (Snapshot + Trended)
│       └── modules.py           ProspectSentimentAnalyzer module
├── eval/
│   ├── dataset.py               Gold-labeled examples
│   ├── metrics.py               Composite evaluation metric
│   ├── run_eval.py              Eval runner with per-mode slicing
│   └── optimize.py              BootstrapFewShot + MIPROv2 optimization
├── tests/
│   ├── test_pipeline.py         Deterministic pipeline tests
│   ├── test_api.py              API tests (stubbed + optional live)
│   └── fixtures/threads.py      Realistic test threads
└── compiled/
    └── analyzer.json            (generated) Optimized program artifact
```

## How It Works

- **Two DSPy Signatures**: `SnapshotSentiment` (single prospect message) and `TrendedSentiment` (multiple messages) — routing is deterministic Python, not prompt-based.
- **Offline optimization**: `BootstrapFewShot` auto-selects demos, `MIPROv2` rewrites instructions via Bayesian search. No optimization in the request path.
- **Composite metric**: Weighted score across sentiment accuracy, trend detection, quote groundedness, objection detection, and action relevance.
- **Quote grounding**: Every `key_signals.quote` must be a verbatim substring from the thread — fast hallucination guardrail with no extra LLM call.
