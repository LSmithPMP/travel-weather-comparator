# Product Requirements Document
## Travel Weather Comparator — A2A Multi-Agent System (HARDENED)

| Field | Detail |
|-------|--------|
| Project Name | Travel Weather Comparator — A2A Multi-Agent System |
| Document Type | Product Requirements Document (PRD) |
| Version | 2.0 — Hardened Release |
| Date | March 2026 |
| Program | Applied Agentic AI |
| Institution | Interview Kickstart |
| Week | Week 5 — A2A Multi-Agent Capstone |
| Architecture | A2A (Agent-to-Agent) · REST · HMAC-SHA256 |
| Primary Language | JavaScript (n8n) |
| Status | Production — Hardened |

---

## 1. Executive Summary

The Travel Weather Comparator is a production-hardened, 16-node Agent-to-Agent (A2A) multi-agent workflow built in n8n. It enables users to compare weather conditions and travel logistics across multiple US cities simultaneously, then delivers a ranked destination recommendation based on a configurable multi-factor scoring model.

This system implements a formal security architecture addressing 12 identified threat vectors — all resolved to Low residual severity through design controls. Security is embedded at every layer: authentication, input validation, rate limiting, A2A communication, and response handling.

---

## 2. Problem Statement

Travelers face a fragmented decision-making process when choosing a destination. Weather forecasts, flight costs, hotel availability, and event calendars exist in separate systems, requiring manual aggregation and trade-off analysis. This creates friction — users either accept suboptimal destinations due to analysis paralysis or fail to account for major cost disruptions such as festivals or peak-season pricing.

---

## 3. Objectives

- Compare weather conditions and travel logistics for up to 4 US cities simultaneously via parallel A2A agent calls
- Enforce a 3-layer security gate (API key + HMAC-SHA256 signature + CORS) on every inbound request
- Apply strict input validation — city whitelist, date regex, string sanitization, and fan-out cap
- Generate cryptographic request IDs and propagate them through all A2A calls for end-to-end traceability
- Score and rank destination cities using a multi-factor model with optional budget mode weighting
- Return sanitized, structured responses with all internal fields stripped before delivery to the caller

---

## 4. User Stories

| ID | User Story | Acceptance Criteria | Priority |
|----|-----------|-------------------|----------|
| US-01 | As a traveler, I want to compare weather for multiple cities so I can pick the best destination. | Weather data returned for up to 4 cities simultaneously; results displayed comparatively. | Must Have |
| US-02 | As a user, I want a ranked recommendation so I don't have to analyze all the data myself. | Coordinator outputs a ranked list with written rationale. | Must Have |
| US-03 | As a budget traveler, I want to filter results by cost so I can make a value decision. | Budget mode parameter adjusts scoring weights; recommendation shifts accordingly. | Should Have |
| US-04 | As a developer, I want all requests authenticated so the endpoint cannot be abused. | 3-layer auth gate enforced — all three must pass. | Must Have |
| US-05 | As a developer, I want rate limiting so the system cannot be flooded. | Sliding-window rate limiter — 10 req/60s — returns 429 on breach. | Must Have |
| US-06 | As a developer, I want all agent calls to propagate request IDs for traceability. | Cryptographic UUID generated at validation and propagated through all A2A calls. | Should Have |

---

## 5. System Architecture

### 5.1 Security Gate (3 Layers)
- **Layer 1 — API Key:** X-Api-Key header validated against TWC_API_KEY environment variable. No fallback or optional modes.
- **Layer 2 — HMAC-SHA256:** X-Hmac-Signature header verified using `crypto.timingSafeEqual()` to prevent timing side-channel attacks. Mandatory — no bypass path exists.
- **Layer 3 — CORS:** allowedOrigins enforced at the webhook node level before any application code executes.

### 5.2 A2A Protocol
Each A2A call uses a standardized task envelope:
```json
{
  "task_id": "TWC-<uuid>-<type>-<city>",
  "sender": "travel-coordinator",
  "task_type": "weather_forecast | travel_logistics",
  "payload": { ... }
}
```
Each call includes Bearer token authentication, X-A2A-Sender and X-A2A-Task headers, X-Request-Id propagation, and a 10-second timeout with certificate validation enforced.

### 5.3 Coordinator Logic
- **Part 1 — Scoring:** Each city scored across weather metrics (temperature, precipitation, comfort index) and travel metrics (flight cost, hotel rate, travel time, event warnings)
- **Part 2 — Ranking:** Cities ranked by composite score; budget mode applies configurable cost weighting multiplier

