---
name: synapseai-payments
metadata: {"skill_profile":{"version":"1.1.1","revision":"2026-03-24.6"},"wallet_cli":{"package":"@panda1105021243/wallet-cli-devtest","auto_update":"major","major_requires_confirm":false,"check_cmd":"npm view @panda1105021243/wallet-cli-devtest version","upgrade_cmd":"npm install -g @panda1105021243/wallet-cli-devtest@latest"}}
description: >
  Unified wallet payment skill. Use for wallet-cli setup, x402 spend, direct spend,
  receiving wallets, payment links, webhook config, and wallet status checks.
  If user says "use wallet-cli" or mentions x402, always execute via wallet-cli.
---

# SynapseAI Payments

## Core Rules

1. Always use `wallet-cli` commands. Never replace with raw HTTP or unrelated APIs.
2. If user asks for x402, never downgrade to non-x402 data sources.
3. Before multi-step actions, run `wallet-cli whoami` once.
4. Keep clarification minimal: at most one blocking question.

## CLI Sync Policy

Before execution:
- `npm view @panda1105021243/wallet-cli-devtest version`
- `wallet-cli --version`

If local is behind, upgrade:
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
  - Always run `wallet-cli whoami` before reporting status (this syncs local `wallet_status`)
  - For multi-agent checks, run `wallet-cli whoami --agent-id <id>` for each agent before summary
  - if troubleshooting: `wallet-cli doctor`

- Spend via x402 (paid external API)
  - `wallet-cli spend x402 ...`

- Spend direct (internal balance deduction)
  - `wallet-cli spend direct ...`

- Receive / merchant operations
  - `wallet-cli receive enable`
  - `wallet-cli receive wallet create ...`
  - `wallet-cli receive link create ...`
  - `wallet-cli receive webhook set ...`
  - `wallet-cli receive status`

- Multi-agent targeting
  - default: `current_agent_id`
  - temporary target: add `--agent-id`
  - change default only when user explicitly asks: `wallet-cli agent use --agent-id <id>`

## Parameter Resolution Contract

Auto-fill missing fields with deterministic defaults.

### Spend x402 defaults
- `merchant`: `x402engine`
- `method`: `GET`
- `amount-hint`: `0.001`
- `purpose`: `x402 call`

### Spend direct defaults
- `merchant`: `openai_api`

### Receive defaults
- `network`: `BASE`
- `currency`: `USDC`
- wallet `label`: `Primary Receiving Wallet`
- webhook `events`: `payment.success`

### URL resolution for x402
- If user provides URL: use it directly.
- If user does not provide URL: use built-in task templates.
- If no template matches: ask one concise question for target endpoint/provider.

Built-in x402 task templates:

- `btc_price`
  - URL: `https://x402-gateway-production.up.railway.app/api/crypto/price?ids=bitcoin`
  - Method: `GET`
  - Merchant: `x402engine`
  - Amount hint: `0.001`
  - Purpose: `x402 btc price lookup`

- `gpt4o_mini_hello`
  - URL: `https://x402-gateway-production.up.railway.app/api/llm/gpt-4o-mini`
  - Method: `POST`
  - Body: `{"messages":[{"role":"user","content":"Say hello"}],"max_tokens":32}`
  - Merchant: `x402engine`
  - Amount hint: `0.003`
  - Purpose: `x402 llm hello`

## Payment Readiness Gate (Required)

Before any spend, ensure:
- `is_bound = true`
- `available_balance > 0`
- policy exists (`daily_limit`, `tx_limit`, `approval_threshold`)

If not ready, do not pay. Return exact missing items and next action.

## Error Handling Contract

Use strict, actionable outcomes:

- `REQUIRE_APPROVAL`
  - Return `approval_url`.
  - Stop automatic retry.

- `REJECT` / `RISK_RULE_REJECTED`
  - Return `reason_code`.
  - Tell user to adjust limits/whitelist/policy.

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

3. Owner bind:
- Ask owner to open `bind_url` from register output.
- Wait for owner confirmation.

4. Confirm readiness:
```bash
wallet-cli whoami
```

## Command Examples

```bash
# x402 spend
wallet-cli spend x402 --url https://x402-gateway-production.up.railway.app/api/crypto/price?ids=bitcoin --method GET --amount-hint 0.001 --purpose "x402 btc price lookup"

# direct spend
wallet-cli spend direct --merchant openai_api --amount 5.0 --purpose "GPT-4 API credits"

# receive enable
wallet-cli receive enable

# create receiving wallet
wallet-cli receive wallet create --label "Premium API Store"

# create payment link
wallet-cli receive link create --amount 10.0 --description "Premium API Access - 30 days"

# set webhook
wallet-cli receive webhook set --url https://your-api.com/callback --events payment.success,payment.failed

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
| 403 merchant disabled | merchant not enabled | run `wallet-cli receive enable` |
| state missing | wrong path or first run | use user-level state path and register once |
