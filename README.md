# runtime-bifrost

Sovereign Bifrost v2.3+ L7 AI & MCP Gateway for Zerops (`PostgreSQL 18` config/logs store + `chromem`/`qdrant` vector store — never Redis for semantic cache).

## Zerops Configuration

Deployed via root [`zerops.yaml`](zerops.yaml) and persisted at `/mnt/localstorage/bifrost/config.json`.

### Required Environment Variables (Project Shared Vault / `zerops.yaml`)
- `PG_HOST`, `PG_PORT`, `PG_USER`, `PG_PASSWORD`, `PG_DATABASE` — PostgreSQL 18 connection.
- `FREELLMAPI_UNIFIED_KEY` (`${FREELLMAPI_CLIENT_KEY}`) — Auth key for internal FreeLLMAPI pool (`http://freellmapi:3001`).
- `BIFROST_VIRTUAL_KEY` (`${BIFROST_MASTER_VK}`) — Single master Virtual Key (`vk-production-main`). Do **not** hardcode plaintext VKs in `config.json`.
- `BIFROST_ADMIN_USERNAME` (`${BIFROST_ADMIN_USER_SECRET}`) & `BIFROST_ADMIN_PASSWORD` (`${BIFROST_ADMIN_PASS_SECRET}`) — Governance UI & API auth.
- `BIFROST_ENCRYPTION_KEY` (`${BIFROST_ENC_KEY_SECRET}`) — Encrypts provider keys and metadata in PostgreSQL `config_store`.
- `DEEPSEEK_API_KEY` — Commercial fallback key, reachable **only** when explicitly requested.

## Strict Zero-Cost ($0) Routing Policy

Bifrost CEL routing rules in [`config.json`](config.json) enforce zero-cost safety by default:
1. **Priority 0 (`rule-explicit-pro-override`)**: Routes to `deepseek` **only** when `headers['x-force-deepseek'] == 'true'`, `headers['x-routing-tier'] == 'pro'`, `headers['x-tier'] in ['pro', 'production']`, or `model.startsWith('deepseek')`.
2. **Priority 1 (`rule-freellm-opt-in-paid-fallback`)**: Routes to `freellmapi` with `deepseek` fallback **only** when the caller sets `x-allow-paid-fallback: true`.
3. **Priority 2 (`rule-freellm-zero-cost-strict`)**: Default `true` rule routes strictly to `freellmapi` with `fallbacks: []` (zero silent credit drain on commercial APIs).

## Schema & Config Store Notes (Verified vs Bifrost `config.schema.json`)
- Omit top-level legacy `version` (legacy schema compatibility mode).
- `retry_backoff_initial` and `retry_backoff_max` use integer milliseconds (e.g., `300`, `1000`, `500`, `3000`).
- `circuit_breaker_config` is an Enterprise-only top-level field; never place it under `governance`.
- `config_store` (PG18) persists `routing_rules` and `config_keys` across restarts; if updating `config.json` on an existing DB, reconcile `routing_rules` or use the Governance API.
- Always reference FreeLLMAPI models via `freellmapi/auto` or `auto` (`model: "freellmapi/auto"`), never non-existent DeepSeek aliases.
