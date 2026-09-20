---
name: aicomglobal-com-verify-before-you-pay
description: Vet a third-party x402 service through aicomglobal before sending it money — read the free Reliability Index and Pulse, quote the signed verdict for free, and only then (if your principal has authorised the spend) settle the $0.05 x402 challenge or pay from prepaid credit, honouring the nonce, idempotency and refund rules the provider publishes.
api: openapi/aicomglobal-com-openapi.json
operations:
  - x402Index
  - x402Pulse
  - verdictQuote
  - verdictBuy
  - routeQuote
method: generated
generated: '2026-09-19'
grounding: >-
  Every operationId above exists verbatim in openapi/aicomglobal-com-openapi.json. Prices, the nonce flow,
  the 402 shape, the credit rail and the refund window are quoted from the live GET /verdict and POST /verdict
  responses fetched 2026-09-19, https://aicomglobal.com/pay, llms.txt and https://aicomglobal.com/terms, and
  are cross-referenced in conventions/, errors/ and plans/ in this repo. Nothing here was invented, and this
  skill spends nothing unless step 4 is reached deliberately.
---

# Verify an x402 service before you pay it (aicomglobal)

Base URL `https://aicomglobal.com`. Everything is `application/json`. Steps 1-3 are free and need no key.

## 0. Know the money rules first

- The only paid step is `POST /verdict` at **$0.05 USDC** on Base (`eip155:8453`). Reading the trust signals is free; you pay only for the portable, Ed25519-signed certificate.
- Paying is *your principal's* decision. The provider's own 402 body says: "Before settling you are responsible for ensuring you are AUTHORIZED by your principal to spend and are FUNDED." Do not reach step 4 without that authorisation.
- Every paid action is **idempotent per nonce** — "a retry with the same nonce returns the same artifact, never a second charge." A 402 you trigger without settling costs nothing.
- Refunds are off-protocol: if you settled and no artifact came back (network failure, server error), email `moonspacenow@gmail.com` with the transaction hash **within 14 days** for re-delivery or a full refund. A delivered artifact is final.

## 1. Read the free Index and Pulse

`GET /api/x402` (`x402Index`) — every live x402 Bazaar service ranked by 30-day demand and, where probed, measured reliability (default 200 rows, `?limit=` up to 2000). Find your candidate; an honest `unmeasured` means the provider has never observed it, not that it is bad.

`GET /pulse.json` (`x402Pulse`) — the delta since the last capture: new operators, gone operators, demand and price moves. If your candidate appears under "gone" or a falling-demand line, stop here.

Optional free snapshot for one subject: `GET /badge.json?subject=<https-url>` (undeclared in the spec; returns 400 `invalid_subject` without a subject). Unsigned, "NOT a warranty".

## 2. Quote the verdict — free

`GET /verdict` (`verdictQuote`). The response carries `price` ("$0.05"), `asset`, `network`, `payTo`, the `settle` instructions, a live `whyPay` block ("of N services aicomglobal currently observes, F are failing") and a **single-use, account-bound nonce**. Keep the nonce; it is what makes step 4 replay-safe.

If you have no wallet, read `noWalletPath` in the same response: it names the prepaid-credit route (fund with `GET /credits/x402/starter` -> `POST` + settle $5 USDC, or your principal funds by card at `POST /credits/checkout`), and the header to spend it with.

## 3. Decide whether a signed verdict is worth $0.05

A free `GET /verdict/sample` returns a REAL signed verdict over the provider's own service, same schema and key as a purchased one. Verify its `signature` against `/.well-known/aicom-pubkey` (kid `8ad71ce94418677d`) now; if your verifier cannot check the sample, do not buy one you cannot check either.

Alternative for "which service should I call at all": `GET /route` (`routeQuote`) quotes the $0.002 router that ranks live services for a free-text need. Same nonce-then-POST shape (`routeNeed`).

## 4. Buy the verdict (only if authorised and funded)

`POST /verdict` (`verdictBuy`) with `{"nonce": "<from step 2>", "subject": "<https URL>"}` or `{"nonce": ..., "offering_id": "off_..."}`.

- **Wallet rail:** the first response is **402** with a `payment-required` header (base64 x402 v2: scheme `exact`, network `eip155:8453`, amount `50000` = $0.05 USDC, `maxTimeoutSeconds` 300, facilitator Coinbase CDP v2). Settle it per x402 and retry the same POST with the same nonce.
- **Credit rail:** add header `X-AICOM-PAY: credits` (or body `pay_with: "credits"`) and the price debits from your balance in-band — no 402. Over MCP/A2A the tool is `aicom_verdict` with `pay_with: "credits"` and an optional `idempotency_key`.

The 200 body is the signed verdict: `axes.identity`, `axes.reputation`, `axes.reliability` (each with `score`, `band`, `coverage`), `headlineBand`, `coverage` (`measured | partially-measured | unmeasured`), `evidence.rawRollup`, `reductionSpec`, `kid`, `signature`, `dataHash`. Verify the signature before relying on it; recompute `reductionSpec` over `evidence.rawRollup` if the decision is material.

## 5. Handle what the contract declares

- **402** on the POST — expected on the wallet rail; not an error. If it persists after settlement, check the nonce is the one you quoted and unexpired (lifetime is not published).
- **429** with `Retry-After` on any free GET — per-IP limit; back off. "No quota is ever silently consumed."
- **401 `auth_required`** only appears if you use the credit rail without a Bearer apiKey (`POST /join` mints one free).
- An `unmeasured` verdict is a valid, signed answer that the subject has never been observed — it is not a failure and is not refundable on that basis.

## After you call the third party

Report what you observed for free with the MCP/A2A tool `aicom_report_telemetry {ok, latency_ms, subject | offering_id}` — it strengthens the measured axis for the next agent. No REST twin is declared.
