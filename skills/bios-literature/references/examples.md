# Examples & Dependencies

## Minimal curl examples

### Step 1: Start literature query — fast mode (get 402)

```bash
curl -s -X POST https://x402.ai.bio.xyz/api/agents/literature/query \
  -H "Content-Type: application/json" \
  -d '{"question":"What are the latest advances in CRISPR gene therapy for sickle cell disease?","mode":"fast"}'
# → 402 with x402Version:2 payment requirements in JSON body
```

### Step 1 (alt): Start literature query — deep mode (get 402)

```bash
curl -s -X POST https://x402.ai.bio.xyz/api/agents/literature/query \
  -H "Content-Type: application/json" \
  -d '{"question":"What are the latest advances in CRISPR gene therapy for sickle cell disease?","mode":"deep","sources":["pubmed","arxiv"]}'
# → 402 with x402Version:2 payment requirements in JSON body ($0.15 for deep)
```

### Step 2: Sign and resubmit (language-specific signing, see SKILL.md Step 2)

```bash
curl -s -X POST https://x402.ai.bio.xyz/api/agents/literature/query \
  -H "Content-Type: application/json" \
  -H "PAYMENT-SIGNATURE: <base64-encoded x402 v2 payment payload>" \
  -d '{"question":"What are the latest advances in CRISPR gene therapy for sickle cell disease?","mode":"fast"}'
# Fast mode → 200 with { answer, formattedAnswer, references, contextPassages, reasoning, status: "completed" }
# Deep mode → 200 with { jobId, status: "queued" }
```

### Step 3: Poll — requires SIWX auth (deep mode only, cannot use plain curl)

The poll endpoint returns 401 with an SIWX challenge that must be signed by the paying wallet. Use the reference Python/Node scripts instead. See `{baseDir}/references/siwx-protocol.md` for the full protocol.

### Step 4 (optional): Fetch feedback data for on-chain submission

```bash
curl -s https://x402.ai.bio.xyz/api/feedback/{jobId}
```

## Full flow with SIWX (TypeScript)

The poll endpoint (deep mode) requires SIWX authentication. Here is the complete TypeScript flow covering both modes:

