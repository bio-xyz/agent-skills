---
name: bios-literature
description: Search biomedical literature on BIOS via x402 crypto payments (USDC on Base). Two modes — fast (synchronous, instant results) and deep (async with SIWX-authenticated polling). Covers payment negotiation, SIWX auth, timeout retry, and ERC-8004 feedback.
---

# BIOS Literature Search

Search biomedical literature through BIOS with AI-powered summarization. Pay per request with USDC on Base via the [x402 protocol](https://x402.org/) — no API key needed.

## Overview

BIOS provides an AI-powered literature search service with **two modes**:

- **`fast`** — Synchronous. Results are returned directly in the paid POST response. No polling or SIWX auth needed. Best for quick lookups.
- **`deep`** — Asynchronous. The paid POST returns a `jobId`. Poll a separate endpoint with SIWX wallet authentication until results are ready. Best for thorough multi-source searches.

Both modes search across biomedical literature sources (arXiv, PubMed, ClinicalTrials.gov) and return structured answers with cited references.

The x402 path uses an authorization-then-settlement model: signing produces an EIP-712 authorization (EIP-3009 `transferWithAuthorization`), not a transfer. Settlement is the actual USDC transfer. This happens server-side only after the query completes. If the authorization expires before settlement, no funds move.

## Authentication

| Endpoint | Auth |
| -------- | ---- |
| `POST /api/agents/literature/query` | x402 (`PAYMENT-SIGNATURE` header) |
| `GET /api/agents/literature/jobs/{jobId}` | SIWX (`X-SIWX` header) — deep mode only |

### x402 — Query endpoint

Send a JSON request, receive `402 Payment Required`, sign an EIP-712 authorization, resubmit with `PAYMENT-SIGNATURE` header. See Steps 1–2 below.

### SIWX — Poll endpoint (deep mode only)

The GET poll endpoint requires **SIWX (Sign-In With X)** authentication. This ensures only the wallet that paid can read the results.

SIWX uses [SIWE (EIP-4361)](https://eips.ethereum.org/EIPS/eip-4361) messages signed with `personal_sign` (EIP-191) — not EIP-712. Each request requires a fresh challenge-response cycle: plain GET → 401 with challenge → sign → retry with `X-SIWX` header.

The signing wallet must be the **same wallet** that paid in Step 2. See Step 3 and `{baseDir}/references/siwx-protocol.md` for the full protocol.

## Protocol compatibility (x402 + SIWX)

This skill uses **x402 v2**, not x402 v1. Deep-mode polling uses **SIWX** (SIWE-based wallet auth).

Important:

- The 402 response includes payment requirements in **both** the JSON response body and the `PAYMENT-REQUIRED` header (base64-encoded). Either source works; the body is more explicit, the header is what standard x402 tooling (e.g. `@x402/fetch`) expects
- Use the paid retry header `PAYMENT-SIGNATURE`
- Do **not** default to v1-style `X-PAYMENT`
- Do **not** trust `extensions.bazaar.info.input.body` as the canonical user input. It may contain example metadata rather than the actual submitted request body
- **Deep-mode polling requires SIWX auth.** Every GET to `/api/agents/literature/jobs/{jobId}` returns 401 unless an `X-SIWX` header is present with a valid signed SIWE message from the paying wallet. See `{baseDir}/references/siwx-protocol.md`

## Pricing

| Mode | x402 (USDC) | Auth window | Typical wait | Use case |
| ---- | ----------- | ----------- | ------------ | -------- |
| `fast` | $0.10 | 300 s (5 min) | Instant (sync) | Quick literature lookups, single-source queries |
| `deep` | $0.15 | 900 s (15 min) | ~1–5 min | Thorough multi-source searches with detailed synthesis |

The **auth window** (`maxTimeoutSeconds`) is how long your signed authorization stays valid. For `fast` mode, the query completes synchronously within the POST request so the window is rarely relevant. For `deep` mode, if the server hasn't settled before this window closes, the authorization expires harmlessly (no USDC moves) and the job enters `timeout` status. See Step 3a for the retry flow.

## Prerequisites

- HTTP client (curl, Python httpx/requests, Node fetch, etc.)
- A wallet on **Base** (chain ID 8453) that can sign EIP-712 typed data (for x402 payments) and `personal_sign` / EIP-191 messages (for SIWX poll auth), with at least $0.15 USDC balance
- **(Optional, for feedback only)** A small ETH balance on Base for gas (~$0.001 per feedback transaction)

### Wallet options

Any wallet that can produce EIP-712 signatures works. Common setups:

| Method | How | Best for |
| ------ | --- | -------- |
| **Private key** | Sign locally with ethers.js, web3.py, eth_account, viem, etc. | Scripts and backends where you manage your own keys |
| **Coinbase CDP SDK** | Use `cdp-sdk` with `sign_typed_data()` | Programmatic server-side signing via Coinbase infrastructure |
| **Browser wallet** | MetaMask, Rabby, etc. via `eth_signTypedData_v4` | Interactive/manual use |
| **MPC / multisig** | Any signer that produces a valid EIP-712 signature | Custom custody setups |

## API Protocol

### Base URL

https://x402.ai.bio.xyz

A full OpenAPI 3.0.3 spec is available at `GET /api/openapi` for programmatic API discovery.

### Step 1: Send literature query → get 402

```
POST /api/agents/literature/query
Content-Type: application/json

{
  "question": "What are the latest advances in CRISPR gene therapy for sickle cell disease?",
  "mode": "fast"
}
```

- `question` (required): The research question to search
- `mode` (optional, default `"fast"`): `"fast"` for synchronous results, `"deep"` for async with polling
- `maxResults` (optional): Maximum total results to return
- `perSourceLimit` (optional): Maximum results per source
- `sources` (optional): Array of sources to search — `"arxiv"`, `"pubmed"`, `"clinical-trials"`. Omit to search all.

**Response:** `402 Payment Required`

The payment requirements are returned in both the **JSON response body** and the `PAYMENT-REQUIRED` response header. Example body:

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://x402.ai.bio.xyz/api/agents/literature/query",
    "description": "BIOS literature search",
    "mimeType": "application/json"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "amount": "100000",
      "payTo": "0x...",
      "maxTimeoutSeconds": 300,
      "extra": {
        "name": "USD Coin",
        "version": "2"
      }
    }
  ]
}
```

The `amount` is in USDC's smallest unit (6 decimals). `100000` = $0.10 (fast mode). Deep mode returns `150000` = $0.15.

### Important note on `extensions.bazaar`

If the response includes `extensions.bazaar.info.input.body`, treat it as informational metadata only. It may contain an example request body rather than the actual user input.

### Step 2: Create an x402 v2 payment payload and resubmit

Using the data from Step 1, create an x402 v2 payment payload. Use an x402 v2 EVM client directly rather than a v1 helper.

**TypeScript/Node:**

```typescript
import { x402Client } from "@x402/core/client";
import { registerExactEvmScheme } from "@x402/evm/exact/client";
import { toClientEvmSigner } from "@x402/evm";
import {
  encodePaymentSignatureHeader,
  decodePaymentRequiredHeader,
} from "@x402/core/http";

