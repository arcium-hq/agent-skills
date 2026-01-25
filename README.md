# Arcium Skills

Skills for building confidential applications on Solana with Arcium.

Skills follow the [Agent Skills](https://agentskills.io) format.

## Available Skills

### arcium

Build privacy-preserving applications on Solana—compute on encrypted data without revealing inputs. Works with Anchor programs using Arcium's confidential computing layer.

**Use when:**
- Computing on data that must stay private (voting, auctions, hidden game state)
- Multiple parties need to combine data without revealing individual inputs
- On-chain state must remain encrypted but still computable

**Covers:**
- Arcium patterns via [MCP documentation search](https://docs.arcium.com/mcp)
- Circuit development (`#[encrypted]`, `#[instruction]`)
- Anchor integration (`queue_computation`, callbacks)
- Client SDK (`@arcium-hq/client`)
- Common patterns: stateless, stateful, multi-party, randomness
- Troubleshooting for hard-to-debug errors

## Installation

### Using skills CLI

```bash
npx skills add arcium-hq/skills
```

### Using amp

[Amp](https://ampcode.com) users:

```bash
amp skill add arcium-hq/skills
```

### Manual installation

```bash
git clone https://github.com/arcium-hq/skills.git
cp -r skills/skills/arcium <your-skills-directory>/
```

See [skills.sh/docs](https://skills.sh/docs) for agent-specific installation paths.

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
