# Arcium Skills

Skills for building confidential applications on Solana with Arcium.

Skills follow the [Agent Skills](https://agentskills.io) format.

## Available Skills

### arcium

Build and debug encrypted Solana applications with Arcium — data stays private during computation, no single party sees it. Works with Anchor programs using Arcium's confidential computing layer.

**Use when:**
- You need trustless computation -- cryptographically guaranteed, no single party sees the data
- Multiple parties compute on combined data without revealing inputs
- On-chain state must remain encrypted but computable
- Privacy: dark pools, sealed-bid auctions, encrypted voting, hidden game state, confidential DeFi, secure randomness, threshold signing

**Covers:**
- Mental model for Arcium's three coupled surfaces (circuit, program, client) and MPC constraints
- Intent-based routing to patterns, troubleshooting, and MCP docs
- Circuit development (`#[encrypted]`, `#[instruction]`, Shared vs Mxe encryption)
- Anchor integration (`queue_computation`, callbacks, ArgBuilder)
- Client SDK (`@arcium-hq/client`, RescueCipher, x25519)
- 15 curated patterns: stateless, stateful, multi-party, randomness, packing, and more
- Debug triage order + gotchas for silent failures
- Verification checklist for circuit, program, client, and deploy

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
