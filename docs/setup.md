AIWFG v13 is a lean, single-tier n8n workflow that returns structured JSON answers via one LLM call, with cache-first behavior over Redis (Upstash REST). This guide covers installation, configuration, and verification.

***

## Prerequisites

- n8n 1.70+ (self-hosted or cloud) • **Estimated Time:** 2 min
- OpenRouter account + API key • **Estimated Time:** 2 min
- Upstash Redis (REST URL + REST token) • **Estimated Time:** 5 min
- HTTPS reverse proxy (e.g., Caddy) • **Estimated Time:** 5–10 min

Optional:
- Cloudflare Proxy with SSL “Full” mode
- `curl` and `jq` for testing

***

## 1) Import the workflow

- [ ] Open n8n → **Workflows** → **Import from File** → select `workflows/aiwf-main.json`.  
  The workflow name is **AIWFG v13**.  
  **Estimated Time:** 1 min

Do **not** activate yet.

***

## 2) Create credentials

In **n8n → Credentials**, create two HTTP Header credentials:

1) **OpenRouter Auth**  
   - Type: *HTTP Header Auth*  
   - Header name: `Authorization`  
   - Header value: `Bearer {{$env.OPENROUTER_API_KEY}}`  
   **Estimated Time:** 1–2 min

2) **Redis REST Auth**  
   - Type: *HTTP Header Auth*  
   - Header name: `Authorization`  
   - Header value: `Bearer {{$env.REDIS_REST_TOKEN}}`  
   **Estimated Time:** 1–2 min

> The workflow references these by name. Keep the names exact.

***

## 3) Set environment variables

Configure these on the n8n host (Docker `.env`, systemd env, or platform UI):

```bash
TOP_MODEL=deepseek/deepseek-reasoner    # or your preferred top model on OpenRouter
MAX_TOKENS=1024
TEMPERATURE=0.2
MAX_INPUT_CHARS=4000

# OpenRouter
OPENROUTER_API_KEY=sk-or-xxxxxxxx

# Upstash Redis REST
REDIS_BASE_URL=https://<your-upstash-id>.upstash.io
REDIS_REST_TOKEN=xxxxxxxxxxxxxxxx
CACHE_TTL=900
HTTP_TIMEOUT_MS=15000

# Webhook security (optional)
API_KEY=supersecret_personal_api_key

# Instance tag (optional)
AIWF_INSTANCE_ID=aiwfg-v13
````

- Restart n8n after updating env.  
    **Estimated Time:** 2–3 min
    

---

## 4) Configure reverse proxy (Caddy example)

If you terminate TLS at Caddy and want simple CORS:

```caddyfile
aiwf.yourdomain.com {
  tls internal
  @api path /aiwf/v13
  header @api Access-Control-Allow-Origin "*"
  header @api Access-Control-Allow-Methods "POST"
  header @api Access-Control-Allow-Headers "authorization,content-type"
  reverse_proxy n8n:5678
}
```

- If you’re behind Cloudflare: use **Proxy ON** and SSL mode **Full**.
    
- Keep `tls internal` when Caddy sits behind Cloudflare proxy.  
    **Estimated Time:** 5–10 min
    

---

## 5) Activate and test

-  In n8n, **Activate** the AIWFG v13 workflow. **Estimated Time:** 10 sec
    

**Warm test (miss → solve → cache):**

```bash
curl -sS -X POST https://aiwf.yourdomain.com/aiwf/v13 \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"Give me a secure Docker + Caddy setup checklist"}' | jq .
```

You should see JSON:

```json
{
  "answer": "…",
  "steps": ["…"],
  "citations": [],
  "meta": {
    "model": "deepseek/deepseek-reasoner",
    "cache_hit": false
  }
}
```

**Hot test (cache hit):** run the same command again.  
`meta.cache_hit` should be `true`.  
**Estimated Time:** 1 min

---

## 6) Common configuration checks

- OpenRouter HTTP node must be **POST** to `https://openrouter.ai/api/v1/chat/completions`.
    
- `response_format: { type: "json_object" }` is set in the body.
    
- Redis REST:
    
    - GET: `GET <BASE>/GET/<key>`
        
    - SETEX: `GET <BASE>/SETEX/<key>/<ttl>/<value>`
        
- Webhook node uses `responseMode: responseNode`.
    
- CORS handled at proxy; workflow returns JSON only.
    

---

## 7) Troubleshooting

