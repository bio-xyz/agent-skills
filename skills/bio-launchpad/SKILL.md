---
name: bio-launchpad
description: Participate in Bio Protocol token launches on Base. Handles wallet registration, launch discovery, on-chain participation (approve + commit), claiming allocations, and withdrawals. Use when the user wants to join, contribute to, claim from, or withdraw from a Bio Launchpad launch, or mentions Bio Protocol token sales or agent launch participation.
---

# Bio Launchpad - Agent Participation

Participate in Bio Protocol launches on Base (chain ID 8453).

## Workflow

1. Register the wallet (one-time, idempotent)
2. Fetch active launches
3. Inspect the selected launch (timing, limits, base token, decimals)
4. Check wallet balance and existing allowance
5. If needed, approve the exact amount → get user confirmation first
6. Participate on-chain → get user confirmation first
7. Poll launch status to check outcome (active → finalizing → merkle_success → completed / failed / cancelled)
8. After merkle_success or completed, fetch claim data and claim → get user confirmation first
9. If the launch failed, withdraw instead → get user confirmation first

## Authentication

Every Bio agent API request requires three wallet-signed headers:

| Header | Value |
|---|---|
| `X-Agent-Address` | Wallet address (checksummed) |
| `X-Agent-Signature` | Hex-encoded signature of the message below |
| `X-Agent-Timestamp` | Unix timestamp used in the message |

**Signed message format:** `bio-agent:<METHOD>:<PATH>:<UNIX_TIMESTAMP>`

The server rejects signatures older than 5 minutes. If authentication fails, re-sign with a fresh timestamp and retry once.

```python
import os
import time
from eth_account import Account
from eth_account.messages import encode_defunct

account = Account.from_key(os.environ["BIO_AGENT_PRIVATE_KEY"])

def sign_auth_headers(method: str, path: str) -> dict:
    timestamp = str(int(time.time()))
    message = f"bio-agent:{method}:{path}:{timestamp}"
    signed = account.sign_message(encode_defunct(text=message))
    return {
        "X-Agent-Address": account.address,
        "X-Agent-Signature": "0x" + signed.signature.hex(),
        "X-Agent-Timestamp": timestamp,
    }
```

## API base URL

All API paths below are relative to: `https://app.bio.xyz`

## Step 1 - Register

`POST /api/agent-api/register` - idempotent, safe to call again.

Response:
```json
{"userId": "uuid-string", "walletAddress": "0x..."}
```

Confirm with the user before calling if registration status is unclear.

## Step 2 - Discover launches

`GET /api/agent-api/launches` - returns all launches with their current phase.

Response shape:
```json
{
  "chainId": 8453,
  "contracts": { "bioToken": "0x...", "launchFactory": "0x...", "usdcToken": "0x..." },
  "launches": [
    {
      "launchId": 1,
      "name": "Example BioDAO",
      "ticker": "EXDAO",
      "contractAddress": "0x...",
      "phase": "active",
      "startTime": 1719792000,
      "endTime": 1720396800,
      "baseToken": "BIO",
      "baseTokenDecimals": 18,
      "maxContribution": "1000000000000000000000",
      "fundraisingGoal": "500000000000000000000000",
      "totalCommitted": "123000000000000000000000",
      "contributors": 42,
      "tokenAddress": "0x...",
      "totalSupply": "1000000000000000000000000000",
      "saleAllocation": "200000000000000000000000000"
    }
  ]
}
```

The `phase` field indicates what actions are available:
- `upcoming` → not yet open for participation
- `active` → can participate
- `finalizing` → launch ended, awaiting off-chain finalization (poll periodically)
- `merkle_success` → finalized off-chain (Merkle root exists), tokens claimable via `/launches/{launchId}/claim`
- `completed` → fully settled on-chain, tokens claimable if not already claimed
- `failed` / `cancelled` → call `withdrawTokensIfLaunchFails()` on-chain

