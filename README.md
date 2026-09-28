# Lead enrichment: n8n workflow (free plan)

`lead-enrichment.free-plan.json` is an n8n workflow. It takes a lead over a webhook and fills in
`company`, `title`, `phone`, `industry`, `employee_count` and `technologies`. The evidence comes from
the company homepage (Firecrawl) and Google results for LinkedIn pages (Serper), and OpenAI reads it.

## What changed from the original to run on the free tier

The original read every secret and setting from `$env` (for example `$env.SERPER_API_KEY` and
`$env.ENRICH_WEBHOOK_SECRET`). n8n Cloud blocks `$env` in nodes, and the replacement, Variables
(`$vars`), needs a paid plan. So every `$env` read is gone:

| Before | Now |
| --- | --- |
| `Allowed?` IF node + `Unauthorized` Code node checking `$env.ENRICH_WEBHOOK_SECRET` | An optional `WEBHOOK_SECRET` constant checked in `Read lead` (401 when wrong) |
| Serper `X-API-KEY: {{ $env.SERPER_API_KEY }}` | The saved `Serper` Custom Auth credential |
| OpenAI `Authorization: Bearer {{ $env.OPEN_AI_API_KEY }}` | The built-in **OpenAI** credential |
| Firecrawl had no API key at all | The saved `Firecrawl` Custom Auth credential |
| `$env.OPENAI_FAST_MODEL`, `$env.OPENAI_REASONING_EFFORT` | Constants `MODEL` / `REASONING_EFFORT` at the top of the `OpenAI: read evidence` body |

The rest of the logic is unchanged.

## Setup

1. Copy the whole JSON file and paste it onto an empty n8n canvas (Ctrl+V / Cmd+V).
2. The nodes point to these saved credentials by name. n8n picks them up on paste; if a node
   still shows a warning, open it and choose the credential:

   | Node | Credential |
   | --- | --- |
   | Firecrawl: homepage | `Firecrawl` (Custom Auth) — must send `Authorization: Bearer fc-...` |
   | Serper: LinkedIn search | `Serper` (Custom Auth) — must send `X-API-KEY: ...` |
   | OpenAI: read evidence | `OpenAI account` (OpenAI) |

3. Optional: to lock the webhook, set `WEBHOOK_SECRET` at the top of the `Read lead` node and send
   the same value in an `x-webhook-secret` header. Left empty, anyone with the URL can call it.
4. Activate the workflow and POST a lead to the production URL:

   ```bash
   curl -X POST https://<your-n8n>/webhook/enrich-lead \
     -H 'Content-Type: application/json' \
     -d '{"email":"jane@acme.com","first_name":"Jane","last_name":"Doe"}'
   ```

## Free-tier usage per lead

- Firecrawl: 1 scrape (1 credit). Only runs when a website or work-email domain is known.
- Serper: 1–2 searches.
- OpenAI: 1 `gpt-5-nano` call. OpenAI has no free API tier, but this model is the cheapest option.
- n8n: 1 execution.
