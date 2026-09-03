# ADR-0001: Alert Delivery Design

## Status

Proposed

## Context

OpenFloodAI detects flood risk states (NORMAL, WATCH, WARNING_CANDIDATE)
from camera streams. When conditions escalate, operators need timely
notification so they can act -- especially in remote deployments along
Nepal's river basins where monitoring dashboards may not be watched
continuously.

The system needs a way to push notifications outward when risk state
transitions occur, and must handle the intermittent connectivity common
in edge deployments.

## Decision

We propose a webhook-based alert delivery system with offline buffering.

### Webhook delivery

- HTTP POST to operator-configured URLs on risk-state escalation
- JSON payload with site_id, camera_id, risk_state, previous_risk_state,
  reason, and timestamp
- Configurable per-endpoint secret for authentication via
  `X-OpenFloodAI-Secret` header (HTTPS only -- never sent over plain HTTP)
- Supports integration with Slack, Telegram bots, SMS gateways, or
  custom dashboards

### Offline buffer

- Failed deliveries are saved as JSON files on disk for retry
- Configurable max capacity (default 500) with oldest-eviction policy
- Exponential retry with configurable max attempts (default 10)
- Batch flush to avoid overwhelming endpoints on reconnection
- Webhook secrets are NOT stored in buffered files -- resolved from
  in-memory config at flush time

### Security considerations

- **SSRF prevention**: Webhook URLs are validated at configuration time
  (scheme must be http or https) and at delivery time (hostname resolved
  and checked against private/loopback/reserved IP ranges)
- **Secret protection**: `X-OpenFloodAI-Secret` header only sent over
  HTTPS connections; secrets never written to disk in buffer files
- **Path traversal**: Site IDs sanitized before use in buffer filenames
- **Buffer SSRF**: Flushed alerts only delivered to URLs present in the
  active webhook configuration, preventing tampered buffer files from
  reaching arbitrary endpoints
- **Response limits**: Webhook response bodies capped at 500 characters
  in delivery records

### Alert rules

Only escalation transitions trigger alerts:
- NORMAL -> WATCH
- NORMAL -> WARNING_CANDIDATE
- WATCH -> WARNING_CANDIDATE

De-escalations (WARNING_CANDIDATE -> NORMAL) and same-state transitions
do not fire alerts to avoid notification fatigue.

## Proposed modules

```
src/openfloodai/alerts/
    __init__.py
    webhook.py      # WebhookConfig, send_alert(), should_alert()
    buffer.py       # BufferConfig, buffer_alert(), flush_buffer()
```

## Consequences

- Operators can receive flood warnings without watching dashboards
- Intermittent connectivity does not cause lost warnings
- No dependency on external message brokers or cloud services
- Alert delivery adds a small delay per webhook on each escalation
- Buffer disk usage is bounded by max_buffered_alerts setting
- Webhook secrets require HTTPS endpoints for authenticated delivery

## Open questions

- Should we add rate limiting per webhook endpoint?
- Should alert payloads include multi-source risk data when available?
- Should we support other delivery mechanisms (MQTT, SMS direct)?
