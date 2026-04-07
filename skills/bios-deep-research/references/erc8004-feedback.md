# ERC-8004 Feedback — Contract Details

Referenced from Step 5 of the main skill. Use after receiving a `completed` research result.

## Contract addresses

| Network     | ReputationRegistry                           |
| ----------- | -------------------------------------------- |
| Base (8453) | `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` |

**BIOS agentId:** `19177`

## ABI (giveFeedback only)

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

## Parameters

| Parameter       | Value                                               |
| --------------- | --------------------------------------------------- |
| `agentId`       | `19177`                                             |
| `value`         | `1` (success)                                       |
| `valueDecimals` | `0`                                                 |
| `tag1`          | `"x402"`                                            |
| `tag2`          | The research mode used (e.g. `"steering"`)          |
| `endpoint`      | `"https://x402.ai.bio.xyz/api/deep-research/start"` |
| `feedbackURI`   | `"{baseDir}/api/feedback/{conversationId}"`         |
| `feedbackHash`  | The `feedbackHash` from step 5a                     |

## Python example (web3.py)

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
    research_mode,                  # tag2: "steering", "semi-autonomous", etc.
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

## TypeScript example (viem)

```typescript
import { createWalletClient, http, parseAbi } from "viem";
import { base } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const REPUTATION_REGISTRY = "0x8004BAa17C55a88189AE136b182e5fdA19dE9b63";
const BIOS_AGENT_ID = 19177n;

const account = privateKeyToAccount(privateKey);
const walletClient = createWalletClient({
  account,
  chain: base,
  transport: http(),
});

const txHash = await walletClient.writeContract({
  address: REPUTATION_REGISTRY,
  abi: parseAbi([
    "function giveFeedback(uint256 agentId, int128 value, uint8 valueDecimals, string tag1, string tag2, string endpoint, string feedbackURI, bytes32 feedbackHash)",
  ]),
  functionName: "giveFeedback",
  args: [
    BIOS_AGENT_ID,
    1n, // value: success
    0, // valueDecimals
    "x402", // tag1
    researchMode, // tag2
    "https://x402.ai.bio.xyz/api/deep-research/start",
    `${apiBase}/api/feedback/${conversationId}`,
    feedbackHash as `0x${string}`, // feedbackHash from step 5a
  ],
});

console.log(`Feedback tx: ${txHash}`);
```
