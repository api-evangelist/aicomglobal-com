---
name: aicomglobal-com-run-a-free-toolkit-service
description: Discover and call one of aicomglobal's 78 free deterministic Toolkit services (JSON repair, hashing, HMAC, safe arithmetic, cron and timezone math, text chunking, prompt-injection scan, secret redaction, idempotency-key derivation) with no key and no cost, and trust each result exactly as far as its method label says.
api: openapi/aicomglobal-com-openapi.json
operations:
  - listServices
  - runService
method: generated
generated: '2026-09-19'
grounding: >-
  Both operationIds exist verbatim in openapi/aicomglobal-com-openapi.json. The catalog shape, the envelope,
  the describe route and the worked example are quoted from live GET /svc, GET /svc/json_repair and POST
  /svc/json_repair responses fetched 2026-09-19 (see sandbox/ and errors/ in this repo). Nothing here was
  invented; nothing here costs money.
---

# Run a free Toolkit service (aicomglobal)

Base URL `https://aicomglobal.com`. No API key, no signup, no payment — "an agent is never charged to discover, read, or speak." Everything is `application/json`.

## 1. Browse the catalog

`GET /svc` (`listServices`) returns `{count: 78, version: "1.0.1", categories: {...}, envelope: "...", services: [{id, category, method, description}]}`. Categories on 2026-09-19: text 13, hash 6, crypto 5, json 12, data 4, time 8, math 6, nlp 8, web 7, agent 9.

Pick by `id`. Useful ones for an agent mid-task: `json_repair`, `idempotency_key` (derive a stable dedupe key from a payload), the hashing/HMAC and Ed25519/Merkle verifiers, `text_diff`, the prompt-injection scanner and the secret/PII redactor (ids and exact input shapes come from step 2 — do not guess them).

## 2. Read the service's own contract

`GET /svc/<id>` (live, not declared in the spec) returns the description, `usage`, `tags`, `provenance` and `examples[].request.body` — the literal POST body that works. Example fetched live:

```json
{"id":"json_repair","title":"Repair near-JSON","category":"json","method":"lossy",
 "usage":"POST /svc/json_repair — the JSON body is the flat params object shown in examples[].request.body.",
 "examples":[{"request":{"method":"POST","path":"/svc/json_repair","body":{"text":"{a: 1, b: 'x',}"}}}]}
```

The body is the flat params object. (The older `{"input": {...}}` wrapper is also accepted, per the provider's changelog.)

## 3. Run it

`POST /svc/<id>` (`runService`) with that body. Observed live:

```
POST /svc/json_repair  {"text":"{a: 1, b: \"x\",}"}
200 {"ok":true,
     "data":{"repaired":{"a":1,"b":"x"},"repairs":["removed trailing commas","quoted bare key"],"wasValid":false,"repairedText":"{\"a\": 1, \"b\": \"x\"}"},
     "meta":{"service":"json_repair","category":"json","method":"lossy","provenance":{"kind":"none","reproducible":true},"warnings":[],"version":"1.0.1"},
     "error":null}
```

## 4. Read the envelope, not just the status

Every service returns the same envelope: `{ ok, data, meta:{ service, category, method, provenance, warnings, version }, error }`.

- `ok: false` with `error` populated is a service-level failure **inside a 200** — check `ok`, not the HTTP code.
- `meta.method` is the trust label: `exact` (deterministic, safe to rely on), `heuristic` (best effort), `lossy` (information may have been discarded — `json_repair` is lossy by design). Do not treat a lossy result as ground truth for anything material.
- `meta.warnings[]` lists non-fatal changes the service made to your input.
- `meta.provenance.reproducible: true` means the same input yields the same output; you can cache by input hash.

## 5. Failures the contract and the live API declare

- **429** with `Retry-After` — free endpoints are per-IP limited; back off and retry. "No quota is ever silently consumed."
- **404** `{"error":"not_found","message":"No route for POST /svc/<id>."}` — unknown service id; re-read step 1.
- **400** `{ok:false, error:{code:"invalid_json"}}` — malformed body (per the provider's changelog); **413** on an oversized body.
- **405** if you GET `/svc/<id>` expecting to run it — describe is GET, run is POST.

## Same thing over MCP or A2A

MCP (`https://aicomglobal.com/mcp`) and A2A (`https://aicomglobal.com/a2a`) expose the identical catalog as `aicom_list_services {category?}`, `aicom_describe_service {service_id}` and `aicom_run_service {service_id | service, input}` — `input` is the same flat params object. Verified live: an A2A `message/send` with `{"skill":"aicom_list_services","input":{}}` returned the same 78-service catalog as `GET /svc`.
