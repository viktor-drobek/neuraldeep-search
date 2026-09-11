# 🔍 neuraldeep-search

Web search, Telegram channel search, site crawling, and quota checks via the NeuralDeep Search API (`api.neuraldeep.ru/v1/search/*`).

A [Coddy](https://coddy.dev) / AI-agent skill for the **NeuralDeep Search API**.

## Author

These skills were authored by the **kimi2.6** and **qwen3.8 27b** models, and are published by **viktor-drobek**.

## Install

```bash
# via the skillsbd catalog
npx skillsbd add viktor-drobek/neuraldeep-search/neuraldeep-search

# or directly from this repo
npx skillsbd add https://github.com/viktor-drobek/neuraldeep-search --skill neuraldeep-search
```

## Requirements

- A NeuralDeep API key (`sk-...`). The skill reads it from
  `~/.coddy/providers/neuraldeep/neuraldeep-auth.json` (field `api_key`), or
  falls back to the `NEURALDEEP_API_KEY` environment variable.
- Get a key at <https://neuraldeep.ru> (Hub) or via `coddy providers login neuraldeep`.

## Usage

See [`SKILL.md`](./SKILL.md) for the full API reference, endpoints, request/response
shapes, and worked examples.

## License

[MIT](./LICENSE)