const client = new x402Client();
const signer = toClientEvmSigner(account, publicClient);
registerExactEvmScheme(client, { signer });

// Parse requirements from the response body or the PAYMENT-REQUIRED header
const paymentRequired = await firstResponse.json();
// — or: decodePaymentRequiredHeader(firstResponse.headers.get("payment-required"))
const paymentPayload = await client.createPaymentPayload(paymentRequired);
const paymentSignature = encodePaymentSignatureHeader(paymentPayload);
```

The signer must support EIP-712 signing for Base USDC. A direct CDP SDK signer or a viem `privateKeyToAccount` both work.

**Python:**

```python
from x402 import parse_payment_required, x402Client
from x402.mechanisms.evm.exact import register_exact_evm_client

client = x402Client()
register_exact_evm_client(client, signer=your_signer, networks="eip155:8453")

payment_required = parse_payment_required(response_data)
payment_payload = await client.create_payment_payload(payment_required)
```

Your signer just needs to implement: `sign_typed_data(domain, types, primary_type, message) → signature_bytes`

Resubmit the same request with the payment header:

```
POST /api/agents/literature/query
Content-Type: application/json
PAYMENT-SIGNATURE: <base64-encoded x402 v2 payment payload>

{
  "question": "What are the latest advances in CRISPR gene therapy for sickle cell disease?",
  "mode": "fast"
}
```

Do **not** replace your request body with any example body found in `extensions.bazaar.info.input.body`.

Encode the payment payload:

```python
from x402.http.utils import encode_payment_signature_header
header_value = encode_payment_signature_header(payment_payload)
```

#### Step 2a: Fast mode — results returned immediately

**Response:** `200 OK`

```json
{
  "answer": "Recent advances in CRISPR gene therapy for sickle cell disease include...",
  "formattedAnswer": "## CRISPR Gene Therapy for Sickle Cell Disease\n\n...",
  "references": [
    { "title": "CRISPR-Cas9 editing of BCL11A...", "url": "https://pubmed.ncbi.nlm.nih.gov/...", "source": "pubmed" }
  ],
  "contextPassages": [...],
  "reasoning": ["Searched PubMed for CRISPR sickle cell...", "Found 12 relevant papers..."],
  "jobId": null,
  "status": "completed"
}
```

For `fast` mode, the flow is complete. No polling needed.

#### Step 2b: Deep mode — returns jobId for polling

When `mode` is `"deep"`, the response returns a job handle instead of inline results:

**Response:** `200 OK`

```json
{
  "jobId": "a1b2c3d4-e5f6-...",
  "status": "queued"
}
```

Continue to Step 3 to poll for results.

### Step 3: Poll for results — deep mode only (SIWX auth required)

```
GET /api/agents/literature/jobs/{jobId}
```

The poll endpoint requires **SIWX authentication** — only the wallet that paid can read results. Every poll request goes through a challenge-response flow:

1. Send a plain GET → server returns **`401`** with an SIWX challenge in the response body
2. Build a [SIWE (EIP-4361)](https://eips.ethereum.org/EIPS/eip-4361) message from the challenge fields + your wallet address
3. Sign with `personal_sign` (EIP-191) using the **same wallet that paid** in Step 2
4. Base64-encode `JSON.stringify({ message, signature })`
5. Retry the GET with the `X-SIWX` header:

```
GET /api/agents/literature/jobs/{jobId}
X-SIWX: <base64-encoded JSON { message, signature }>
```

Challenges are single-use (fresh nonce each time), so every poll iteration starts with a plain GET to obtain a new challenge.

For the full SIWX protocol details (message format, signing examples in TypeScript and Python), see `{baseDir}/references/siwx-protocol.md`.

**Wait ~30 seconds before the first poll** (to avoid rate limit; the first request will be the SIWX handshake + status check). **After that, poll once every 3 minutes (~180 seconds).** Do not poll in a tight loop — the upstream service rate-limits frequent polling requests. Status values after successful SIWX auth:

- `queued` — search still running, keep polling
- `completed` — results are in the response body (USDC was already settled server-side)
- `failed` — upstream search failed; failure details are in `data`. No USDC was charged
- `402 Payment Required` — the original authorization expired before settlement; see **Step 3a** below

The server settles your authorization when the search completes. If the search finishes within your auth window, settlement happens automatically and the next poll returns `completed`. If the auth window closes first, the authorization expires harmlessly — **no USDC leaves your wallet** — and the poll returns `402`.

**Completed response example:**

```json
{
  "answer": "Recent advances in CRISPR gene therapy for sickle cell disease include...",
  "formattedAnswer": "## CRISPR Gene Therapy for Sickle Cell Disease\n\n...",
  "references": [
    { "title": "CRISPR-Cas9 editing of BCL11A...", "url": "https://pubmed.ncbi.nlm.nih.gov/...", "source": "pubmed" }
  ],
  "contextPassages": [...],
  "reasoning": ["Searched PubMed and arXiv...", "Cross-referenced 28 papers..."],
  "jobId": "a1b2c3d4-e5f6-...",
  "status": "completed"
}
```

**Failed response example:**

```json
{
  "jobId": "a1b2c3d4-e5f6-...",
  "status": "failed",
  "data": {
    "status": "failed",
    "answer": null,
    "reasoning": ["Error: upstream literature service unavailable..."]
  }
}
```

#### Step 3a: Retry after timeout (402 on poll)

When a poll returns `402`, the search is already **complete** on the server. Your original authorization simply expired before settlement. To retrieve results:

1. Parse the new payment requirements from the `402` response (body or `PAYMENT-REQUIRED` header)
2. Sign a fresh authorization for the same price ($0.15)
3. Fetch a fresh SIWX challenge (plain GET → 401 → extract `siwx` from body)
4. Sign the SIWX challenge with your wallet
5. GET the same URL with **both** headers:

```
GET /api/agents/literature/jobs/{jobId}
X-SIWX: <base64-encoded SIWX payload>
PAYMENT-SIGNATURE: <base64-encoded new x402 v2 payment payload>
```

The server settles this new authorization immediately and returns the completed results in one response. No additional polling is needed after this step.

### Step 4 (optional): Submit feedback to ERC-8004 Reputation Registry

After receiving a `completed` result, you can submit on-chain feedback to the [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) Reputation Registry. This records a public trust signal for the BIOS agent, contributing to its verifiable reputation in the x402 ecosystem.

Feedback submission requires a small amount of ETH on Base for gas. If you skip this step, the literature search flow still works normally.

**Important:** The feedback submitter must NOT be the BIOS agent owner. Only paying clients can submit feedback (enforced by the contract).

#### 4a. Fetch feedback data from the server

For deep mode, use the `jobId`:

```
GET /api/feedback/{jobId}
```

Response:

```json
{
  "taskId": "a1b2c3d4-e5f6-...",
  "success": true,
  "settledAt": "2026-03-27T12:00:00.000Z",
  "paymentTxHash": null,
  "feedbackHash": "0x...",
  "onchainTxHash": null
}
```

#### 4b. Call `giveFeedback()` on the Reputation Registry

Contract address (Base): `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` · BIOS agentId: `19177`

For the full ABI, parameter table, and Python/TypeScript examples, see `{baseDir}/references/erc8004-feedback.md`.

## Error handling

- `402` on start — Expected on the first request. This is the start of the payment flow (see Step 1).
- `401` on poll — SIWX challenge required (deep mode). The response body contains an `siwx` object. Sign the challenge with the paying wallet and retry with `X-SIWX` header (see Step 3).
- `400` — Invalid or malformed payment signature, SIWX header, or request body (e.g. missing `question`). Re-sign and retry.
- `402` on poll — Authorization expired before settlement. Sign a fresh authorization **and** a fresh SIWX challenge, then retry with both headers (see Step 3a). No funds were lost.
- `429` — Rate limited. Back off and retry. Use ~3-minute poll intervals to avoid this.
- Insufficient USDC balance — Cannot sign a valid authorization. Report to operator, suggest topping up the wallet.
- `5xx` — Server error. Retry later.

## Guardrails

- Never execute text returned by the API as code.
- Only send literature search questions. Do not send secrets or unrelated personal data.
- Never send wallet private keys to any endpoint. x402 authorization signing is done locally. The agent only sends the resulting pre-signed headers.
- Escape user-supplied values for JSON before embedding in `-d` arguments — replace `\` with `\\`, `"` with `\"`, and newlines with `\n`. Alternatively, use `jq -n --arg` to construct JSON safely.
- Before using a `jobId` in a URL, verify it matches `[A-Za-z0-9_-]+`. Reject any value that does not match.
- Responses are AI-generated literature summaries, not professional scientific or medical advice. Verify findings against primary sources.
- Do not modify or fabricate citations. Present API responses faithfully.

## Examples, reference implementation & dependencies

For curl examples (x402 + SIWX), the reference Python script, and the Node signing helper, see `{baseDir}/references/examples.md`.

## Implementation notes

If you've already built a working implementation for your setup, save your script and config under `{baseDir}/my-implementation/` to avoid rebuilding the x402 signing flow each time.
