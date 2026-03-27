---
name: synapseai-payments
metadata: {"skill_profile":{"version":"1.4.0","revision":"2026-03-27.3"},"wallet_cli":{"package":"@panda1105021243/wallet-cli-devtest","auto_update":"major","major_requires_confirm":false,"check_cmd":"npm view @panda1105021243/wallet-cli-devtest version","upgrade_cmd":"npm install -g @panda1105021243/wallet-cli-devtest@latest"}}
description: >
  Buyer-side wallet payment skill. Use for setup, readiness checks, discovering purchasable
  offers, and paying via x402/direct with wallet-cli.
---

# SynapseAI Payments

## Core Rules

1. Always use `wallet-cli` commands. Never replace with raw HTTP or unrelated APIs.
2. If user asks for x402, never downgrade to non-x402 data sources.
3. Before multi-step actions, run `wallet-cli whoami` once.
4. Keep clarification minimal: at most one blocking question.
5. If the user explicitly requests setup or re-setup, always run `wallet-cli register ...` to create a new agent, even if an active agent already exists.
6. If the caller passes custom `--headers`, keep header names and values HTTP-safe ASCII unless the target explicitly documents another encoding contract.

## CLI Sync Policy

Run version sync **once at session start**:
- `npm view @panda1105021243/wallet-cli-devtest version`
- `wallet-cli --version`

If local is behind, upgrade once:
- `npm install -g @panda1105021243/wallet-cli-devtest@latest`

After upgrade:
- `wallet-cli --version`
- `wallet-cli whoami`

## Intent Routing Contract

Route user intent to one command family:

- Setup / re-setup / re-register
  - Always create a new agent: `wallet-cli register ...`
  - Always switch default to the new agent immediately
  - Confirm with: `wallet-cli whoami --agent-id <new_agent_id>`
  - Reply must state: "已切换到新 Agent: <new_agent_id>"
  - wait for owner bind if needed

- Wallet status / readiness / policy / balance
  - Always run `wallet-cli whoami` before reporting status
  - For multi-agent checks, run `wallet-cli whoami --agent-id <id>` for each agent before summary
  - if troubleshooting: `wallet-cli doctor`

- Discover purchasable offers
  - Primary: `wallet-cli merchant catalog`
  - Fallback: `wallet-cli merchant offer list --status ACTIVE [--merchant-id <merchant_code>]`
  - Fallback for merchant existence: `wallet-cli merchant list`

- Spend via x402 (paid external API)
  - `wallet-cli spend x402 ...`

- Spend direct (internal balance deduction)
  - `wallet-cli spend direct ...`

- Multi-agent targeting
  - default: `current_agent_id`
  - temporary target: add `--agent-id`
  - change default only when user explicitly asks: `wallet-cli agent use --agent-id <id>`

## Parameter Resolution Contract

Auto-fill missing fields with deterministic defaults.

### Spend x402 defaults
- `method`: `GET`
- `amount-hint`: `0.001`
- `purpose`: `x402 call`
- If buying platform-internal offers, use merchant/endpoint from Discovery Gate results.

### Spend direct defaults
- `merchant`: `openai_api`

### URL resolution priority for x402
- If user provides URL: use it directly.
- If user intent is to buy platform-internal merchant offers: run Discovery Gate first and use discovered endpoint.
- Otherwise, ask one concise question for target endpoint/provider.

## Discovery Gate (Required for x402 purchase)

Before any x402 purchase, resolve all required fields:
- `merchant` (target merchant)
- `offer` (valid product/service)
- `unit_price` (price)
- `endpoint` / `target_url` (paid endpoint)

If any field is missing, do not pay. Return missing fields explicitly.

Recommended command sequence:
- `wallet-cli merchant catalog` (primary, agent context required)
- If catalog is incomplete or fails: `wallet-cli merchant offer list --status ACTIVE [--merchant-id <merchant_code>]` (agent context required)
- If merchant existence needs verification: `wallet-cli merchant list` (public discovery, no agent token required)

Parameter semantics:
- `--merchant-id` expects **merchant_code** (for example: `x402engine`), not database primary id.

`merchant catalog` should be treated as the canonical aggregated view for: merchant + active offers + unit_price + endpoint/payment_link.

## Payment Readiness Gate (Required)

Before any spend, ensure:
- `is_bound = true`
- `available_balance > 0`
- policy exists (`daily_limit`, `tx_limit`, `approval_threshold`)

If not ready, do not pay. Return exact missing items and next action in SynapseAI dashboard.

## Error Handling Contract

Use strict, actionable outcomes:

- `REQUIRE_APPROVAL`
  - Return `approval_url`.
  - Stop automatic retry.

- `REJECT` / `RISK_RULE_REJECTED`
  - Return `reason_code`.
  - Tell user to adjust limits/whitelist/policy in SynapseAI dashboard.

- `INSUFFICIENT_BALANCE`
  - Tell user to fund wallet.
  - Suggest minimum working balance (e.g. `>= 0.01 USDC`).

- Target fetch/network failure
  - Report target endpoint failure clearly.
  - Do not switch to non-x402 providers.

## Setup

1. Install CLI:
```bash
npm install -g @panda1105021243/wallet-cli-devtest@latest
```

2. Register:
```bash
wallet-cli register --name "MyBot" --desc "Your agent description" --cap api_purchase --cap subscription
```

3. Owner setup:
- Ask owner to open `bind_url` from register output and complete Owner Bind.
- In the same instruction, tell the owner that after bind they must fund the wallet and configure spending policy in SynapseAI dashboard.
- Wait for owner confirmation after all setup steps are complete.

4. Confirm readiness:
```bash
wallet-cli whoami
```

## Command Examples

```bash
# primary query path: purchasable catalog
wallet-cli merchant catalog

# fallback: list active offers
wallet-cli merchant offer list --status ACTIVE

# fallback: verify merchants
wallet-cli merchant list

# x402 spend
wallet-cli spend x402 --url <execute_url> --method GET --amount-hint 0.001 --purpose "purchase offer"

# direct spend
wallet-cli spend direct --merchant openai_api --amount 5.0 --purpose "GPT-4 API credits"

# health check
wallet-cli doctor
```

## State Rules

- State path is user-level only:
  - Windows: `%APPDATA%/synapseai-wallet-cli/state.json`
  - macOS/Linux: `~/.config/synapseai-wallet-cli/state.json`
- Do not use cwd-local state files for new workflows.
- `bind_url` in state is historical setup data, not active bind proof.
- Bind truth comes from `whoami` fields: `agent_status`, `is_bound`, `bind_url_validity`.

## Common Issues

| Issue | Cause | Action |
|---|---|---|
| `wallet-cli: command not found` | CLI missing | Install CLI and retry |
| `--token is required` | missing agent token in state | register target agent or use correct `--agent-id` |
| `REQUIRE_APPROVAL` | amount exceeds threshold | return approval URL, wait for owner |
| `REJECT` | policy violation | return reason and stop |
| insufficient balance | wallet not funded | fund wallet, retry |
| 401 invalid token | expired/wrong token | re-register |
| state missing | wrong path or first run | use user-level state path and register once |
| policy is null / missing limits | agent trading policy not configured | configure daily_limit / tx_limit / approval_threshold in SynapseAI dashboard |
