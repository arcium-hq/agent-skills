# Arcium Skills

Skills for building privacy-preserving Solana apps with Arcium MPC.

## Install

```bash
npx add-skill arcium-hq/skills
```

Or install specific skill:

```bash
npx add-skill arcium-hq/skills --skill arcium
```

## Available Skills

### arcium

Build Arcium MPC applications with Arcis circuits, Anchor programs, and TypeScript clients.

**Use when:**
- Writing Arcis circuits (`#[encrypted]`, `#[instruction]`)
- Integrating with Anchor programs (`queue_computation`, callbacks)
- Encrypting client inputs (`@arcium-hq/client`)

**Covers:**
- MPC mental model and constraints
- Circuit development patterns
- Solana program integration
- TypeScript client SDK
- Common patterns (stateless, stateful, multi-party, randomness)

## Resources

- [Arcium Docs](https://docs.arcium.com/developers/)
- [Examples Repo](https://github.com/arcium-hq/examples)

## License

MIT