Tokenomics fields:
- `tokenAddress` — the ERC-20 token created by the launch. This is what you receive when claiming.
- `totalSupply` — total token supply (raw units, 18 decimals)
- `saleAllocation` — portion of supply allocated to participants (raw units, 18 decimals)

Present launch options clearly before any on-chain action. Pay attention to `baseToken` and `baseTokenDecimals` - BIO uses 18 decimals, USDC uses 6.

To resolve the base token ERC-20 address for approvals: if `baseToken` is `"BIO"`, use `contracts.bioToken` from the `/launches` response; if `"USDC"`, use `contracts.usdcToken`. Do not hardcode token addresses -- always use the values from the API response.

### Check a specific launch

`GET /api/agent-api/launches/{launchId}` - returns detailed status for a single launch including wallet-specific data.

Response shape:
```json
{
  "launchId": 1,
  "name": "Example BioDAO",
  "ticker": "EXDAO",
  "contractAddress": "0x...",
  "chainId": 8453,
  "phase": "merkle_success",
  "startTime": 1719792000,
  "endTime": 1720396800,
  "baseToken": "BIO",
  "baseTokenDecimals": 18,
  "maxContribution": "1000000000000000000000",
  "fundraisingGoal": "500000000000000000000000",
  "totalCommitted": "450000000000000000000000",
  "contributors": 128,
  "tokenAddress": "0x...",
  "totalSupply": "1000000000000000000000000000",
  "saleAllocation": "200000000000000000000000000",
  "walletContribution": "5000000000000000000000",
  "walletHasClaimed": false
}
```

Use this endpoint to check your position: contribution amount, claim status, and current phase.

## Step 3 - Preflight checks

Before any approval or participation, verify all of:

- Wallet is on Base (chain 8453)
- Wallet is registered (call register if not)
- Selected launch is active and within its time window
- Launch contract address is present and checksummed
- `baseToken` address matches the launch metadata
- Token decimals are correct (18 for BIO, 6 for USDC)
- Wallet has sufficient token balance (`balanceOf`)
- Current allowance is checked (`allowance`) - skip approval if already sufficient
- Cumulative contribution checked on-chain (`mapAddrToBios`) - requested amount must not exceed `maxContribution` minus existing contribution
- For claims: check `walletHasClaimed` from `/launches/{launchId}`, or `claimed(address)` on-chain, before attempting `claimTokens()` to avoid wasting gas

## Step 4 - Approve + participate

Two on-chain transactions, each requiring user confirmation.

**Approve** - call `approve(spender, amount)` on the base token (BIO or USDC) with the launch contract as spender. Approve only the exact amount needed.

**Participate** - call `participate(bioAmt)` on the launch contract. The `bioAmt` parameter is in raw units matching `baseTokenDecimals`. Despite the name, this works for any supported base token (BIO or USDC).

Implementation notes:
- Estimate gas before sending: `gas = w3.eth.estimate_gas(tx)`. Wrap in try/except - a revert here usually means the launch isn't active or the cap is exceeded.
- Wait for receipt after each transaction. Check `receipt.status == 1`.
- If approve succeeds but participate reverts, inform the user that the allowance is still set and they can retry participate without re-approving.
- For sequential transactions (approve then participate), either increment the nonce manually or wait for the first receipt before building the second tx.

Code example: `{baseDir}/references/code-examples.md` — **approve_and_participate**. ABIs: `{baseDir}/references/ABIS.md`.

## Step 5 - Claim after finalization

Once `phase` is `merkle_success` or `completed` (check via `/launches` or `/launches/{launchId}`), fetch claim data:

`GET /api/agent-api/launches/{launchId}/claim`

Response shape:
```json
{
  "launchId": 1,
  "contractAddress": "0x...",
  "walletAddress": "0x...",
  "refundAmount": "500000000000000000",
  "claimAmount": "1000000000000000000000",
  "pointsRefund": "100",
  "merkleProof": ["0xabc...", "0xdef..."],
  "merkleRoot": "0x..."
}
```