|Symptom|Likely cause|Fix|
|---|---|---|
|`401 UNAUTHORIZED`|Missing/wrong `Authorization: Bearer` header|Set `API_KEY` in env and pass the same in requests, or unset `API_KEY` to disable auth|
|`413 TOO_LARGE`|Input exceeds `MAX_INPUT_CHARS`|Reduce input or raise `MAX_INPUT_CHARS`|
|`502 INVALID_MODEL_OUTPUT`|Model returned non-JSON or malformed JSON|Keep `response_format: json_object`; reduce `TEMPERATURE`; retry|
|Hang/timeout|`HTTP_TIMEOUT_MS` too low or network issue|Increase timeout to 20000; check OpenRouter status|
|Cache never hits|Different cache keys per call|Ensure `q/path/locale/model/max_tokens/temperature` are stable; verify `CACHE_TTL`|
|4xx from Redis|URL pattern or token wrong|Confirm `REDIS_BASE_URL` and REST token; use `/GET/<key>` / `/SETEX/<key>/<ttl>/<value>`|

Enable n8n execution logs for more detail. The workflow populates a `reqId` internally for tracing in Code nodes (you can add `console.log` as needed).

---

## 8) Backup and restore

- Export the workflow JSON from n8n.
    
- Keep a copy of `.env`/runtime env vars and Caddy config.
    
- Redis cache is ephemeral (recomputable); no backup needed.
    

**Estimated Time:** 2–3 min

---

## 9) Upgrade notes (v12 → v13)

- Single tier only; remove prior tier/bandit subflows.
    
- One LLM call; no planner/validator subflows.
    
- Webhook uses `responseNode`; `Respond` nodes deliver the reply.
    
- Cache is Upstash REST only (simple GET/SETEX).
    
- Keep credentials named **exactly**: “OpenRouter Auth”, “Redis REST Auth”.
    

---

## 10) Security hardening (optional)

- Enforce IP allowlist at proxy (Caddy/Cloudflare).
    
- Add HMAC verification at proxy (recommended if exposing publicly).
    
- Lower `MAX_TOKENS` and `TEMPERATURE` where feasible.
    
- Rate limit at proxy (requests/min per IP).  
    **Estimated Time:** 10–20 min
    

---

# AIWFG v13 — docs/architecture.md

AIWFG v13 is a minimal, single-tier design: cache-first, one model call, structured JSON out. This document describes flow, data shapes, and operational behavior.

---

## High-level flow

```
Client
  │ POST /aiwf/v13
  ▼
Webhook (responseNode)
  ▼
Code: Normalize+Auth+Guards  ──┐ error → Respond Error
  ▼                             │
Code: CacheKey                  │
  ▼                             │
HTTP: Redis GET                 │
  ▼                             │
Code: Cache Hit? ── hit ────────┘→ Respond OK
          │
          └─ miss
              ▼
HTTP: LLM Solve (TopTier, POST, JSON-only)
  ▼
Code: Finalize ── error → Respond Error
  ▼
HTTP: Redis SETEX
  ▼
Respond OK
```

- Nodes: **10 total** (no subflows, no tiering).
    
- Single path on miss; cache short-circuits on hit.
    

---

## Nodes and responsibilities

1. **Webhook (POST, responseNode)**  
    Receives requests and defers the response to downstream `Respond` nodes.
    
2. **Normalize+Auth+Guards (Code)**
    
    - Verifies Bearer authorization if `API_KEY` is set.
        
    - Normalizes `q` (trim, collapse whitespace).
        
    - Enforces `MAX_INPUT_CHARS`.
        
    - Stamps `model`, `max_tokens`, `temperature`, `reqId`.
        
    - **Outputs:** success → main; failure → error output.
        
3. **CacheKey (Code)**
    
    - Builds SHA-256 over stable inputs:  
        `q, path, locale, policyVersion, allowWeb, model, max_tokens, temperature`.
        
    - Output: `{ cacheKey, ... }`.
        
4. **Redis GET (HTTP)**
    
    - GET `${REDIS_BASE_URL}/GET/${cacheKey}` (responseFormat=string).
        
5. **Cache Hit? (Code)**
    
    - Tries to parse GET result to JSON.
        
    - If valid, marks `meta.cache_hit=true` and routes to `Respond OK`.
        
    - Otherwise routes to model call.
        
6. **LLM Solve (TopTier) (HTTP)**
    
    - POST `https://openrouter.ai/api/v1/chat/completions`.
        
    - Body includes `response_format: { type: "json_object" }`.
        
    - Messages contain the schema instruction.
        
