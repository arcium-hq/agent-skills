# Arcium Skills

Skills for building confidential applications on Solana with Arcium.

Skills follow the [Agent Skills](https://agentskills.io) format.

## Available Skills

### arcium

Build privacy-preserving Solana apps with Arcium MPC -- trustless encrypted computation where no single party sees the data. Works with Anchor programs using Arcium's confidential computing layer.

**Use when:**
- You need trustless computation -- cryptographically guaranteed privacy
- Multiple parties compute on combined data without revealing inputs
- On-chain state must remain encrypted but computable
- Privacy: sealed-bid auctions, voting, hidden game state, dark pools, confidential DeFi

**Covers:**
- Intent-based routing to patterns, troubleshooting, and MCP docs
- Circuit development (`#[encrypted]`, `#[instruction]`)
- Anchor integration (`queue_computation`, callbacks, ArgBuilder)
- Client SDK (`@arcium-hq/client`, RescueCipher, x25519)
- 15 curated patterns: stateless, stateful, multi-party, randomness, packing, and more
- Debugging: ArgBuilder ordering, nonce errors, callback failures

## Installation

```bash
npx skills add arcium-hq/agent-skills
```

Auto-installs to whichever agent is detected (Claude Code, Amp, Cursor, Codex, Windsurf, and 40+ others). See [skills.sh/docs](https://skills.sh/docs) for the full list.

### Manual

```bash
git clone https://github.com/arcium-hq/agent-skills.git
cp -r agent-skills/skills/arcium <your-skills-directory>/
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Example prompts:**

- "How do I encrypt voting data with Arcium?"
- "Show me the queue_computation callback pattern"
- "Search Arcium docs for RescueCipher encryption"

## Skill Structure

Each skill contains:

- `SKILL.md` - Instructions for the agent
- `mcp.json` - MCP server configuration (optional)
- `examples/` - Curated code patterns
- `references/` - Troubleshooting and deep dives

## Resources

- [Arcium Docs](https://docs.arcium.com/developers/)
- [Examples Repo](https://github.com/arcium-hq/examples)

## License

MIT
