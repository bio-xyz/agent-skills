# agent-skills

Agent skills for [bio.xyz](https://bio.xyz) - installable via [`npx skills`](https://github.com/vercel-labs/skills).

## Available Skills

| Skill | Description |
|-------|-------------|
| **bios-deep-research** | Run deep research queries on BIOS. Supports API key auth and x402 crypto payments with USDC on Base. |
| **bios-data-analysis** | Upload files and run paid data analysis on BIOS via x402 crypto payments (USDC on Base). Handles file upload, payment negotiation, SIWX-authenticated polling, timeout retry, and artifact download. |
| **bios-literature** | Search biomedical literature on BIOS via x402 crypto payments (USDC on Base). Two modes — fast (synchronous) and deep (async with SIWX-authenticated polling). Covers payment negotiation, timeout retry, and ERC-8004 feedback. |
| **bio-launchpad** | Participate in Bio Protocol launches on Base. Handles launch discovery, on-chain participation, claiming allocations, and withdrawals. |

## Install

```bash
npx skills add bio-xyz/agent-skills
```

Install a specific skill:

```bash
npx skills add bio-xyz/agent-skills --skill bios-deep-research
```

Install to a specific agent:

```bash
npx skills add bio-xyz/agent-skills -a cursor
npx skills add bio-xyz/agent-skills -a claude-code
```

List available skills without installing:

```bash
npx skills add bio-xyz/agent-skills --list
```

## License

MIT
