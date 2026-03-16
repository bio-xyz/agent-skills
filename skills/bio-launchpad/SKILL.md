---
name: bio-launchpad
description: Participate in Bio Protocol launches on Base. Handles wallet registration, launch discovery, on-chain participation (approve + commit), claiming allocations, and withdrawals. Use when the user wants to join, contribute to, claim from, or withdraw from a Bio Launchpad launch on Base, or when they mention Bio Protocol token sales, Bio launchpad, or agent launch participation.
metadata:
  author: bio-protocol
  version: "1.0"
compatibility: Requires network access, Python 3.10+, web3.py, and an EVM wallet with ETH (gas), BIO, or USDC on Base.
---

# Bio Launchpad - Agent Participation

Participate in Bio Protocol launches on Base (chain ID 8453).

## Safety rules

Follow these for every wallet-affecting action:

1. **Never send a write transaction without explicit user confirmation.** Read-only checks (balances, allowances, launch status) are always fine.
2. **Show a transaction summary before signing.** Include: network, wallet address, launch name/ID, contract address, token symbol + address, human-readable and raw amount, function name.
3. **Use exact approvals.** Do not approve unlimited spend unless the user explicitly asks for it and understands the risk.
4. **Verify chain and addresses.** Confirm the chain is Base (8453). Use contract addresses returned by the Bio launch API, not stale hardcoded values.
5. **Secure key handling.** Load private keys from environment variables or a secure wallet integration. Never hardcode, log, or print secrets.
6. **Fail closed.** If any data looks wrong or incomplete - launch status, decimals, contract address, claim proof - stop and ask the user before proceeding.

## Workflow

1. Register the wallet (one-time, idempotent)
2. Fetch active launches
3. Inspect the selected launch (timing, limits, base token, decimals)
4. Check wallet balance and existing allowance
5. If needed, approve the exact amount → get user confirmation first
6. Participate on-chain → get user confirmation first
7. After finalization, fetch claim data and claim → get user confirmation first
8. If the launch failed, withdraw instead → get user confirmation first

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
        "X-Agent-Signature": signed.signature.hex(),
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

`GET /api/agent-api/launches` - returns active launches.

Response shape:
```json
{
  "chainId": 8453,
  "contracts": {},
  "launches": [
    {
      "launchId": 1,
      "name": "Example BioDAO",
      "ticker": "EXDAO",
      "contractAddress": "0x...",
      "startTime": "2025-07-01T00:00:00Z",
      "endTime": "2025-07-08T00:00:00Z",
      "baseToken": "0x...BIO or USDC address...",
      "baseTokenDecimals": 18,
      "maxContribution": "1000000000000000000000",
      "reserve": "...",
      "started": true
    }
  ]
}
```

Present launch options clearly before any on-chain action. Pay attention to `baseToken` and `baseTokenDecimals` - BIO uses 18 decimals, USDC uses 6.

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
- For claims: check `claimed(address)` on-chain before attempting `claimTokens()` to avoid wasting gas

## Step 4 - Approve + participate

Two on-chain transactions, each requiring user confirmation.

**Approve** - call `approve(spender, amount)` on the base token (BIO or USDC) with the launch contract as spender. Approve only the exact amount needed.

**Participate** - call `participate(bioAmt)` on the launch contract. The `bioAmt` parameter is in raw units matching `baseTokenDecimals`. Despite the name, this works for any supported base token (BIO or USDC).

Implementation notes:
- Estimate gas before sending: `gas = w3.eth.estimate_gas(tx)`. Wrap in try/except - a revert here usually means the launch isn't active or the cap is exceeded.
- Wait for receipt after each transaction. Check `receipt.status == 1`.
- If approve succeeds but participate reverts, inform the user that the allowance is still set and they can retry participate without re-approving.
- For sequential transactions (approve then participate), either increment the nonce manually or wait for the first receipt before building the second tx.

