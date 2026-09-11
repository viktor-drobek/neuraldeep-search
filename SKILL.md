---
name: neuraldeep-search
description: >
  Global search skill powered by NeuralDeep Search API (api.neuraldeep.ru/v1/search/*).
  Enables web search, Telegram channel search, site crawling, and quota checks.
  Automatically activates when user asks for search, research, fresh data, or current events.
metadata:
  neuraldeep:
    emoji: "🔍"
    requires_file: "~/.coddy/providers/neuraldeep/neuraldeep-auth.json"
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

### Primary key source (Coddy-hosted)
The key is read from the Coddy provider auth file:
```bash
~/.coddy/providers/neuraldeep/neuraldeep-auth.json
```
That JSON contains `api_key` (e.g. `sk-...`).

Extract it inline:
```bash
jq -r '.api_key' ~/.coddy/providers/neuraldeep/neuraldeep-auth.json
```

### Fallback key source (env)
If the file above is missing, read from the environment variable:
```bash
${NEURALDEEP_API_KEY}
```

### System requirement
- The `curl` tool is available on the system.
- The `jq` tool is available (or fallback to `python -m json.tool` / `node -e` / `grep -o`).

## Resolve API key helper

Before any call, compute the bearer token once:
```bash
ND_KEY=$(jq -r '.api_key' ~/.coddy/providers/neuraldeep/neuraldeep-auth.json 2>/dev/null)
[ -z "$ND_KEY" ] && ND_KEY="${NEURALDEEP_API_KEY}"
```
If both are empty → return `BLOCKED` and suggest: `coddy providers login neuraldeep`.

## Base URL

```
https://api.neuraldeep.ru/v1
```

---

## 1. Web Search

Search the open web for fresh data, articles, docs, and news.

### curl example
```bash
ND_KEY=$(jq -r '.api_key' ~/.coddy/providers/neuraldeep/neuraldeep-auth.json 2>/dev/null || echo "${NEURALDEEP_API_KEY}")
curl -sS "https://api.neuraldeep.ru/v1/search/web" \
  -H "Authorization: Bearer ${ND_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "<search query>",
    "limit": 5,
    "include_snippets": true
  }' | python3 -m json.tool
```

### Python example
```python
import httpx, json, os, subprocess

KEY = json.load(
    open(os.path.expanduser("~/.coddy/providers/neuraldeep/neuraldeep-auth.json"))
)["api_key"]

r = httpx.post(
    "https://api.neuraldeep.ru/v1/search/web",
    headers={"Authorization": f"Bearer {KEY}"},
    json={"query": "что нового в LLM?", "limit": 5},
).json()
for item in r.get("results", []):
    print(item["title"], "—", item["url"])
    print(item.get("content", "")[:200])
print("remaining:", r.get("quota", {}).get("day", {}).get("remaining"))
```

### Parameters
- `query` (string, required) — search query
- `limit` (integer, optional) — max results, default 5
- `include_snippets` (boolean, optional) — include text snippets

### Response shape
```json
{
  "success": true,
  "results": [
    {"index": 1, "title": "...", "url": "...", "content": "...", "date": ""}
  ],
  "total": 3,
  "query": "...",
  "cost": 15,
  "quota": {"pool": "search", "day": {...}, "month": {...}}
}
```

---

## 2. Telegram Search

Search NeuralDeep's indexed Telegram channel database.

### curl example
```bash
ND_KEY=$(jq -r '.api_key' ~/.coddy/providers/neuraldeep/neuraldeep-auth.json 2>/dev/null || echo "${NEURALDEEP_API_KEY}")
curl -sS "https://api.neuraldeep.ru/v1/search/tg?q=<query>&limit=5" \
  -H "Authorization: Bearer ${ND_KEY}"
```

### Python example
```python
import httpx, json, os
KEY = json.load(open(os.path.expanduser(
    "~/.coddy/providers/neuraldeep/neuraldeep-auth.json"
)))["api_key"]
DATA = httpx.get(
    "https://api.neuraldeep.ru/v1/search/tg",
    params={"q": "RAG в проде", "limit": 5},
    headers={"Authorization": f"Bearer {KEY}"},
).json()
for item in DATA.get("results", []):
    print(item.get("channel"), "—", item.get("text", "")[:120], "→", item.get("message_url"))
```

### Parameters (query string)
- `q` (string, required) — search query
- `limit` (integer, optional) — max results

### Response shape (проверено)
```json
{
  "results": [
    {"channel": "neuraldeep", "text": "...", "date": "...",
     "message_url": "...", "message_id": 123, "views_count": 100,
     "category": "...", "score": 0.9, "source": "tg"}
  ],
  "quota": {"pool": "search", "day": {...}, "month": {...}}
}
```

---

## 3. Site Crawl

Crawl a specific website to extract multiple pages.

### curl example
```bash
ND_KEY=$(jq -r '.api_key' ~/.coddy/providers/neuraldeep/neuraldeep-auth.json 2>/dev/null || echo "${NEURALDEEP_API_KEY}")
curl -sS "https://api.neuraldeep.ru/v1/search/crawl" \
  -H "Authorization: Bearer ${ND_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com", "limit": 3, "depth": 1}'
```

### Python example
```python
import httpx, json, os
KEY = json.load(open(os.path.expanduser(
    "~/.coddy/providers/neuraldeep/neuraldeep-auth.json"
)))["api_key"]
DATA = httpx.post(
    "https://api.neuraldeep.ru/v1/search/crawl",
    headers={"Authorization": f"Bearer {KEY}"},
    json={"url": "https://example.com", "limit": 3, "depth": 1},
).json()
for page in DATA.get("pages", []):
    print(page["url"], page["title"])
```

### Parameters
- `url` (string, required) — starting URL
- `limit` (integer, optional) — max pages to fetch
- `depth` (integer, optional) — crawl depth (default 1)

### Response shape (проверено)
```json
{
  "success": true,
  "pages": [
    {"url": "...", "title": "...", "content": "...", "char_count": 1234}
  ]
}
```

---

## 4. Quota Check

Check remaining search quota. Search and crawl have separate buckets.

### curl example
```bash
ND_KEY=$(jq -r '.api_key' ~/.coddy/providers/neuraldeep/neuraldeep-auth.json 2>/dev/null || echo "${NEURALDEEP_API_KEY}")
curl -sS "https://api.neuraldeep.ru/v1/search/quota" \
  -H "Authorization: Bearer ${ND_KEY}"
```

### Response shape
```json
{
  "tier": "starter",
  "search": {"day": {"used": 0, "limit": 300, "remaining": 300, "reset_at": "..."}, "month": {...}},
  "crawl": {"day": {"used": 0, "limit": 30, "remaining": 30, "reset_at": "..."}, "month": {...}}
}
```

---

## Usage Workflow

1. **Resolve the API key** using the helper above.
2. **Check quota first** if multiple searches are planned:
   ```bash
   curl -sS "https://api.neuraldeep.ru/v1/search/quota" -H "Authorization: Bearer ${ND_KEY}"
   ```
3. **Execute the appropriate endpoint** based on user intent:
   - General web facts → `v1/search/web`
   - Russian Telegram sources → `v1/search/tg`
   - Deep dive into a specific site → `v1/search/crawl`
4. **Parse JSON** and present results concisely with source URLs.

## Error Handling

- `401 Unauthorized` — invalid/missing API key → run `coddy providers login neuraldeep`
- `429 Too Many Requests` — quota exhausted → response includes `Retry-After`; wait or use fallback `v1/search/quota`
- `503 Service Unavailable` — transient outage → retry once after 5 seconds
- `400 Bad Request` — malformed params → verify query encoding and JSON schema

## Cost & Limits

- Search API has its **own quota bucket**, separate from chat completions.
- Rate limits and daily quotas depend on the NeuralDeep Hub tariff (starter, pro, etc).
- Cost per request is returned in the `cost` field of successful responses.
- Consult `v1/search/quota` before heavy batch operations.

## Privacy & Security

- Do not log or expose the full API key.
- Do not send internal/private repository content to search endpoints as queries unless authorized.
- Search queries leave the local machine; treat query content as potentially external.
- SSRF guard blocks private networks and localhost; do not try to route to 10.x, 192.168.x, or 127.0.0.1.
