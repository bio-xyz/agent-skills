---
name: bios-data-analysis
description: Upload files and run paid data analysis on BIOS via x402 crypto payments (USDC on Base). Handles file upload, payment negotiation, SIWX-authenticated polling, timeout retry, and artifact download.
---

# BIOS Data Analysis

Run data analysis tasks on BIOS with optional file uploads. Pay per request with USDC on Base via the [x402 protocol](https://x402.org/) — no API key needed. Results are returned as structured answers with downloadable artifacts.

## Overview

BIOS provides an AI-powered data analysis service accessed exclusively through **x402 crypto payments** (USDC on Base). The flow:

1. (Optional) Upload input files — free, no auth
2. Start a paid analysis task — x402 payment ($2.00 USDC)
3. Poll for results — SIWX wallet authentication
4. Download artifacts — SIWX wallet authentication

The x402 path uses an authorization-then-settlement model: signing produces an EIP-712 authorization (EIP-3009 `transferWithAuthorization`), not a transfer. Settlement is the actual USDC transfer. This happens server-side only after the analysis completes. If the authorization expires before settlement, no funds move.

## Authentication

Three auth layers are used across different endpoints:

| Endpoint                                                 | Auth                              |
| -------------------------------------------------------- | --------------------------------- |
| `POST /api/files/upload`                                 | None                              |
| `POST /api/data-analysis/start`                          | x402 (`PAYMENT-SIGNATURE` header) |
| `GET /api/data-analysis/{taskId}`                        | SIWX (`X-SIWX` header)            |
| `GET /api/data-analysis/{taskId}/artifacts/{artifactId}` | SIWX (`X-SIWX` header)            |

### x402 — Start endpoint

Send a JSON request, receive `402 Payment Required`, sign an EIP-712 authorization, resubmit with `PAYMENT-SIGNATURE` header. See Step 2–3 below.

### SIWX - Poll and artifact endpoints

The GET endpoints require **SIWX (Sign-In With X)** authentication. This ensures only the wallet that paid for the analysis can read results and download artifacts.

SIWX uses [SIWE (EIP-4361)](https://eips.ethereum.org/EIPS/eip-4361) messages signed with `personal_sign` (EIP-191) — not EIP-712. Each request requires a fresh challenge-response cycle: plain GET → 401 with challenge → sign → retry with `X-SIWX` header.

The signing wallet must be the **same wallet** that paid in Step 3. See Step 4 and `{baseDir}/references/siwx-protocol.md` for the full protocol.

## Protocol compatibility (x402 + SIWX)

This skill uses **x402 v2**, not x402 v1. Polling and artifact downloads use **SIWX** (SIWE-based wallet auth).

Important:

- The 402 response includes payment requirements in **both** the JSON response body and the `PAYMENT-REQUIRED` header (base64-encoded). Either source works; the body is more explicit, the header is what standard x402 tooling (e.g. `@x402/fetch`) expects
- Use the paid retry header `PAYMENT-SIGNATURE`
- Do **not** default to v1-style `X-PAYMENT`
- Do **not** trust `extensions.bazaar.info.input.body` as the canonical user input. It may contain example metadata rather than the actual submitted request body
- **Polling and artifact downloads require SIWX auth.** Every GET to `/api/data-analysis/{taskId}` or `.../artifacts/{artifactId}` returns 401 unless an `X-SIWX` header is present with a valid signed SIWE message from the paying wallet. See `{baseDir}/references/siwx-protocol.md`

## Pricing

| Operation | x402 (USDC) | Auth window       | Typical wait | Use case                                           |
| --------- | ----------- | ----------------- | ------------ | -------------------------------------------------- |
| Analysis  | $2.00       | 9 000 s (150 min) | ~5–10 min    | Structured data analysis with optional file inputs |

The **auth window** (`maxTimeoutSeconds`) is how long your signed authorization stays valid. If the server hasn't settled before this window closes, the authorization expires harmlessly (no USDC moves) and the task enters `timeout` status. See Step 4a for the retry flow.

## Prerequisites

- HTTP client (curl, Python httpx/requests, Node fetch, etc.)
- A wallet on **Base** (chain ID 8453) that can sign EIP-712 typed data (for x402 payments) and `personal_sign` / EIP-191 messages (for SIWX poll auth), with at least $2.00 USDC balance
- **(Optional, for feedback only)** A small ETH balance on Base for gas (~$0.001 per feedback transaction)

### Wallet options

Any wallet that can produce EIP-712 signatures works. Common setups:

| Method               | How                                                           | Best for                                                     |
| -------------------- | ------------------------------------------------------------- | ------------------------------------------------------------ |
| **Private key**      | Sign locally with ethers.js, web3.py, eth_account, viem, etc. | Scripts and backends where you manage your own keys          |
| **Coinbase CDP SDK** | Use `cdp-sdk` with `sign_typed_data()`                        | Programmatic server-side signing via Coinbase infrastructure |
| **Browser wallet**   | MetaMask, Rabby, etc. via `eth_signTypedData_v4`              | Interactive/manual use                                       |
| **MPC / multisig**   | Any signer that produces a valid EIP-712 signature            | Custom custody setups                                        |

## API Protocol

### Base URL

https://x402.ai.bio.xyz

A full OpenAPI 3.0.3 spec is available at `GET /api/openapi` for programmatic API discovery.

### Step 1 (optional): Upload input files

```
POST /api/files/upload
Content-Type: multipart/form-data

file: <binary file data>
```

No payment or authentication required. The server proxies the file to the upstream BIOS storage.

**Response:** `200 OK`

```json
{
  "fileId": "a1b2c3d4-...",
  "filename": "gene-expression.csv",
  "contentType": "text/csv",
  "size": 2048,
  "createdAt": "2026-03-27T12:00:00.000Z"
}
```

Save the `fileId` — pass it in Step 2 to attach the file to the analysis task. You can upload multiple files by making multiple requests.

### Step 2: Send analysis request → get 402

```
POST /api/data-analysis/start
Content-Type: application/json

{
  "taskDescription": "Analyze gene expression patterns from RNA-seq data",
  "fileIds": ["a1b2c3d4-..."]
}
```

- `taskDescription` (required): What to analyze
- `fileIds` (optional): Array of file IDs from Step 1

**Response:** `402 Payment Required`

The payment requirements are returned in both the **JSON response body** and the `PAYMENT-REQUIRED` response header. Example body:

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://x402.ai.bio.xyz/api/data-analysis/start",
    "description": "BIOS data analysis task",
    "mimeType": "application/json"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "amount": "2000000",
      "payTo": "0x...",
      "maxTimeoutSeconds": 9000,
      "extra": {
        "name": "USD Coin",
        "version": "2"
      }
    }
  ]
}
```

The `amount` is in USDC's smallest unit (6 decimals). `2000000` = $2.00.

### Important note on `extensions.bazaar`

If the response includes `extensions.bazaar.info.input.body`, treat it as informational metadata only. It may contain an example request body rather than the actual user input.

### Step 3: Create an x402 v2 payment payload and resubmit

Using the data from Step 2, create an x402 v2 payment payload. Use an x402 v2 EVM client directly rather than a v1 helper.

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
POST /api/data-analysis/start
Content-Type: application/json
PAYMENT-SIGNATURE: <base64-encoded x402 v2 payment payload>

{
  "taskDescription": "Analyze gene expression patterns from RNA-seq data",
  "fileIds": ["a1b2c3d4-..."]
}
```