```python
def approve_and_participate(launch_contract: str, base_token: str, amount: int, max_contribution: int):
    launch_contract = Web3.to_checksum_address(launch_contract)
    base_token = Web3.to_checksum_address(base_token)

    # Preflight: check cumulative contribution against cap
    contract = w3.eth.contract(address=launch_contract, abi=LAUNCH_ABI)
    current = contract.functions.mapAddrToBios(ADDRESS).call()
    if current + amount > max_contribution:
        raise ValueError(f"Would exceed max contribution: {current} existing + {amount} new > {max_contribution}")

    # Approve
    token = w3.eth.contract(address=base_token, abi=ERC20_ABI)
    approve_tx = token.functions.approve(launch_contract, amount).build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    approve_tx["gas"] = w3.eth.estimate_gas(approve_tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(approve_tx).raw_transaction)
    )
    assert receipt.status == 1, "Approve transaction reverted"

    # Participate
    participate_tx = contract.functions.participate(amount).build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    participate_tx["gas"] = w3.eth.estimate_gas(participate_tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(participate_tx).raw_transaction)
    )
    assert receipt.status == 1, "Participate transaction reverted"
```

ABI variables (`ERC20_ABI`, `LAUNCH_ABI`) are defined in `references/ABIS.md`.

## Step 5 - Claim after finalization

After the launch ends and is finalized off-chain by the platform, fetch claim data:

`GET /api/agent-api/launches/{launchId}/claim`

Response shape:
```json
{
  "contractAddress": "0x...",
  "refundAmount": "500000000000000000",
  "claimAmount": "1000000000000000000000",
  "merkleProof": ["0xabc...", "0xdef..."]
}
```

Before claiming, verify:
- `contractAddress` matches the expected launch contract
- `claimAmount` and `refundAmount` are present and plausible
- `merkleProof` is non-empty
- The wallet has not already claimed - call `claimed(address)` on-chain first

Then show a summary and get user confirmation before calling `claimTokens(refundAmount, claimAmount, proof)`.

```python
def claim_tokens(launch_id: int):
    path = f"/api/agent-api/launches/{launch_id}/claim"
    resp = requests.get(BASE_URL + path, headers=sign_auth_headers("GET", path))
    resp.raise_for_status()
    data = resp.json()

    contract_addr = Web3.to_checksum_address(data["contractAddress"])
    contract = w3.eth.contract(address=contract_addr, abi=LAUNCH_ABI)

    # Check if already claimed
    if contract.functions.claimed(ADDRESS).call():
        raise ValueError("Wallet has already claimed for this launch")

    proof = [bytes.fromhex(p.removeprefix("0x")) for p in data["merkleProof"]]
    tx = contract.functions.claimTokens(
        int(data["refundAmount"]), int(data["claimAmount"]), proof
    ).build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    tx["gas"] = w3.eth.estimate_gas(tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )
    assert receipt.status == 1, "Claim transaction reverted"
```

If the claim API is unavailable or returns an error, do not guess parameters. Wait and retry later.

## Step 6 - Withdraw if launch failed

If the launch failed or was cancelled, call `withdrawTokensIfLaunchFails()` on the launch contract.

Confirm the launch status before withdrawing. Show the contract address, expected token refund, and function name, then get user confirmation.

```python
def withdraw_if_failed(launch_contract: str):
    contract = w3.eth.contract(
        address=Web3.to_checksum_address(launch_contract),
        abi=LAUNCH_ABI,
    )
    tx = contract.functions.withdrawTokensIfLaunchFails().build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    tx["gas"] = w3.eth.estimate_gas(tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )
    assert receipt.status == 1, "Withdraw transaction reverted"
```

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

## Known token addresses (verify before use)

These are common at time of writing. Always confirm against the `/launches` API response before any on-chain action.

| Token | Address | Decimals |
|---|---|---|
| BIO | `0x226A2FA2556C48245E57cd1cbA4C6c9e67077DD2` | 18 |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 |

## References

- Full contract ABIs: [`references/ABIS.md`](references/ABIS.md)
