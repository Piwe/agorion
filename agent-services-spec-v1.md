# Agent Services Discovery Protocol v1

## Status of This Memo

This document specifies version 1 of the Agent Services Discovery Protocol (ASDP). Distribution of this memo is unlimited.

This specification uses the key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

## Abstract

The Agent Services Discovery Protocol (ASDP) defines a lightweight, web-native mechanism that allows AI agents to discover, evaluate, and pay for services at runtime. The protocol standardizes a provider manifest at `.well-known/agent-services.json` and a registry-facing discovery API for indexing, filtering, health checks, and reputation lookup.

Today, most agent-to-service integrations are static and hardcoded. ASDP introduces interoperable service metadata, budget-aware discovery, and payment-compatible service declarations (including x402 terms), enabling autonomous service selection without manual intervention.

## Problem

AI agents can call tools, APIs, and agent-operated endpoints, but they usually do so through hand-wired integrations. This creates four structural constraints:

1. Integration lock-in: each agent runtime must pre-bundle provider-specific adapters.
2. Slow market participation: new providers are unavailable until explicitly integrated.
3. No runtime optimization: agents cannot compare alternatives by capability, price, or reliability.
4. Fragile monetization: paid endpoints have no common declaration model for autonomous payment decisions.

As the agent economy scales, these constraints prevent fluid service markets. Agents need a way to discover providers dynamically, evaluate quality and cost, and transact under policy limits in real time.

## Motivation

ASDP is designed to provide the functional equivalent of a service-level directory for autonomous software clients. The protocol does not replace API design, authentication standards, or payment rails. Instead, it standardizes discoverability and selection metadata so existing APIs can become agent-callable with minimal changes.

The goals are:

- Minimal provider lift: publish a single JSON manifest under a standard `.well-known` path.
- Deterministic parsing: fixed schema versioning and strict required fields.
- Composable discovery: registry APIs that support filtering by capability, budget, and payment network.
- Economic interoperability: explicit support for x402-style paid calls and budget-aware agent policies.
- Quality signaling: objective health and reliability metrics exposed via reputation endpoints.

Non-goals include replacing OpenAPI, enforcing one payment protocol, or requiring centralized execution. ASDP can be used in centralized registries, federated registries, or local indexes.

## Terminology

- Agent: software process that decides and performs API calls autonomously.
- Provider: organization or operator that publishes one or more callable services.
- Service: a single callable endpoint with semantics, constraints, auth mode, and optional price terms.
- Manifest: the JSON document published at `.well-known/agent-services.json` that describes provider services.
- Registry: a system that crawls or receives manifests and exposes query APIs for discovery and quality signals.
- Discovery agent: an agent component that queries registries and selects services.
- Capability: normalized tag indicating service function, e.g., `crypto-prices`, `news-summarization`.
- x402: payment authorization model where API access terms can be machine-interpreted and machine-settled.
- Reputation score: normalized 0-100 score representing service/provider quality over a rolling interval.

## Protocol Overview

ASDP has two required protocol surfaces and two optional but recommended surfaces.

### 1. Service Manifest (`.well-known/agent-services.json`)

Any API provider can publish this file to declare agent-callable services:

```json
{
  "schema": "agent-services/v1",
  "provider": "Example Corp",
  "contact": "https://example.com",
  "services": [
    {
      "id": "unique-service-id",
      "name": "Human readable name",
      "description": "What this service does",
      "endpoint": "https://api.example.com/v1/data",
      "method": "GET",
      "auth": {
        "type": "x402|apikey|bearer|none",
        "price": "$0.01",
        "currency": "USDC",
        "network": "base"
      },
      "capabilities": ["crypto-prices", "market-data"],
      "rate_limit": "100/hour",
      "response_format": "application/json",
      "uptime_sla": "99.5%"
    }
  ]
}
```

Provider requirements:

