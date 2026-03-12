# Examples & Dependencies

## Minimal curl example (manual signing)

```bash
# 1. Get payment requirements (x402 v2 in response body)
curl -s -X POST https://x402.ai.bio.xyz/api/deep-research/start \
  -H "Content-Type: application/json" \
  -d '{"message":"What is NAD+?","researchMode":"steering"}'
# → 402 with x402Version:2 payment requirements in JSON body

# 2. Sign the authorization (language-specific, see SKILL.md Step 2)

# 3. Submit with PAYMENT-SIGNATURE header
curl -s -X POST https://x402.ai.bio.xyz/api/deep-research/start \
  -H "Content-Type: application/json" \
  -H "PAYMENT-SIGNATURE: <base64-encoded x402 v2 payment payload>" \
  -d '{"message":"What is NAD+?","researchMode":"steering"}'
# → 200 with conversationId

# 4. Poll
curl -s https://x402.ai.bio.xyz/api/deep-research/{conversationId}

# 5. (Optional) Fetch feedback data for on-chain submission
curl -s https://x402.ai.bio.xyz/api/feedback/{conversationId}
```

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