```typescript
import { createSiweMessage } from "viem/siwe";
import { x402Client } from "@x402/core/client";
import { registerExactEvmScheme } from "@x402/evm/exact/client";
import { toClientEvmSigner } from "@x402/evm";
import { encodePaymentSignatureHeader } from "@x402/core/http";

const BASE = "https://x402.ai.bio.xyz";

type SiwxChallenge = {
  domain: string;
  uri: string;
  nonce: string;
  chainId: number;
  statement: string;
  version: "1";
  issuedAt: string;
  expirationTime: string;
};

// --- SIWX helper ---

async function signSiwxChallenge(challenge: SiwxChallenge, account: LocalAccount): Promise<string> {
  const message = createSiweMessage({
    address: account.address,
    domain: challenge.domain,
    uri: challenge.uri,
    nonce: challenge.nonce,
    chainId: challenge.chainId,
    statement: challenge.statement,
    version: challenge.version,
    issuedAt: new Date(challenge.issuedAt),
    expirationTime: new Date(challenge.expirationTime),
  });
  const signature = await account.signMessage({ message });
  return Buffer.from(JSON.stringify({ message, signature })).toString("base64");
}

// --- Steps 1–2: Start literature query with x402 payment ---

async function startLiteratureQuery(
  question: string,
  mode: "fast" | "deep",
  account: LocalAccount,
  publicClient: PublicClient,
  options?: { maxResults?: number; perSourceLimit?: number; sources?: string[] },
): Promise<any> {
  const body: Record<string, unknown> = { question, mode };
  if (options?.maxResults) body.maxResults = options.maxResults;
  if (options?.perSourceLimit) body.perSourceLimit = options.perSourceLimit;
  if (options?.sources) body.sources = options.sources;

  // Step 1: Get 402
  const res402 = await fetch(`${BASE}/api/agents/literature/query`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  if (res402.status !== 402) throw new Error(`Expected 402, got ${res402.status}`);

  // Step 2: Sign and resubmit
  const paymentRequired = await res402.json();
  const client = new x402Client();
  const signer = toClientEvmSigner(account, publicClient);
  registerExactEvmScheme(client, { signer });

  const paymentPayload = await client.createPaymentPayload(paymentRequired);
  const paymentSig = encodePaymentSignatureHeader(paymentPayload);

  const res200 = await fetch(`${BASE}/api/agents/literature/query`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "PAYMENT-SIGNATURE": paymentSig,
    },
    body: JSON.stringify(body),
  });
  if (!res200.ok) throw new Error(`Paid start failed: ${res200.status}`);
  return res200.json();
}

// --- Step 2a: Fast mode — results already in the response ---

// After startLiteratureQuery with mode "fast", the response contains:
// { answer, formattedAnswer, references, contextPassages, reasoning, status: "completed" }
// No polling needed. You're done.

// --- Steps 2b + 3: Deep mode — poll with SIWX ---

async function pollForResults(jobId: string, account: LocalAccount): Promise<any> {
  const url = `${BASE}/api/agents/literature/jobs/${jobId}`;

  // Wait 30s before the first poll
  await new Promise((r) => setTimeout(r, 30_000));

  while (true) {
    // Get SIWX challenge
    let res = await fetch(url);

    if (res.status === 401) {
      const { siwx } = await res.json();
      const siwxHeader = await signSiwxChallenge(siwx, account);
      res = await fetch(url, { headers: { "X-SIWX": siwxHeader } });
    }

    // Handle timeout retry (402 after SIWX)
    if (res.status === 402) {
      const paymentRequired = await res.json();
      const paymentSig = /* sign fresh x402 payment from paymentRequired */;

      // Fetch fresh SIWX challenge
      const challengeRes = await fetch(url);
      const { siwx } = await challengeRes.json();
      const siwxHeader = await signSiwxChallenge(siwx, account);

      res = await fetch(url, {
        headers: { "X-SIWX": siwxHeader, "PAYMENT-SIGNATURE": paymentSig },
      });
    }

    if (res.status === 429) {
      const retryAfter = res.headers.get("retry-after");
      const waitSec = retryAfter ? parseInt(retryAfter, 10) : 60;
      await new Promise((r) => setTimeout(r, waitSec * 1000));
      continue;
    }

    const data = await res.json();

    if (data.status === "completed") return data;
    if (data.status === "failed") throw new Error(`Literature search failed: ${JSON.stringify(data.data)}`);
    if (data.status === "queued") {
      await new Promise((r) => setTimeout(r, 180_000)); // poll every 3 min
      continue;
    }

    throw new Error(`Unexpected status: ${data.status}`);
  }
}

// --- Usage ---

async function main() {
  // Fast mode — synchronous, no polling
  const fastResult = await startLiteratureQuery(
    "What are the latest CRISPR therapies?",
    "fast",
    account,
    publicClient,
  );
  console.log("Fast result:", fastResult.answer);

  // Deep mode — async, requires polling
  const deepResult = await startLiteratureQuery(
    "Comprehensive review of mRNA vaccine platforms",
    "deep",
    account,
    publicClient,
    { sources: ["pubmed", "arxiv"] },
  );
  console.log("Job ID:", deepResult.jobId);

  const completed = await pollForResults(deepResult.jobId, account);
  console.log("Deep result:", completed.answer);
  console.log("References:", completed.references);
}
```

## Reference implementation (x402)

A Python script for the x402 path is at `{baseDir}/references/python-script.md`. It delegates EIP-712 signing to a Node helper (same pattern as the deep-research skill).

These scripts handle x402 payment negotiation, SIWX poll authentication for deep mode, and both fast/deep result flows.

Usage:

```bash
# Fast mode — synchronous results (default)
export WALLET_PRIVATE_KEY=0x...
python3 literature.py "What are the latest CRISPR therapies?"

# Deep mode — async with polling
python3 literature.py --mode deep "Comprehensive review of mRNA vaccine platforms"

# Deep mode with source filtering
python3 literature.py --mode deep --sources pubmed,arxiv "mRNA vaccine platforms"

# Dry run (no payment)
python3 literature.py --dry-run "test"

# Submit without polling (deep mode only)
python3 literature.py --mode deep --no-poll "Analyze this literature"
```

## Dependencies

**TypeScript/Node:**
- `@x402/core` — x402 v2 client
- `@x402/evm` — EVM signer for x402
- `viem` — Ethereum client (SIWE, accounts, chains)

**Python:**
- `eth-account` — EIP-191 signing for SIWX
- `node` + `npm` — Required for the x402 signing helper (Node subprocess)
- `@x402/core`, `@x402/evm`, `viem` — Installed via npm by the helper