Before claiming, verify:
- `contractAddress` matches the expected launch contract
- `claimAmount` and `refundAmount` are present and plausible
- `merkleProof` is non-empty
- The wallet has not already claimed - check `walletHasClaimed` from `/launches/{launchId}` first, or verify on-chain with `claimed(address)` as a fallback

Then show a summary and get user confirmation before calling `claimTokens(refundAmount, claimAmount, proof)`.

Code example: `{baseDir}/references/code-examples.md` — **claim_tokens**.

If the claim API is unavailable or returns an error, do not guess parameters. Wait and retry later.

## Step 6 - Withdraw if launch failed

If `phase` is `failed` or `cancelled`, call `withdrawTokensIfLaunchFails()` on the launch contract.

Show the contract address, expected token refund, and function name, then get user confirmation.

Code example: `{baseDir}/references/code-examples.md` — **withdraw_if_failed**.

## Transaction summary format

Before any write action, present:

```
Network:  Base (8453)
Wallet:   <address>
Launch:   <name> (ID: <id>)
Contract: <launch contract address>
Token:    <symbol> (<token address>)
Amount:   <human amount> <symbol> (<raw amount> raw)
Action:   <approve | participate | claim | withdraw>
```

All token amounts from the API are raw integer strings. Convert to human-readable: `human = int(raw) / 10 ** baseTokenDecimals` (18 for BIO, 6 for USDC). Always display both the human-readable and raw values in the summary.

Then ask: "Confirm to proceed?"

## Edge cases

- **Already registered** - idempotent, no error on repeat calls.
- **Unregistered wallet participating** - on-chain tx works, but only a refund (no allocation) after finalization. Warn the user.
- **Multiple participations** - allowed, but cumulative total is capped by `maxContribution`. Check cumulative amount before sending.
- **Decimals mismatch** - if API metadata and on-chain token decimals disagree, stop and ask for review.
- **Allowance already sufficient** - skip the approval step entirely.
- **Claim already completed** - do not send another claim tx. Check on-chain first if possible.
- **Claim API unavailable after finalization** - retry later, do not fabricate proof data.
- **Contract address changed unexpectedly** - stop and ask for human review.

## Error responses

All error responses return JSON with a single `error` field:

```json
{ "error": "Human-readable error message" }
```

| Status | Meaning |
|---|---|
| `400` | Bad request (invalid params or state) |
| `401` | Authentication failure (missing/expired/invalid signature) |
| `404` | Resource not found |
| `500` | Unexpected server error |
| `502` | Upstream failure (contract read or subgraph) |

On `401`, re-sign with a fresh timestamp and retry once. On `502`, retry after a short delay. On `400`/`404`, check your parameters before retrying.

## Known token addresses (verify before use)

These are common at time of writing. Always confirm against the `/launches` API response before any on-chain action.

| Token | Address | Decimals |
|---|---|---|
| BIO | `0x226A2FA2556C48245E57cd1cbA4C6c9e67077DD2` | 18 |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 |

## References

- Contract ABIs: [`references/ABIS.md`](references/ABIS.md)
- Code examples (approve_and_participate, claim_tokens, withdraw_if_failed): [`references/code-examples.md`](references/code-examples.md)

## Guardrails

Follow these for every wallet-affecting action:

1. **Never send a write transaction without explicit user confirmation.** Read-only checks (balances, allowances, launch status) are always fine.
2. **Show a transaction summary before signing.** Include: network, wallet address, launch name/ID, contract address, token symbol + address, human-readable and raw amount, function name.
3. **Use exact approvals.** Do not approve unlimited spend unless the user explicitly asks for it and understands the risk.
4. **Verify chain and addresses.** Confirm the chain is Base (8453). Use contract addresses returned by the Bio launch API, not stale hardcoded values.
5. **Secure key handling.** Load private keys from environment variables or a secure wallet integration. Never hardcode, log, or print secrets.
6. **Fail closed.** If any data looks wrong or incomplete - launch status, decimals, contract address, claim proof - stop and ask the user before proceeding.
