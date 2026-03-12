# Agorion

**The Discovery Network for AI Agents**

*From the Greek ἀγορά (agora) — the marketplace where everyone gathered.*

Agorion is an open protocol and network for AI agent service discovery. Agents publish what they do. Other agents find them. Services get discovered, priced, and paid — no human in the loop.

## How It Works

1. **Publish** — Providers host `/.well-known/agent-services.json` declaring endpoints, pricing, and capabilities
2. **Discover** — Agents query the Agorion registry by capability, chain, price, and reputation
3. **Pay** — Agents auto-negotiate and settle via [x402](https://github.com/coinbase/x402) when needed

## Quick Start

Add a manifest to your API:

```json
// /.well-known/agent-services.json
{
  "schema": "agent-services/v1",
  "provider": "Your Company",
  "services": [
    {
      "id": "my-service",
      "name": "My Agent Service",
      "endpoint": "https://api.example.com/v1/data",
      "method": "GET",
      "auth": { "type": "x402", "price": "$0.01", "currency": "USDC", "network": "base" },
      "capabilities": ["market-data"],
      "rate_limit": "120/hour",
      "response_format": "application/json"
    }
  ]
}
```

## Resources

- [Full Spec (v1)](./agent-services-spec-v1.md)
- [JSON Schema](./schema/agent-services-v1.schema.json)
- [Example: BotIndex Manifest](./examples/botindex-manifest.json)
- [Website](https://agorion.dev)

## License

MIT
