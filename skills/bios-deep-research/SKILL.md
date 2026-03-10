---
name: bios-deep-research
description: Run paid deep research queries on BIOS via the x402 payment protocol. Uses USDC on Base for micro-payments.
---

# BIOS Deep Research (x402)

Query the BIOS deep research API, paying per-request with USDC on Base via the x402 payment protocol.

## Overview

BIOS provides an AI-powered research service behind a paywall using the [x402 protocol](https://x402.org/). You send a research query, receive a `402 Payment Required` response, sign a USDC payment authorization, and resubmit. Then you poll until results are ready.

No tokens leave your wallet until the server successfully delivers results. The payment is an EIP-712 signature (an authorization), and settlement happens server-side via a facilitator only after delivery.

## Pricing

| Mode | Cost (USDC) | Typical wait |
|------|-------------|--------------|
| `steering` | $0.20 | ~5 min |
| `smart` | $1.00 | ~15 min |
| `fully-autonomous` | $8.00 | ~60+ min |

## Prerequisites

- A wallet on **Base** (chain ID 8453) that can sign EIP-712 typed data
- USDC balance on Base sufficient for the chosen mode
- HTTP client (curl, Python httpx/requests, Node fetch, etc.)
- **(Optional, for feedback only)** A small ETH balance on Base for gas (~$0.001 per feedback transaction)

### Wallet options

Any wallet that can produce EIP-712 signatures works. Common setups:

| Method | How | Best for |
|--------|-----|----------|
| **Coinbase Agentic Wallet** | [`npx awal`](https://docs.cdp.coinbase.com/agentic-wallet/welcome) - CLI with built-in x402 support. Handles 402 detection, EIP-712 signing, and payment in one command. No private key management; keys live in Coinbase's TEE. | AI agents and automated workflows |
| **Private key** | Sign locally with ethers.js, web3.py, eth_account, viem, etc. | Scripts and backends where you manage your own keys |
| **Coinbase CDP SDK** | Use `cdp-sdk` with `sign_typed_data()` | Programmatic server-side signing via Coinbase infrastructure |
| **Browser wallet** | MetaMask, Rabby, etc. via `eth_signTypedData_v4` | Interactive/manual use |
| **MPC / multisig** | Any signer that produces a valid EIP-712 signature | Custom custody setups |

## Quickstart: Coinbase Agentic Wallet (recommended for agents)

The fastest path for an AI agent. The [`awal` CLI](https://docs.cdp.coinbase.com/agentic-wallet/welcome) handles x402 payment negotiation (steps 1-3 below) in a single command - no manual EIP-712 signing required.

**One-time setup:**

```bash
# Authenticate (email OTP)
npx awal auth login user@example.com
npx awal auth verify <flowId> <otp>

# Confirm wallet is ready and has USDC balance
npx awal status
npx awal balance
```

**Run a research query (handles 402 → sign → pay automatically):**

```bash
npx awal x402 pay https://x402.chat.bio.xyz/api/deep-research/start \
  -X POST \
  -d '{"message":"Your research query here","researchMode":"steering"}' \
  --max-amount 200000
```

This returns a `conversationId`. Then poll for results:

```bash
curl -s https://x402.chat.bio.xyz/api/deep-research/{conversationId}
```

You can also inspect pricing before paying:

```bash
npx awal x402 details https://x402.chat.bio.xyz/api/deep-research/start
```

For the full manual protocol (private key, CDP SDK, or other signers), see below.

## API Protocol (manual)

### Base URL

https://x402.chat.bio.xyz

### Step 1: Send research request → get 402

```
POST /api/deep-research/start
Content-Type: application/json

{
  "message": "Your research query here",
  "researchMode": "steering"           // or "smart" or "fully-autonomous"
}
```

Optional field: `"conversationId": "..."` to continue a previous conversation.

**Response:** `402 Payment Required`

The payment requirements are in the response body and/or the `x-payment-required` header (base64-encoded JSON). Example body:

```json
{
  "x402Version": 1,
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "amount": "200000",
      "payTo": "0x4b4F85C16B488181F863a5e5a392A474B86157e0",
      "maxTimeoutSeconds": 1800
    }
  ],
  "resource": {
    "url": "/api/deep-research/start",
    "description": "BIOS Deep Research",
    "mimeType": "application/json"
  }
}
```

The `amount` is in USDC's smallest unit (6 decimals). `200000` = $0.20.

### Step 2: Sign EIP-712 payment authorization

Using the data from step 1, construct and sign an EIP-712 typed data payload. The x402 "exact" scheme uses this structure:

**Domain:**

```json
{
  "name": "x402",
  "version": "1",
  "chainId": 8453,
  "verifyingContract": "<USDC contract: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913>"
}
```

**Types & message:** Follow the [x402 exact scheme spec](https://x402.org/docs). The `x402` client libraries handle this automatically if you use them.

**Using x402 client libraries (recommended):**

Python:

```python
from x402 import parse_payment_required, x402Client
from x402.mechanisms.evm.exact import register_exact_evm_client

client = x402Client()
register_exact_evm_client(client, signer=your_signer, networks="eip155:8453")

payment_required = parse_payment_required(response_data)
payment_payload = await client.create_payment_payload(payment_required)
```

TypeScript/Node:

```typescript
import { createPaymentHeader } from "@x402/fetch";
// See https://github.com/coinbase/x402 for full examples
```

Your signer just needs to implement: `sign_typed_data(domain, types, primary_type, message) → signature_bytes`

### Step 3: Resubmit with payment header

```
POST /api/deep-research/start
Content-Type: application/json
X-PAYMENT: <base64-encoded payment payload>
PAYMENT-SIGNATURE: <base64-encoded payment payload>
```

Send both `X-PAYMENT` and `PAYMENT-SIGNATURE` headers for compatibility (server may check either).

Encode the payment payload:

```python
from x402.http.utils import encode_payment_signature_header
header_value = encode_payment_signature_header(payment_payload)
```

**Response:** `200 OK`

```json
{
  "conversationId": "abc-123-def",
  "status": "queued"
}
```

### Step 4: Poll for results

```
GET /api/deep-research/{conversationId}
```

Poll every 60 seconds. Status values:
- `queued` / `processing` → keep polling
- `completed` → results are in the response body
- `timeout` → research timed out, stop polling

### Step 5 (optional): Submit feedback to ERC-8004 Reputation Registry

After receiving a `completed` result, you can submit on-chain feedback to the [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) Reputation Registry. This records a public trust signal for the BIOS agent, contributing to its verifiable reputation in the x402 ecosystem.

Feedback submission requires a small amount of ETH on Base for gas. If you skip this step, the research flow still works normally.

**Important:** The feedback submitter must NOT be the BIOS agent owner. Only paying clients can submit feedback (enforced by the contract).

#### 5a. Fetch feedback data from the server

```
GET /api/feedback/{conversationId}
```

Response:

```json
{
  "conversationId": "abc-123-def",
  "success": true,
  "settledAt": "2026-03-03T12:00:00.000Z",
  "paymentTxHash": null,
  "feedbackHash": "0x...",
  "onchainTxHash": null
}
```

#### 5b. Call `giveFeedback()` on the Reputation Registry

**Contract addresses:**

| Network | ReputationRegistry |
|---------|--------------------|
| Base (8453) | `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` |
| Base Sepolia (84532) | `0x8004B663056A597Dffe9eCcC1965A193B7388713` |

**BIOS agentId:** `19177`

**ABI (giveFeedback only):**

```json
[
  {
    "inputs": [
      { "internalType": "uint256", "name": "agentId", "type": "uint256" },
      { "internalType": "int128", "name": "value", "type": "int128" },
      { "internalType": "uint8", "name": "valueDecimals", "type": "uint8" },
      { "internalType": "string", "name": "tag1", "type": "string" },
      { "internalType": "string", "name": "tag2", "type": "string" },
      { "internalType": "string", "name": "endpoint", "type": "string" },
      { "internalType": "string", "name": "feedbackURI", "type": "string" },
      { "internalType": "bytes32", "name": "feedbackHash", "type": "bytes32" }
    ],
    "name": "giveFeedback",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```

**Parameters:**

| Parameter | Value |
|-----------|-------|
| `agentId` | `19177` |
| `value` | `1` (success) |
| `valueDecimals` | `0` |
| `tag1` | `"x402"` |
| `tag2` | The research mode used (e.g. `"steering"`) |
| `endpoint` | `"https://x402.ai.bio.xyz/api/deep-research/start"` |
| `feedbackURI` | `"{baseDir}/api/feedback/{conversationId}"` |
| `feedbackHash` | The `feedbackHash` from step 5a |

**Python example (web3.py):**

```python
from web3 import Web3

REPUTATION_REGISTRY = "0x8004BAa17C55a88189AE136b182e5fdA19dE9b63"
BIOS_AGENT_ID = 19177
ABI = [{"inputs":[{"internalType":"uint256","name":"agentId","type":"uint256"},{"internalType":"int128","name":"value","type":"int128"},{"internalType":"uint8","name":"valueDecimals","type":"uint8"},{"internalType":"string","name":"tag1","type":"string"},{"internalType":"string","name":"tag2","type":"string"},{"internalType":"string","name":"endpoint","type":"string"},{"internalType":"string","name":"feedbackURI","type":"string"},{"internalType":"bytes32","name":"feedbackHash","type":"bytes32"}],"name":"giveFeedback","outputs":[],"stateMutability":"nonpayable","type":"function"}]

w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))
contract = w3.eth.contract(address=REPUTATION_REGISTRY, abi=ABI)

tx = contract.functions.giveFeedback(
    BIOS_AGENT_ID,
    1,                              # value: success
    0,                              # valueDecimals
    "x402",                         # tag1
    research_mode,                  # tag2: "steering", "smart", etc.
    "https://x402.ai.bio.xyz/api/deep-research/start",
    f"{api_base}/api/feedback/{conversation_id}",
    bytes.fromhex(feedback_hash[2:]),  # feedbackHash from step 5a (strip 0x prefix)
).build_transaction({
    "from": account.address,
    "nonce": w3.eth.get_transaction_count(account.address),
})

signed = account.sign_transaction(tx)
tx_hash = w3.eth.send_raw_transaction(signed.raw_transaction)
print(f"Feedback tx: {tx_hash.hex()}")
```

**TypeScript example (viem):**

```typescript
import { createWalletClient, http, parseAbi } from "viem";
import { base } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const REPUTATION_REGISTRY = "0x8004BAa17C55a88189AE136b182e5fdA19dE9b63";
const BIOS_AGENT_ID = 19177n;

const account = privateKeyToAccount(privateKey);
const walletClient = createWalletClient({ account, chain: base, transport: http() });

const txHash = await walletClient.writeContract({
  address: REPUTATION_REGISTRY,
  abi: parseAbi([
    "function giveFeedback(uint256 agentId, int128 value, uint8 valueDecimals, string tag1, string tag2, string endpoint, string feedbackURI, bytes32 feedbackHash)",
  ]),
  functionName: "giveFeedback",
  args: [
    BIOS_AGENT_ID,
    1n,                             // value: success
    0,                              // valueDecimals
    "x402",                         // tag1
    researchMode,                   // tag2
    "https://x402.ai.bio.xyz/api/deep-research/start",
    `${apiBase}/api/feedback/${conversationId}`,
    feedbackHash as `0x${string}`,  // feedbackHash from step 5a
  ],
});

console.log(`Feedback tx: ${txHash}`);
```

## Minimal curl example (manual signing)

```bash
# 1. Get payment requirements
curl -s -X POST https://x402.chat.bio.xyz/api/deep-research/start \
  -H "Content-Type: application/json" \
  -d '{"message":"What is NAD+?","researchMode":"steering"}'
# → 402 with payment requirements in body/headers

# 2. Sign the payment (language-specific, see above)

# 3. Submit with payment
curl -s -X POST https://x402.chat.bio.xyz/api/deep-research/start \
  -H "Content-Type: application/json" \
  -H "X-PAYMENT: <base64-payment>" \
  -H "PAYMENT-SIGNATURE: <base64-payment>" \
  -d '{"message":"What is NAD+?","researchMode":"steering"}'
# → 200 with conversationId

# 4. Poll
curl -s https://x402.chat.bio.xyz/api/deep-research/{conversationId}

# 5. (Optional) Fetch feedback data for on-chain submission
curl -s https://x402.chat.bio.xyz/api/feedback/{conversationId}
```

## Implementation notes

If you've already built a working implementation for your setup, store it so you can reuse it:

```
{baseDir}/my-implementation/
```

Save your script, config, or notes there. This avoids rebuilding the x402 signing flow each time. The `scripts/` directory in this skill contains a reference Python implementation if you need a starting point.

## Reference implementation

A complete Python script is included at `{baseDir}/scripts/research.py`. It supports private key and CDP wallet backends with auto-detection. See the script header for env var configuration.

```bash
# Private key wallet
export WALLET_PRIVATE_KEY=0x...
python3 {baseDir}/scripts/research.py "Your query" --mode steering

# CDP wallet
export CDP_API_KEY_ID=...
export CDP_API_KEY_SECRET=...
export CDP_WALLET_ADDRESS=0x...
python3 {baseDir}/scripts/research.py "Your query" --mode smart

# Dry run (no payment)
python3 {baseDir}/scripts/research.py --dry-run "test"
```

## Dependencies (for reference script)

```bash
pip install x402 httpx nest_asyncio
pip install eth-account      # for private key signing
pip install cdp-sdk           # only if using CDP wallet
pip install web3              # only if submitting ERC-8004 feedback
```