7. **Finalize (Code)**
    
    - Extracts `choices[0].message.content`.
        
    - Parses JSON; ensures `answer:string`, `steps:string[]`, `citations:string[]`.
        
    - Sets `meta.model` and `cache_hit=false`.
        
    - Builds `cache_value` (stringified).
        
    - On parse failure: route to `Respond Error`.
        
8. **Redis SETEX (HTTP)**
    
    - GET `${REDIS_BASE_URL}/SETEX/${cacheKey}/${CACHE_TTL}/${encodeURIComponent(cache_value)}`.
        
9. **Respond OK (RespondToWebhook)**
    
    - `200` + JSON body (`Content-Type: application/json`).
        
10. **Respond Error (RespondToWebhook)**
    

- `status` from error path (default `500`) + JSON `{ error }`.
    

---

## Data contracts

### Request

- **Method**: `POST`
    
- **Path**: `/aiwf/v13`
    
- **Headers**:
    
    - `Authorization: Bearer <API_KEY>` (optional; enforced if set)
        
    - `Content-Type: application/json`
        
- **Body**:
    
    ```json
    {
      "q": "Your question or task",
      "path": "optional string",
      "locale": "en",
      "allowWeb": false,
      "policyVersion": "v1"
    }
    ```
    

### Response (success)

```json
{
  "answer": "string",
  "steps": ["string", "..."],
  "citations": ["string", "..."],
  "meta": {
    "model": "string",
    "cache_hit": false
  }
}
```

### Response (error)

```json
{ "error": "ERROR_CODE" }
```

Error codes produced internally:

- `UNAUTHORIZED` (401)
    
- `EMPTY` (400)
    
- `TOO_LARGE` (413)
    
- `INVALID_MODEL_OUTPUT` (502)
    
- `UNKNOWN` (500 fallback)
    

---

## Caching behavior

- Key: SHA-256 over stable input & knobs.
    
- Redis routes:
    
    - `GET /GET/<key>` → returns stringified JSON (or nullish).
        
    - `SETEX /SETEX/<key>/<ttl>/<value>` → stores final JSON.
        
- Cache hit returns immediately; cache miss proceeds to model call then writes.
    

TTL is controlled by `CACHE_TTL` (seconds). Pick a value that balances freshness vs. cost.

---

## Security model

- Optional Bearer auth at workflow level (`API_KEY`).
    
- Input size limits (`MAX_INPUT_CHARS`).
    
- CORS at reverse proxy.
    
- HMAC is intentionally omitted to keep nodes minimal; add it at the proxy if needed.
    

---

## Correctness model

- Forces JSON output via `response_format: { type: "json_object" }` (OpenRouter-compatible).
    
- Finalize node validates/normalizes the result shape.
    
- Any malformed output triggers error response; clients can retry.
    

---

## Performance tuning

- **Cache first**: Use moderate `CACHE_TTL` (e.g., 900) for reuse.
    
- **Token budget**: Set realistic `MAX_TOKENS` for your tasks (e.g., 768–1024).
    
- **Temperature**: Lower (0.1–0.3) to reduce schema drift.
    
- **Timeouts**: `HTTP_TIMEOUT_MS=15000` is a good default; raise if needed.
    

---

## Cost model (rough)

Per miss:

- 1 OpenRouter request (tokens_in + tokens_out) × model price
    
- 2 Redis REST calls (GET miss + SETEX)
    

Per hit:

- 1 Redis REST GET
    

To reduce cost:

- Increase `CACHE_TTL`.
    
- Tighten prompts and `MAX_TOKENS`.
    
- Use a model with lower price if quality remains acceptable.
    

---

## Observability

- `reqId` is generated in the first Code node.
    
- You can add `console.log({ reqId, … })` in Code nodes for tracing.
    
- Consider proxy logs (Caddy) for request timing and rate limiting.
    

---

## Extensibility (keep core minimal)

- Add a proxy-level HMAC check without changing workflow nodes.
    
- Add streaming (SSE) at the proxy if the model/provider supports it.
    
- Add a separate “metrics” workflow that reads n8n execution logs; keep AIWFG v13 unchanged.
    

---

## Design constraints

- Minimal node count; no subflows.
    
- Single model call per miss; no planner or validator subflows.
    
- Cache carries the load; correctness is enforced via schema and a single finalize pass.
