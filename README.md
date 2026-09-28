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
| `Allowed?` IF node + `Unauthorized` Code node checking `$env.ENRICH_WEBHOOK_SECRET` | The webhook's built-in **Header Auth** (n8n answers 403 by itself when the header is wrong) |
| Serper `X-API-KEY: {{ $env.SERPER_API_KEY }}` | **Header Auth** credential |
| OpenAI `Authorization: Bearer {{ $env.OPEN_AI_API_KEY }}` | The built-in **OpenAI** credential |
| Firecrawl had no API key at all | **Header Auth** credential |
| `$env.OPENAI_FAST_MODEL`, `$env.OPENAI_REASONING_EFFORT` | Constants `MODEL` / `REASONING_EFFORT` at the top of the `OpenAI: read evidence` body |

The rest of the logic is unchanged.

## Setup

1. In n8n, go to **Workflows → Import from file** and choose `lead-enrichment.free-plan.json`.
2. Create these credentials and select each one on its node:

   | Node | Credential type | Name / value |
   | --- | --- | --- |
   | New lead | Header Auth | Name `x-webhook-secret`, Value: a long random string |
   | Firecrawl: homepage | Header Auth | Name `Authorization`, Value `Bearer fc-...` |
   | Serper: LinkedIn search | Header Auth | Name `X-API-KEY`, Value: your Serper key |
   | OpenAI: read evidence | OpenAI | Your OpenAI API key |

3. Activate the workflow. Then POST a lead to the production URL:

   ```bash
   curl -X POST https://<your-n8n>/webhook/enrich-lead \
     -H 'Content-Type: application/json' \
     -H 'x-webhook-secret: <your secret>' \
     -d '{"email":"jane@acme.com","first_name":"Jane","last_name":"Doe"}'
   ```

## Free-tier usage per lead

- Firecrawl: 1 scrape (1 credit). Only runs when a website or work-email domain is known.
- Serper: 1–2 searches.
- OpenAI: 1 `gpt-5-nano` call. OpenAI has no free API tier, but this model is the cheapest option.
- n8n: 1 execution.