---

## 6. Workflow Nodes (16)

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | Travel Query Webhook | Webhook (POST) | Entry point — /travel-weather-compare |
| 2 | Authenticate Request | Code | 3-layer auth: API key + HMAC-SHA256 + CORS |
| 3 | Rate Limiter | Code | Sliding-window rate limiting + fan-out cap |
| 4 | Validate and Normalize Input | Code | Whitelist, regex, sanitization, crypto IDs |
| 5 | Split Cities | Code | Fan-out: one item per city (max 4) |
| 6 | Weather Agent A2A Call | HTTP Request | A2A task dispatch to Weather Agent |
| 7 | Travel Agent A2A Call | HTTP Request | A2A task dispatch to Travel Agent |
| 8 | Merge Weather Results | Merge | Collect parallel weather responses |
| 9 | Merge Travel Results | Merge | Collect parallel travel responses |
| 10 | Aggregate All City Data | Code | Combine all agent results per city |
| 11 | Coordinator Part 1 Scoring | Code | Score each city on weather + travel metrics |
| 12 | Coordinator Part 2 Ranking | Code | Rank cities, apply budget mode logic |
| 13 | Format Final Response | Code | Structure output, strip internal fields |
| 14 | Return Response to Caller | Respond to Webhook | Send secured response |
| 15 | Error Handler | Code | Centralized error normalization |
| 16 | Return Error Response | Respond to Webhook | Sanitized error response |

---

## 7. Security Architecture — Threat Model

| ID | Threat | Remediation | Residual Risk |
|----|--------|------------|--------------|
| T-01 | Unauthenticated access | 3-layer auth gate (API key + HMAC-SHA256 + CORS) | **Low** |
| T-03 | Input injection | Whitelist validation, regex on dates, string sanitization | **Low** |
| T-04 | Denial of service / flooding | Sliding-window rate limiter — 10 req/60s | **Low** |
| T-06 | Request replay / spoofing | Cryptographic UUID request IDs via crypto.randomUUID() | **Low** |
| T-07 | Fan-out amplification | Hard cap of 4 cities per request enforced pre-fanout | **Low** |
| T-09 | Cross-origin abuse | allowedOrigins enforced at webhook level | **Low** |
| T-12 | Timezone / timestamp leakage | All timestamps in UTC — no local timezone exposure | **Low** |
| T-SC | Timing side-channel attacks | crypto.timingSafeEqual() for HMAC comparison | **Low** |

### Security Response Headers
All responses include:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
- `Cache-Control: no-store`
- `X-Request-Id: <cryptographic-uuid>`

---

## 8. Environment Variables

| Variable | Description |
|----------|-------------|
| TWC_API_KEY | Required: API key for authenticating inbound requests |
| TWC_HMAC_SECRET | Required: Secret for HMAC-SHA256 body signature verification |
| A2A_AGENT_TOKEN | Required: Bearer token for outbound A2A agent calls |
| WEATHER_AGENT_URL | Required: URL of the Weather Agent service |
| TRAVEL_AGENT_URL | Required: URL of the Travel Agent service |

---

## 9. Supported Cities & Origins

**Destination Cities (whitelist-validated):**
Austin · Miami · Denver · Chicago · New Orleans · San Francisco · Nashville

**Origin Cities:**
NYC · LA · Chicago · Austin · Miami · Denver

---

## 10. Constraints & Non-Functional Requirements

- Maximum 4 cities per request — enforced at both Rate Limiter and Input Validator nodes
- Maximum 10 requests per 60-second sliding window per client
- All timestamps in UTC — no local timezone exposure
- All HMAC comparisons use timing-safe comparison to prevent side-channel attacks
- Error responses are sanitized — no stack traces, internal paths, or system details exposed
- Allowed origins must be explicitly configured — no wildcard CORS

---

## 11. Academic Context

| Field | Detail |
|-------|--------|
| Program | Applied Agentic AI |
| Institution | Interview Kickstart |
| Week | Week 5 — A2A Multi-Agent Capstone |
| Semester | Spring 2026 |
| Author | Lamonte Smith |
| GitHub | github.com/LSmithPMP/travel-weather-comparator |

---

*Lamonte Smith · Interview Kickstart — Applied Agentic AI · March 2026 · Confidential*
