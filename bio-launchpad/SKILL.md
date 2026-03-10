---
name: bio-launchpad
description: Participate in Bio Protocol launches on Base. Register a wallet, discover active launches, approve and commit tokens on-chain, and claim allocations after finalization. Use when the user wants to participate in a Bio Launchpad launch.
compatibility: Requires network access, Python 3.10+, and an EVM wallet with funds on Base.
metadata:
  author: bio-protocol
  version: "1.0"
---

# Bio Launchpad — Agent Participation

Participate in Bio Protocol launches on Base (chain ID 8453).

Python (web3.py) examples are included as reference code — adapt to your setup.

Requirements: `pip install web3 requests eth-account`

## Lifecycle

1. **Register** — one-time, links your wallet so you receive token allocations
2. **Discover launches** — find active launches with contract addresses
3. **Approve + participate** — approve base token spend, then commit on-chain
4. **Wait** — platform finalizes the launch off-chain after it ends
5. **Claim** — fetch Merkle proof from API, call `claimTokens()` on-chain
6. **Or withdraw** — if launch failed, call `withdrawTokensIfLaunchFails()`

## Setup

```python
import time
import requests
from eth_account import Account
from eth_account.messages import encode_defunct
from web3 import Web3

PRIVATE_KEY = "0x..."
RPC_URL = "https://mainnet.base.org"
BASE_URL = "https://app.bio.xyz"
CHAIN_ID = 8453

BIO_TOKEN = Web3.to_checksum_address("0x226A2FA2556C48245E57cd1cbA4C6c9e67077DD2")
USDC_TOKEN = Web3.to_checksum_address("0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913")

w3 = Web3(Web3.HTTPProvider(RPC_URL))
account = Account.from_key(PRIVATE_KEY)
ADDRESS = account.address
```

## Authentication

Every API request needs three headers signed by your wallet. The server verifies the signature is less than 5 minutes old.

**Message format:** `bio-agent:<METHOD>:<PATH>:<UNIX_TIMESTAMP>`

```python
def sign_auth_headers(method: str, path: str) -> dict:
    timestamp = str(int(time.time()))
    message = f"bio-agent:{method}:{path}:{timestamp}"
    signed = account.sign_message(encode_defunct(text=message))
    return {
        "X-Agent-Address": ADDRESS,
        "X-Agent-Signature": signed.signature.hex(),
        "X-Agent-Timestamp": timestamp,
    }
```

## Step 1 — Register

`POST /api/agent-api/register` — idempotent, safe to call again.

```python
def register():
    path = "/api/agent-api/register"
    resp = requests.post(BASE_URL + path, headers=sign_auth_headers("POST", path))
    resp.raise_for_status()
    return resp.json()  # {"userId": "...", "walletAddress": "0x..."}
```

## Step 2 — Discover launches

`GET /api/agent-api/launches` — returns active launches with contract addresses, base token, and timing.

```python
def get_launches():
    path = "/api/agent-api/launches"
    resp = requests.get(BASE_URL + path, headers=sign_auth_headers("GET", path))
    resp.raise_for_status()
    return resp.json()
    # {"chainId": 8453, "contracts": {...}, "launches": [{
    #   "launchId", "name", "ticker", "contractAddress",
    #   "startTime", "endTime", "baseToken", "baseTokenDecimals",
    #   "maxContribution", "reserve", "started"
    # }]}
```

## Step 3 — Approve + participate on-chain

Approve the launch contract to spend your BIO (18 decimals) or USDC (6 decimals), then call `participate()`.

```python
ERC20_ABI = [{"inputs": [{"name": "spender", "type": "address"}, {"name": "value", "type": "uint256"}], "name": "approve", "outputs": [{"type": "bool"}], "stateMutability": "nonpayable", "type": "function"}]
LAUNCH_ABI = [{"inputs": [{"name": "bioAmt", "type": "uint256"}], "name": "participate", "outputs": [], "stateMutability": "nonpayable", "type": "function"}]

def approve_and_participate(launch_contract: str, base_token: str, amount: int):
    token = w3.eth.contract(address=base_token, abi=ERC20_ABI)
    tx = token.functions.approve(launch_contract, amount).build_transaction(
        {"from": ADDRESS, "nonce": w3.eth.get_transaction_count(ADDRESS), "chainId": CHAIN_ID}
    )
    w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )

    contract = w3.eth.contract(address=launch_contract, abi=LAUNCH_ABI)
    tx = contract.functions.participate(amount).build_transaction(
        {"from": ADDRESS, "nonce": w3.eth.get_transaction_count(ADDRESS), "chainId": CHAIN_ID}
    )
    w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )
```

## Step 4 — Claim tokens

After the launch ends and is finalized, fetch the Merkle proof and call `claimTokens()`.

`GET /api/agent-api/launches/{launchId}/claim`

```python
CLAIM_ABI = [{"inputs": [{"name": "_refundAmount", "type": "uint256"}, {"name": "_claimAmount", "type": "uint256"}, {"name": "_proof", "type": "bytes32[]"}], "name": "claimTokens", "outputs": [], "stateMutability": "nonpayable", "type": "function"}]

def claim_tokens(launch_id: int):
    path = f"/api/agent-api/launches/{launch_id}/claim"
    resp = requests.get(BASE_URL + path, headers=sign_auth_headers("GET", path))
    resp.raise_for_status()
    data = resp.json()

    contract = w3.eth.contract(address=data["contractAddress"], abi=CLAIM_ABI)
    proof = [bytes.fromhex(p[2:]) for p in data["merkleProof"]]
    tx = contract.functions.claimTokens(
        int(data["refundAmount"]), int(data["claimAmount"]), proof
    ).build_transaction(
        {"from": ADDRESS, "nonce": w3.eth.get_transaction_count(ADDRESS), "chainId": CHAIN_ID}
    )
    w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )
```

## Step 5 — Withdraw if launch failed

If the launch failed or was cancelled, get your full deposit back.

```python
WITHDRAW_ABI = [{"inputs": [], "name": "withdrawTokensIfLaunchFails", "outputs": [], "stateMutability": "nonpayable", "type": "function"}]

def withdraw_if_failed(launch_contract: str):
    contract = w3.eth.contract(address=launch_contract, abi=WITHDRAW_ABI)
    tx = contract.functions.withdrawTokensIfLaunchFails().build_transaction(
        {"from": ADDRESS, "nonce": w3.eth.get_transaction_count(ADDRESS), "chainId": CHAIN_ID}
    )
    w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )
```

## Contract addresses

| Network | Contract | Address |
|---|---|---|
| Base (8453) | BIO Token | `0x226A2FA2556C48245E57cd1cbA4C6c9e67077DD2` |
| Base (8453) | USDC Token | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |

## Edge cases

- **Already registered**: `POST /register` is idempotent — returns existing record.
- **Multiple participations**: `participate()` can be called multiple times. Cumulative total is capped by `maxContributionBioAmount`.
- **Unregistered wallet**: On-chain participation still works, but after finalization you only get a refund and no token allocation.
- **Token decimals**: BIO = 18 decimals, USDC = 6 decimals. Check `baseTokenDecimals` from the launches endpoint.

Full ABIs are in [references/ABIS.md](references/ABIS.md).
