# Bio Launchpad ABIs

ABI fragments for on-chain interactions with the Bio Protocol launchpad on Base (chain ID 8453).

These are minimal ABIs containing only the functions used by this skill. For the complete contract ABIs, refer to the verified contracts on BaseScan.

## Table of contents

- [ERC-20 (approve, allowance, balanceOf)](#erc-20-approve-allowance-balanceof)
- [Launch - participate](#launch---participate)
- [Launch - mapAddrToBios](#launch---mapaddrtobios)
- [Launch - claimed](#launch---claimed)
- [Launch - claimTokens](#launch---claimtokens)
- [Launch - withdrawTokensIfLaunchFails](#launch---withdrawtokensiflaunchfails)
- [Python usage](#python-usage)

---

## ERC-20 (approve, allowance, balanceOf)

Standard ERC-20 functions for token balance checks, allowance checks, and approvals.

```json
[
  {
    "inputs": [{ "name": "spender", "type": "address" }, { "name": "value", "type": "uint256" }],
    "name": "approve",
    "outputs": [{ "type": "bool" }],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [{ "name": "owner", "type": "address" }, { "name": "spender", "type": "address" }],
    "name": "allowance",
    "outputs": [{ "type": "uint256" }],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [{ "name": "account", "type": "address" }],
    "name": "balanceOf",
    "outputs": [{ "type": "uint256" }],
    "stateMutability": "view",
    "type": "function"
  }
]
```

Use `balanceOf` to check the wallet has enough tokens before approving. Use `allowance` to check whether approval can be skipped.

## Launch - participate

Commit tokens to an active launch. The `bioAmt` parameter is in raw units matching the launch's `baseTokenDecimals` (18 for BIO, 6 for USDC). Despite the parameter name, this works for any supported base token - Solidity function selectors use types, not names.

```json
[
  {
    "inputs": [{ "name": "bioAmt", "type": "uint256" }],
    "name": "participate",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```

## Launch - mapAddrToBios

Read a user's cumulative contribution to the launch. Use this before participating to check whether the next contribution would exceed `maxContribution`.

```json
[
  {
    "inputs": [{ "name": "user", "type": "address" }],
    "name": "mapAddrToBios",
    "outputs": [{ "type": "uint256" }],
    "stateMutability": "view",
    "type": "function"
  }
]
```

Example check:
```python
current = contract.functions.mapAddrToBios(ADDRESS).call()
remaining = max_contribution - current
if amount > remaining:
    raise ValueError(f"Amount {amount} exceeds remaining capacity {remaining}")
```

## Launch - claimed

Check whether a wallet has already claimed its allocation. Call before attempting `claimTokens()` to avoid wasting gas on a revert.

```json
[
  {
    "inputs": [{ "name": "user", "type": "address" }],
    "name": "claimed",
    "outputs": [{ "type": "bool" }],
    "stateMutability": "view",
    "type": "function"
  }
]
```

## Launch - claimTokens

Claim your token allocation and any refund after a launch is finalized. Requires a Merkle proof from the Bio claim API (`GET /api/agent-api/launches/{launchId}/claim`).

```json
[
  {
    "inputs": [
      { "name": "_refundAmount", "type": "uint256" },
      { "name": "_claimAmount", "type": "uint256" },
      { "name": "_proof", "type": "bytes32[]" }
    ],
    "name": "claimTokens",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```

Convert the hex proof strings from the API:
```python
proof = [bytes.fromhex(p.removeprefix("0x")) for p in data["merkleProof"]]
```

## Launch - withdrawTokensIfLaunchFails

Withdraw your full deposit if the launch failed or was cancelled. No parameters needed.

```json
[
  {
    "inputs": [],
    "name": "withdrawTokensIfLaunchFails",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```

---

## Python usage

Load all ABIs for use with web3.py:

```python
ERC20_ABI = [
    {"inputs": [{"name": "spender", "type": "address"}, {"name": "value", "type": "uint256"}], "name": "approve", "outputs": [{"type": "bool"}], "stateMutability": "nonpayable", "type": "function"},
    {"inputs": [{"name": "owner", "type": "address"}, {"name": "spender", "type": "address"}], "name": "allowance", "outputs": [{"type": "uint256"}], "stateMutability": "view", "type": "function"},
    {"inputs": [{"name": "account", "type": "address"}], "name": "balanceOf", "outputs": [{"type": "uint256"}], "stateMutability": "view", "type": "function"},
]

LAUNCH_ABI = [
    {"inputs": [{"name": "bioAmt", "type": "uint256"}], "name": "participate", "outputs": [], "stateMutability": "nonpayable", "type": "function"},
    {"inputs": [{"name": "user", "type": "address"}], "name": "mapAddrToBios", "outputs": [{"type": "uint256"}], "stateMutability": "view", "type": "function"},
    {"inputs": [{"name": "user", "type": "address"}], "name": "claimed", "outputs": [{"type": "bool"}], "stateMutability": "view", "type": "function"},
    {"inputs": [{"name": "_refundAmount", "type": "uint256"}, {"name": "_claimAmount", "type": "uint256"}, {"name": "_proof", "type": "bytes32[]"}], "name": "claimTokens", "outputs": [], "stateMutability": "nonpayable", "type": "function"},
    {"inputs": [], "name": "withdrawTokensIfLaunchFails", "outputs": [], "stateMutability": "nonpayable", "type": "function"},
]
```