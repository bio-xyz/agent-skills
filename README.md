# agent-skills

Agent skills for [bio.xyz](https://bio.xyz) - installable via [`npx skills`](https://github.com/vercel-labs/skills).

## Available Skills

| Skill | Description |
|-------|-------------|
| **bios-deep-research** | Run paid deep research queries on BIOS via the x402 payment protocol. Uses USDC on Base for micro-payments. |

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
