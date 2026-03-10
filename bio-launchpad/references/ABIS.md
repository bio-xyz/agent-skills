# Bio Launchpad ABIs

## ERC-20 (approve, allowance, balanceOf)

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

## Launch (participate, claim, withdraw, read state)

```json
[
  {
    "inputs": [{ "name": "bioAmt", "type": "uint256" }],
    "name": "participate",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [{ "name": "user", "type": "address" }],
    "name": "mapAddrToBios",
    "outputs": [{ "type": "uint256" }],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [{ "name": "user", "type": "address" }],
    "name": "claimed",
    "outputs": [{ "type": "bool" }],
    "stateMutability": "view",
    "type": "function"
  },
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
  },
  {
    "inputs": [],
    "name": "withdrawTokensIfLaunchFails",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```