- The manifest MUST be served over HTTPS.
- The path MUST be `/.well-known/agent-services.json`.
- The top-level `schema` field MUST equal `agent-services/v1` for this version.
- At least one service object MUST be present.
- Each service `id` MUST be unique within the manifest.
- Each service `endpoint` SHOULD be absolute and HTTPS.

### 2. Discovery API

A central or federated registry indexes manifests and exposes query endpoints.

Required endpoints:

- `GET /discover?capability=X&max_price=Y&chain=Z`
- `GET /health/{service_id}`
- `GET /reputation/{provider_id}`
- `POST /register` (submit manifest URL)

Registry behavior requirements:

- Registry MUST validate manifests against the v1 schema before indexing.
- Registry SHOULD refresh indexed manifests periodically (recommended interval: <= 1 hour).
- Registry MUST mark manifests with parsing failures as invalid and return structured errors.
- `discover` responses MUST include enough data for selection (service id, endpoint, auth type, price metadata, and reputation summary).

### 3. Payment Integration

Services can declare x402 payment terms. Discovery agents can filter by budget and auto-negotiate based on user policy constraints.

### 4. Reputation

Registry tracks uptime, response time, error rate, and payment disputes. Reputation score is 0-100 and updated hourly.

## Manifest Schema (JSON Schema)

### Design Principles

The manifest schema is intentionally compact. It avoids embedding full API contracts and focuses on discovery-critical metadata. ASDP complements, rather than replaces, OpenAPI and provider docs.

Top-level object fields:

- `schema` (string, REQUIRED): protocol version identifier.
- `provider` (string, REQUIRED): provider display name.
- `contact` (string URI, REQUIRED): canonical contact/support URL.
- `services` (array, REQUIRED): at least one service declaration.

Service object required fields:

- `id`, `name`, `description`
- `endpoint`, `method`
- `auth`
- `capabilities`
- `rate_limit`
- `response_format`
- `uptime_sla`

Auth object requirements:

- `type` REQUIRED, enum: `x402`, `apikey`, `bearer`, `none`
- If `type = x402`, then `price`, `currency`, and `network` are REQUIRED.

The canonical JSON Schema for v1 is provided in this repository at `schema/agent-services-v1.schema.json`. The following excerpt illustrates key constraints:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema", "provider", "contact", "services"],
  "properties": {
    "schema": { "const": "agent-services/v1" },
    "provider": { "type": "string", "minLength": 1 },
    "contact": { "type": "string", "format": "uri" },
    "services": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/service" }
    }
  }
}
```

### Manifest Validation and Failure Handling

- Providers SHOULD validate manifests before publication.
- Registries MUST reject manifests that fail schema validation.
- Registries SHOULD return line-item validation errors including JSON Pointer paths.
- Agents MUST treat unknown schema versions as unsupported unless explicitly configured for fallback behavior.

### Caching and Freshness

- Providers SHOULD set `Cache-Control` and `ETag` headers for efficient re-crawling.
- Registries SHOULD use conditional requests (`If-None-Match`) when refreshing manifests.
- Registries SHOULD expose `last_indexed_at` in discovery responses.

## Discovery API (OpenAPI-style)

The following interface is a normative profile for ASDP discovery registries. Providers MAY implement additional endpoints if they preserve these semantics.

```yaml
openapi: 3.1.0
info:
  title: Agent Services Discovery API
  version: 1.0.0
paths:
  /discover:
    get:
      summary: Query services by capability, price, and chain
      parameters:
        - name: capability
          in: query
          required: true
          schema: { type: string }
        - name: max_price
          in: query
          required: false
          schema: { type: string, description: "Decimal in USD or token-denominated value" }
        - name: chain
          in: query
          required: false
          schema: { type: string, example: "base" }
        - name: auth_type
          in: query
          required: false
          schema: { type: string, enum: [x402, apikey, bearer, none] }
        - name: min_reputation
          in: query
          required: false
          schema: { type: integer, minimum: 0, maximum: 100 }
      responses:
        '200':
          description: Ranked list of matching services
  /health/{service_id}:
    get:
      summary: Service health telemetry
      parameters:
        - name: service_id
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Current health and historical window stats
  /reputation/{provider_id}:
    get:
      summary: Provider-level reputation data
      parameters:
        - name: provider_id
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: Reputation score and components
  /register:
    post:
      summary: Submit manifest URL for indexing
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [manifest_url]
              properties:
                manifest_url:
                  type: string
                  format: uri
      responses:
        '202':
          description: Accepted for asynchronous validation/indexing