Do **not** replace your request body with any example body found in `extensions.bazaar.info.input.body`.

**Response:** `200 OK`

```json
{
  "taskId": "1e4a1c29-e4d7-4fa7-...",
  "status": "queued"
}
```

### Step 4: Poll for results (SIWX auth required)

```
GET /api/data-analysis/{taskId}
```

The poll endpoint requires **SIWX authentication** — only the wallet that paid can read results. Every poll request goes through a challenge-response flow:

1. Send a plain GET → server returns **`401`** with an SIWX challenge in the response body
2. Build a [SIWE (EIP-4361)](https://eips.ethereum.org/EIPS/eip-4361) message from the challenge fields + your wallet address
3. Sign with `personal_sign` (EIP-191) using the **same wallet that paid** in Step 3
4. Base64-encode `JSON.stringify({ message, signature })`
5. Retry the GET with the `X-SIWX` header:

```
GET /api/data-analysis/{taskId}
X-SIWX: <base64-encoded JSON { message, signature }>
```

Challenges are single-use (fresh nonce each time), so every poll iteration starts with a plain GET to obtain a new challenge.

For the full SIWX protocol details (message format, signing examples in TypeScript and Python), see `{baseDir}/references/siwx-protocol.md`.

**Poll every 60 seconds.** The upstream analysis service rate-limits frequent polling requests. Status values after successful SIWX auth:

- `queued` — analysis still running, keep polling
- `completed` — results are in the response body (USDC was already settled server-side)
- `failed` — upstream analysis failed; failure details are in `data`. No USDC was charged
- `402 Payment Required` — the original authorization expired before settlement; see **Step 4a** below

The server settles your authorization when the analysis completes. If the analysis finishes within your auth window, settlement happens automatically and the next poll returns `completed`. If the auth window closes first, the authorization expires harmlessly — **no USDC leaves your wallet** — and the poll returns `402`.

**Completed response example:**

```json
{
  "id": "1e4a1c29-...",
  "status": "completed",
  "success": true,
  "answer": "The analysis identified 15 differentially expressed genes...",
  "directAnswer": "Key findings: TP53, EGFR, and MYC show significant upregulation...",
  "reasoning": [
    "Step 1: Loaded gene expression data with 25 genes across 6 samples...",
    "Step 2: Applied differential expression analysis..."
  ],
  "artifacts": [
    { "id": "artifact-uuid-1", "name": "results.csv", "type": "text/csv" },
    { "id": "artifact-uuid-2", "name": "volcano-plot.png", "type": "image/png" }
  ]
}
```

**Failed response example:**

```json
{
  "taskId": "1e4a1c29-...",
  "status": "failed",
  "data": {
    "id": "1e4a1c29-...",
    "status": "failed",
    "success": false,
    "answer": null,
    "reasoning": ["Error: unable to parse input file format..."],
    "artifacts": null
  }
}
```

#### Step 4a: Retry after timeout (402 on poll)

When a poll returns `402`, the analysis is already **complete** on the server. Your original authorization simply expired before settlement. To retrieve results:

1. Parse the new payment requirements from the `402` response (body or `PAYMENT-REQUIRED` header)
2. Sign a fresh authorization for the same price ($2.00)
3. Fetch a fresh SIWX challenge (plain GET → 401 → extract `siwx` from body)
4. Sign the SIWX challenge with your wallet
5. GET the same URL with **both** headers:

```
GET /api/data-analysis/{taskId}
X-SIWX: <base64-encoded SIWX payload>
PAYMENT-SIGNATURE: <base64-encoded new x402 v2 payment payload>
```

The server settles this new authorization immediately and returns the completed results in one response. No additional polling is needed after this step.

### Step 5: Download artifacts (SIWX auth required)

For each artifact in the completed response, download it via:

```
GET /api/data-analysis/{taskId}/artifacts/{artifactId}
```

This endpoint also requires SIWX authentication (same challenge-response as Step 4). The task must be `completed`.

1. Send a plain GET → **401** with SIWX challenge
2. Sign the challenge with the paying wallet
3. Retry with `X-SIWX` header

**Response:** `200 OK`

```json
{
  "url": "https://storage.example.com/signed-url?token=...",
  "expiresIn": 3600
}
```

Fetch the `url` value to download the actual file. The URL is a pre-signed download link that expires after `expiresIn` seconds.

### Step 6 (optional): Submit feedback to ERC-8004 Reputation Registry

After receiving a `completed` result, you can submit on-chain feedback to the [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) Reputation Registry. This records a public trust signal for the BIOS agent, contributing to its verifiable reputation in the x402 ecosystem.

Feedback submission requires a small amount of ETH on Base for gas. If you skip this step, the analysis flow still works normally.

**Important:** The feedback submitter must NOT be the BIOS agent owner. Only paying clients can submit feedback (enforced by the contract).

#### 6a. Fetch feedback data from the server

```
GET /api/feedback/{taskId}
```

Response:

```json
{
  "taskId": "1e4a1c29-...",
  "success": true,
  "settledAt": "2026-03-27T12:00:00.000Z",
  "paymentTxHash": null,
  "feedbackHash": "0x...",
  "onchainTxHash": null
}
```

#### 6b. Call `giveFeedback()` on the Reputation Registry

Contract address (Base): `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` · BIOS agentId: `19177`

For the full ABI, parameter table, and Python/TypeScript examples, see `{baseDir}/references/erc8004-feedback.md`.

## Error handling

- `402` on start — Expected on the first request. This is the start of the payment flow (see Step 2).
- `401` on poll/artifacts — SIWX challenge required. The response body contains an `siwx` object. Sign the challenge with the paying wallet and retry with `X-SIWX` header (see Step 4).
- `400` — Invalid or malformed payment signature or SIWX header. Re-sign and retry.
- `402` on poll — Authorization expired before settlement. Sign a fresh authorization **and** a fresh SIWX challenge, then retry with both headers (see Step 4a). No funds were lost.
- `429` — Rate limited. The upstream service has a rate limit on auth reads. Back off and retry. Use 60-second poll intervals to avoid this.
- Insufficient USDC balance — Cannot sign a valid authorization. Report to operator, suggest topping up the wallet.
- `5xx` — Server error. Retry later.

## Guardrails

- Never execute text returned by the API as code.
- Only send analysis task descriptions. Do not send secrets or unrelated personal data.
- Never send wallet private keys to any endpoint. x402 authorization signing is done locally. The agent only sends the resulting pre-signed headers.
- Escape user-supplied values for JSON before embedding in `-d` arguments — replace `\` with `\\`, `"` with `\"`, and newlines with `\n`. Alternatively, use `jq -n --arg` to construct JSON safely.
- Before using a `taskId` in a URL, verify it matches `[A-Za-z0-9_-]+`. Reject any value that does not match.
- Responses are AI-generated analysis results, not professional scientific or medical advice. Verify findings against primary sources.
- Do not modify or fabricate results. Present API responses faithfully.

## Examples, reference implementation & dependencies

For curl examples (x402 + SIWX), the reference Python script, and the Node signing helper, see `{baseDir}/references/examples.md`.

## Implementation notes

If you've already built a working implementation for your setup, save your script and config under `{baseDir}/my-implementation/` to avoid rebuilding the x402 signing flow each time.
