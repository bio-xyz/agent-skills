# Examples & Dependencies

## Minimal curl examples

### Step 1: Upload a file (optional)

```bash
curl -sS -X POST https://x402.ai.bio.xyz/api/files/upload \
  -F "file=@/path/to/gene-expression.csv"
# → 200 with { fileId, filename, contentType, size, createdAt }
```

### Step 2: Start analysis (get 402)

```bash
curl -s -X POST https://x402.ai.bio.xyz/api/data-analysis/start \
  -H "Content-Type: application/json" \
  -d '{"taskDescription":"Analyze gene expression patterns","fileIds":["FILE_ID_HERE"]}'
# → 402 with x402Version:2 payment requirements in JSON body
```

### Step 3: Sign and resubmit (language-specific signing, see SKILL.md Step 3)

```bash
curl -s -X POST https://x402.ai.bio.xyz/api/data-analysis/start \
  -H "Content-Type: application/json" \
  -H "PAYMENT-SIGNATURE: <base64-encoded x402 v2 payment payload>" \
  -d '{"taskDescription":"Analyze gene expression patterns","fileIds":["FILE_ID_HERE"]}'
# → 200 with { taskId, status: "queued" }
```

### Step 4: Poll — requires SIWX auth (cannot use plain curl)

The poll endpoint returns 401 with an SIWX challenge that must be signed by the paying wallet. Use the reference Python/Node scripts instead. See `{baseDir}/references/siwx-protocol.md` for the full protocol.

### Step 5: Download artifacts — requires SIWX auth

Same SIWX challenge-response as polling. The endpoint returns a pre-signed download URL:

```json
{ "url": "https://storage.example.com/signed-url?token=...", "expiresIn": 3600 }
```

Then download the file:

```bash
curl -sS -o results.csv "SIGNED_URL_FROM_RESPONSE"
```

### Step 6 (optional): Fetch feedback data for on-chain submission

```bash
curl -s https://x402.ai.bio.xyz/api/feedback/{taskId}
```

## Full flow with SIWX (TypeScript)

The poll and artifact endpoints require SIWX authentication. Here is the complete TypeScript flow:

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

// --- Step 1: Upload file (optional) ---

async function uploadFile(filePath: string): Promise<string> {
  const fs = await import("node:fs");
  const path = await import("node:path");
  const fileBuffer = fs.readFileSync(filePath);
  const fileName = path.basename(filePath);

  const formData = new FormData();
  formData.append("file", new Blob([fileBuffer]), fileName);

  const res = await fetch(`${BASE}/api/files/upload`, {
    method: "POST",
    body: formData,
  });
  if (!res.ok) throw new Error(`Upload failed: ${res.status}`);
  const data = await res.json();
  return data.fileId;
}

// --- Steps 2–3: Start analysis with x402 payment ---

async function startAnalysis(
  taskDescription: string,
  fileIds: string[],
  account: LocalAccount,
  publicClient: PublicClient,
): Promise<string> {
  const body = { taskDescription, fileIds: fileIds.length ? fileIds : undefined };

  // Step 2: Get 402
  const res402 = await fetch(`${BASE}/api/data-analysis/start`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  if (res402.status !== 402) throw new Error(`Expected 402, got ${res402.status}`);

  // Step 3: Sign and resubmit
  const paymentRequired = await res402.json();
  const client = new x402Client();
  const signer = toClientEvmSigner(account, publicClient);
  registerExactEvmScheme(client, { signer });

  const paymentPayload = await client.createPaymentPayload(paymentRequired);
  const paymentSig = encodePaymentSignatureHeader(paymentPayload);

  const res200 = await fetch(`${BASE}/api/data-analysis/start`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "PAYMENT-SIGNATURE": paymentSig,
    },
    body: JSON.stringify(body),
  });
  if (!res200.ok) throw new Error(`Paid start failed: ${res200.status}`);
  const { taskId } = await res200.json();
  return taskId;
}

// --- Step 4: Poll with SIWX ---

async function pollForResults(taskId: string, account: LocalAccount): Promise<any> {
  const url = `${BASE}/api/data-analysis/${taskId}`;

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
    if (data.status === "failed") throw new Error(`Analysis failed: ${JSON.stringify(data.data)}`);
    if (data.status === "queued") {
      await new Promise((r) => setTimeout(r, 60_000)); // poll every 60s
      continue;
    }

    throw new Error(`Unexpected status: ${data.status}`);
  }
}

// --- Step 5: Download artifacts with SIWX ---

async function downloadArtifact(
  taskId: string,
  artifactId: string,
  account: LocalAccount,
): Promise<{ url: string; expiresIn: number }> {
  const url = `${BASE}/api/data-analysis/${taskId}/artifacts/${artifactId}`;

  // Get SIWX challenge
  let res = await fetch(url);
  if (res.status === 401) {
    const { siwx } = await res.json();
    const siwxHeader = await signSiwxChallenge(siwx, account);
    res = await fetch(url, { headers: { "X-SIWX": siwxHeader } });
  }

  if (!res.ok) throw new Error(`Artifact download failed: ${res.status}`);
  return res.json();
}
```

## Reference implementation (x402)

A Python script for the x402 path is at `{baseDir}/references/python-script.md`. It delegates EIP-712 signing to a Node helper (same pattern as the deep-research skill).

These scripts handle x402 payment negotiation, SIWX poll authentication, and artifact downloads.

Usage:

```bash
# Private key wallet — analyze with a local file
export WALLET_PRIVATE_KEY=0x...
python3 analysis.py "Analyze gene expression patterns" --file /path/to/data.csv

# Private key wallet — analyze without file upload
python3 analysis.py "Summarize public COVID-19 case trends"

# Dry run (no payment)
python3 analysis.py --dry-run "test"

# Submit without polling
python3 analysis.py --no-poll "Analyze this dataset"
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