```

### Discovery Response Model

`GET /discover` SHOULD return:

- `service_id`, `provider_id`, `provider_name`
- `endpoint`, `method`, `capabilities`
- `auth.type`, optional `auth.price`, `auth.currency`, `auth.network`
- `reputation.score`, `reputation.updated_at`
- `health.uptime_24h`, `health.p95_latency_ms`, `health.error_rate_24h`
- `last_indexed_at`

Registries MAY include a ranking explanation object. If present, each scoring component SHOULD expose a bounded normalized value.

### Ranking Guidance

ASDP does not enforce one ranking function, but registries SHOULD provide transparent and stable ordering. A practical default:

`score = w1*reputation + w2*latency_score + w3*price_score + w4*freshness_score`

Where each term is normalized to `[0,1]`, and weights are configuration-specific.

### Error Semantics

- `400` for invalid query params.
- `404` for unknown service/provider IDs.
- `409` when registering a manifest already pending validation.
- `422` for schema-invalid manifests.
- `429` for rate limits.
- `5xx` for transient server failures.

Errors SHOULD be returned as:

```json
{
  "error": {
    "code": "invalid_manifest",
    "message": "schema validation failed",
    "details": [{"path": "/services/0/endpoint", "reason": "must be uri"}]
  }
}
```

## Payment Flow

ASDP v1 supports payment declaration and selection. Settlement may be handled by x402-compatible infrastructure or provider-specific processors.

### Objectives

- Let agents identify paid services before execution.
- Allow budget-constrained selection and policy enforcement.
- Preserve auditability of payment decisions and disputes.

### Reference Flow (x402)

1. Discovery: agent queries `GET /discover` with `capability`, `max_price`, and optional `chain`.
2. Selection: agent ranks candidates based on price, quality, and policy.
3. Payment preparation: if `auth.type = x402`, agent obtains payment authorization artifact according to provider/network requirements.
4. Invocation: agent calls service endpoint with payment proof/headers.
5. Verification: provider validates payment artifact.
6. Fulfillment: provider returns response payload.
7. Reconciliation: agent records outcome (success/failure, amount spent, latency).

### Budget and Policy Controls

Agent runtimes SHOULD support:

- Per-call maximum spend.
- Per-session and per-day spend ceilings.
- Allowed chains/tokens whitelist.
- Automatic fallback to `auth.type = none` services when paid calls exceed budget.

### Price Semantics

- `auth.price` MUST be machine-parseable as a decimal string with optional currency symbol.
- `auth.currency` SHOULD identify token or fiat-denominated unit.
- `auth.network` SHOULD indicate settlement network (e.g., `base`).

### Disputes and Refunds

ASDP does not define settlement arbitration. Registries MAY index dispute rates and include them in reputation components.

## Reputation Model

Reputation is intended to reflect service reliability and transaction integrity, not marketing popularity.

### Inputs

Registry implementations SHOULD track at least:

- Uptime over rolling windows (24h, 7d, 30d)
- Latency distribution (p50/p95)
- Error rate (transport + application failures)
- Payment dispute rate (for paid services)

Optional inputs include schema conformance consistency, freshness of manifest updates, and security incidents.

### Score Computation

A normative baseline model:

- `U` = uptime score [0..100]
- `L` = latency score [0..100]
- `E` = error score [0..100]
- `D` = dispute score [0..100]

`reputation = 0.40*U + 0.25*L + 0.25*E + 0.10*D`

Registries MAY tune weights but SHOULD publish them.

### Update Cadence and Decay

- Reputation MUST be recomputed at least hourly.
- Recent windows SHOULD carry greater weight than older windows.
- Providers with insufficient data SHOULD be marked `cold_start` and assigned conservative defaults.

### Transparency

`GET /reputation/{provider_id}` SHOULD return:

- current score
- component breakdown
- sample size or confidence indicator
- `updated_at` timestamp

## Security Considerations

ASDP introduces a discovery layer and therefore a new trust surface. Implementers should consider the following threats and mitigations.

### Manifest Authenticity and Integrity

Threat: malicious actor publishes spoofed manifest impersonating a known provider.

Mitigations:

- Registry MUST require HTTPS manifest URLs.
- Registry SHOULD verify domain ownership consistency between submitted URL and manifest endpoint host.
- Registry SHOULD support optional signature verification (e.g., detached JWS) in future-compatible extensions.

### Endpoint Substitution and SSRF Risks

Threat: manifest endpoints target internal metadata services or private infrastructure.

Mitigations:

- Registry crawlers MUST block private RFC1918, link-local, and loopback targets unless explicitly allowlisted.
- Crawlers MUST enforce outbound request policy and timeout ceilings.
- Agents SHOULD restrict callable domains via policy.

### Price and Payment Manipulation

Threat: provider changes `auth.price` unexpectedly or returns inconsistent payment requirements.

Mitigations:

- Registries SHOULD track and expose price history.
- Agents SHOULD enforce local max-spend checks immediately before invocation.
- Payment payloads SHOULD include nonce/timestamp to prevent replay.

### Abuse and Denial of Service

Threat: attackers spam `/register` or flood `/discover`.

Mitigations:

- Registry MUST apply authentication and rate limits for write endpoints.
- Registry SHOULD implement abuse detection and request throttling for read endpoints.
- Registries MAY require proof-of-work, stake, or allowlisted onboarding for high-volume publishers.

### Data Leakage and Privacy

Threat: discovery queries reveal agent strategy or proprietary intents.

Mitigations:

- Discovery clients SHOULD minimize query specificity where possible.
- Registries SHOULD provide retention controls and publish privacy policy.
- Agents SHOULD avoid sending user-identifying data in query parameters.

### Supply Chain and Schema Drift

Threat: incompatible schema changes cause silent parse failures.

Mitigations:

- Version field MUST be explicit (`agent-services/v1`).
- Breaking changes MUST use a new schema version.
- Registries SHOULD maintain backward compatibility windows and deprecation notices.

## Future Work

ASDP v1 intentionally defines a minimal interoperability core. Future versions may standardize additional features:

1. Signed manifests and key distribution.
2. Capability taxonomies and semantic ontology mapping.
3. Federated discovery synchronization across multiple registries.
4. Native policy bundles (budget, jurisdiction, compliance tags).
5. SLA verification proofs and independent telemetry attestation.
6. Negotiable pricing models (volume tiers, dynamic quotes, auction routing).
7. Payment abstraction layer beyond x402 (fiat rails, credit channels, hybrid custody).
8. Stronger provenance metadata (operator identity, software bill of materials, audit evidence).

## Conformance

An implementation is ASDP v1 conformant if:

- A provider publishes a manifest that validates against `agent-services-v1.schema.json` and is served at the required `.well-known` path.
- A registry supports and correctly implements the required discovery endpoints and semantics in this document.
- Agents parse manifest and discovery responses without violating required field or payment constraints.

Interoperability testing should include malformed manifest handling, pricing filter correctness, rate limit behavior, and reputation update cadence checks.

## Conclusion

ASDP v1 defines a practical discovery substrate for autonomous agents: publish standardized service manifests, index them through queryable registries, evaluate quality and price, and execute paid or free calls within policy limits. The model is deliberately incremental, so providers can participate with minimal overhead while enabling a dynamic, competitive service market for AI-native workloads.
