# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| HARDENED (current) | ✅ |
| Pre-hardening builds | ❌ |

## Security Design

This workflow was built with a formal threat model. All 12 identified threat vectors have been remediated. See README.md for the full threat-remediation matrix.

### Authentication
- **Three-layer mandatory auth** — all three layers must pass; no optional or fallback modes
- Layer 1: API Key via `X-Api-Key` header validated against `TWC_API_KEY` environment variable
- Layer 2: HMAC-SHA256 body signature via `X-Hmac-Signature` header
- Layer 3: CORS origin restriction enforced at the webhook node level

### Secrets Management
- All secrets are stored as n8n environment variables — never hardcoded in node logic
- Required variables: `TWC_API_KEY`, `TWC_HMAC_SECRET`, `A2A_AGENT_TOKEN`
- Timing-safe comparison (`crypto.timingSafeEqual`) used for HMAC verification to prevent side-channel attacks

### Input Validation
- All fields validated against whitelists — no raw passthrough of user input
- String length caps enforced on all fields (max 64 characters)
- `travel_date` validated with strict `YYYY-MM-DD` regex and calendar date check
- Special characters stripped via sanitization function

### Rate Limiting
- Sliding-window rate limiter: 10 requests per 60-second window
- Fan-out amplification capped at 4 cities per request

### Response Security
- All responses include hardened security headers
- Internal fields stripped before response is returned to caller
- Error responses sanitized — no stack traces or internal paths exposed

## Reporting a Vulnerability

If you discover a security vulnerability in this workflow, please report it responsibly:

1. **Do not** open a public GitHub issue
2. Email: lamontesmithpmp@gmail.com with subject line `[SECURITY] travel-weather-comparator`
3. Include: description, reproduction steps, and potential impact
4. Expected response time: 72 hours

## Disclaimer

This workflow was built for academic and demonstration purposes. Before deploying to production, conduct a full security review appropriate to your threat model and compliance requirements.
