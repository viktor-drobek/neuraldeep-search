---
name: neuraldeep-search
description: >
  Global search skill powered by NeuralDeep Search API (api.neuraldeep.ru/v1/search/*).
  Enables web search, Telegram channel search, site crawling, and quota checks.
  Automatically activates when user asks for search, research, fresh data, or current events.
metadata:
  version: 1.0.0
  neuraldeep:
    emoji: "🔍"
    base_url: "https://api.neuraldeep.ru/v1"
    endpoints:
      - /v1/search/web
      - /v1/search/tg
      - /v1/search/crawl
      - /v1/search/quota
---

> **Author:** these skills were authored by the **kimi2.6** and **qwen3.8 27b** models, and are published by **viktor-drobek**.

# NeuralDeep Search

Use this skill whenever the user needs:
- Web search or up-to-date information from the internet
- Research on topics not present in the local codebase or conversation
- Telegram channel search (Russian-language sources)
- Site crawling / fetching specific pages for RAG or fact-checking
- Checking remaining search quota

## Requirements

Python 3.10+ is sufficient; the helper uses only the standard library.
Run commands from this skill's directory. No `curl`, `jq`, or `httpx` is required.

The helper reads the key from `NEURALDEEP_SEARCH_API_KEY`, this skill's own variable, then
from the shared `NEURALDEEP_API_KEY`. Each is taken from the environment first,
then from `${CODDY_HOME:-~/.coddy}/.env`, the file Coddy loads at startup, so
a key added there works without a restart. A variable may hold several keys
separated by commas: before a billed request the helper uses the first key
whose free subscription and quota checks pass, and records its number (never
the key) in the state, so `resume` uses the same key. The Coddy provider file
is not read. The helper never prints a key. Without one it returns `blocked`,
naming the variable: ask the user to add it to `~/.coddy/.env` themselves,
for example `NEURALDEEP_SEARCH_API_KEY=sk-...`, never in the chat.

Read [Starter and Relay safety](STARTER_RELAY.md) before submitting work.
Every billed operation must pass the helper's live subscription, public-price,
and service-quota checks. A chat `decision.can_request=false` is not a service
quota decision. State files and artifacts are private runtime data, not repo files.

## Base URL

```
https://api.neuraldeep.ru/v1
```

---

## 1. Web Search

Search the open web for fresh data, articles, docs, and news.

### Guarded helper example
```bash
printf '%s\n' '{"query":"public API documentation","limit":3}' > request.json
python3 scripts/client.py run web --payload request.json --state web-state.json --output web-result.json
```

### Python example
```python
import json
from pathlib import Path
import subprocess
Path("request.json").write_text(json.dumps({"query": "public API documentation", "limit": 3}))
subprocess.run(["python3", "scripts/client.py", "run", "web", "--payload", "request.json",
                "--state", "python-web-state.json", "--output", "web-result.json"], check=True)
data = json.loads(Path("web-result.json").read_text())
for item in data["results"]:
    print(item.get("title"), item.get("url"))
```

### Parameters
- `query` (string, required) — search query
- `limit` (integer, optional) — max results, default 5
- `include_snippets` (boolean, optional) — include text snippets

### Response shape
```text
{
  "success": true,
  "results": [
    {"index": 1, "title": "...", "url": "...", "content": "...", "date": ""}
  ],
  "total": 3,
  "query": "...",
  "cost": 15,
  "quota": {"pool": "search", "mode": "tier", "day": {...}, "month": {...}}
}
```

---

## 2. Telegram Search

Search NeuralDeep's indexed Telegram channel database.

### Guarded helper example
```bash
printf '%s\n' '{"q":"public API documentation","limit":3}' > tg-request.json
python3 scripts/client.py run tg --payload tg-request.json --state tg-state.json --output tg-result.json
```

### Python example
```python
import subprocess
subprocess.run(["python3", "scripts/client.py", "run", "tg", "--payload", "tg-request.json",
                "--state", "python-tg-state.json", "--output", "tg-result.json"], check=True)
```

### Parameters (query string)
- `q` (string, required) — search query
- `limit` (integer, optional) — max results

### Illustrative response shape
Unlike web and crawl, the Telegram response has no `success` or `cost` field.
```text
{
  "results": [
    {"channel": "neuraldeep", "text": "...", "date": "...",
     "message_url": "...", "message_id": 123, "views_count": 100,
     "forwards_count": 0, "replies_count": 0, "reactions": {},
     "category": "...", "score": 0.9, "source": "...", "file_path": null}
  ],
  "total": 1,
  "query": "...",
  "search_type": "...",
  "sources": ["..."],
  "quota": {"pool": "search", "mode": "tier", "day": {...}, "month": {...}}
}
```

---

## 3. Site Crawl

Crawl a specific website to extract multiple pages.

### Guarded helper example
```bash
printf '%s\n' '{"url":"https://example.com","limit":3}' > crawl-request.json
python3 scripts/client.py run crawl --payload crawl-request.json --state crawl-state.json --output crawl-result.json
```

### Python example
```python
import subprocess
subprocess.run(["python3", "scripts/client.py", "run", "crawl", "--payload", "crawl-request.json",
                "--state", "python-crawl-state.json", "--output", "crawl-result.json"], check=True)
```

### Parameters
- `url` (string, required) — starting URL
- `limit` (integer, optional) — max pages to fetch
- `depth` (integer, optional) — crawl depth (default 1)

### Illustrative response shape
```text
{
  "success": true,
  "pages": [
    {"url": "...", "title": "...", "content": "...", "char_count": 1234}
  ],
  "total": 1,
  "cost": 15,
  "quota": {"pool": "crawl", "mode": "tier", "day": {...}, "month": {...}}
}
```

---

## 4. Quota Check

Check remaining search quota. Search and crawl have separate buckets.

### Guarded helper example
```bash
python3 scripts/client.py check web
python3 scripts/client.py check crawl
```

### Verified quota structure

`GET /v1/search/quota` returns top-level `tier`, with separate `search` and
`crawl` objects. Each has `mode: "tier"` and `day.remaining` / `month.remaining`.
The helper requires both windows to cover the operation. Read live values;
do not infer a budget from a pricing tier's advertised ceiling.

---

## Usage Workflow

1. **Resolve the API key** using the helper above.
2. **Check quota before every billed operation** (the helper always does):
   ```bash
   python3 scripts/client.py check web
   ```
3. **Execute the appropriate endpoint** based on user intent:
   - General web facts → `v1/search/web`
   - Russian Telegram sources → `v1/search/tg`
   - Deep dive into a specific site → `v1/search/crawl`
4. **Parse JSON** and present results concisely with source URLs.

## Error Handling

- `401 Unauthorized` — invalid/missing API key → check the key in `NEURALDEEP_SEARCH_API_KEY`
- `429 Too Many Requests` — quota exhausted → response includes `Retry-After`; wait or use fallback `v1/search/quota`
- `503 Service Unavailable` — stop; do not retry a billed request automatically
- `400 Bad Request` — malformed params → verify query encoding and JSON schema

## Cost & Limits

- Search API has its **own quota bucket**, separate from chat completions.
- Rate limits and daily quotas depend on the NeuralDeep Hub tariff (starter, pro, etc).
- Web and crawl responses include a `cost` field; Telegram responses do not. On a
  Starter subscription each request takes one unit from the day and month buckets
  in `quota`, whatever `cost` says.
- Consult `v1/search/quota` before heavy batch operations.

## Privacy & Security

- Do not log or expose the full API key.
- Do not send internal/private repository content to search endpoints as queries unless authorized.
- Search queries leave the local machine; treat query content as potentially external.
- SSRF guard blocks private networks and localhost; do not try to route to 10.x, 192.168.x, or 127.0.0.1.
