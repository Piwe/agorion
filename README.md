# Agent Services Discovery Protocol

[![Version](https://img.shields.io/badge/version-v1-0052ff)](./agent-services-spec-v1.md)
[![License: MIT](https://img.shields.io/badge/license-MIT-00ff88.svg)](./LICENSE)

The Agent Services Discovery Protocol (ASDP) defines a lightweight standard for AI agents to discover callable services at runtime, compare quality and pricing, and execute paid or free requests autonomously using a shared manifest format and discovery API conventions.

## Quick Start

### 1) Publish a manifest

Host a valid manifest at `/.well-known/agent-services.json`:

```json
{
  "schema": "agent-services/v1",
  "provider": "Example Corp",
  "contact": "https://example.com",
  "services": [
    {
      "id": "crypto-prices-v1",
      "name": "Crypto Prices",
      "description": "Spot prices for top pairs",
      "endpoint": "https://api.example.com/v1/prices",
      "method": "GET",
      "auth": { "type": "none" },
      "capabilities": ["crypto-prices", "market-data"],
      "rate_limit": "100/hour",
      "response_format": "application/json",
      "uptime_sla": "99.5%"
    }
  ]
}
```

Validate it against [`schema/agent-services-v1.schema.json`](./schema/agent-services-v1.schema.json).

### 2) Query a discovery API

```bash
curl "https://registry.example.com/discover?capability=market-data&max_price=0.01&chain=base"
```

Check service health:

```bash
curl "https://registry.example.com/health/correlation-matrix"
```

Register a manifest:

```bash
curl -X POST "https://registry.example.com/register" \
  -H "content-type: application/json" \
  -d '{"manifest_url":"https://example.com/.well-known/agent-services.json"}'
```

## Full Spec

- [Agent Services Discovery Protocol v1](./agent-services-spec-v1.md)

## Landing Page

- [Agent Discovery Network Landing Page](./index.html)

## Contributing

Contributions are welcome. Please open an issue or PR for schema improvements, interoperability feedback, or implementation examples.

## License

This project is licensed under the MIT License. See [`LICENSE`](./LICENSE).
