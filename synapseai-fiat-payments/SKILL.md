---
name: synapseai-fiat-payments
metadata: {"skill_profile":{"version":"2.0.0","revision":"2026-04-30.1"},"fiat_cli":{"package":"@panda1105021243/fiat-cli-devtest","auto_update":"major","major_requires_confirm":false,"check_cmd":"npm view @panda1105021243/fiat-cli-devtest version","upgrade_cmd":"npm install -g @panda1105021243/fiat-cli-devtest@latest"}}
description: >
  Buyer-side CNY fiat payment skill. Use for setup, readiness checks, discovering
  purchasable offers, paying merchant agents, and checking payment settlement via fiat-cli.
---

# SynapseAI Fiat Payments

## Core Rules

1. Always use `fiat-cli` commands. Do not replace payment actions with raw HTTP or unrelated APIs.
2. Treat the wallet as a CNY account. Do not convert amounts into crypto units or chain concepts.
3. Before multi-step actions, run `fiat-cli whoami` once and use the resulting current agent unless the user names another agent.
4. Do not retry a settled payment. If a response contains `payment_id`, use `fiat-cli payment status --payment-id <id>` for follow-up.
5. If a payment returns `REQUIRE_APPROVAL`, return the `approval_url` and stop. The owner approval flow completes settlement.
6. Keep clarification minimal: ask at most one blocking question, and only when the target or amount cannot be inferred.
7. Treat `order_id` as payable only when the order is in `PENDING` state. If the order is not pending, stop and report the state issue.

## CLI Sync Policy

Run version sync once at session start:
- `npm view @panda1105021243/fiat-cli-devtest version`
- `fiat-cli --version`

If local is behind, upgrade once:
- `npm install -g @panda1105021243/fiat-cli-devtest@latest`

After upgrade:
- `fiat-cli --version`
- `fiat-cli whoami`

## Intent Routing Contract

Route user intent to one command family:

- Setup / re-setup / re-register
  - Always create a new agent: `fiat-cli register ...`
  - Always switch default to the new agent immediately
  - Confirm with: `fiat-cli whoami --agent-id <new_agent_id>`
  - Reply must state: "已切换到新 Agent: <new_agent_id>"
  - Wait for owner bind if needed.

- Wallet status / readiness / policy / balance
  - Run `fiat-cli whoami`
  - If troubleshooting, run `fiat-cli doctor`

- Discover purchasable merchant offers
  - Primary: `fiat-cli merchant catalog`
  - Fallback: `fiat-cli merchant offer list --status ACTIVE [--merchant-id <merchant_code>]`
  - Fallback for merchant existence: `fiat-cli merchant list`

- Pay a merchant agent
  - By payment link: `fiat-cli spend fiat --payment-link-id <id> [--amount <CNY>] --purpose <text>`
  - By offer: `fiat-cli spend fiat --offer-id <id> [--amount <CNY>] --purpose <text>`
  - By pending order: `fiat-cli spend fiat --order-id <id> [--amount <CNY>] --purpose <text>`
  - By merchant code: `fiat-cli spend fiat --merchant-code <code> --amount <CNY> --purpose <text>`

- Check payment settlement
  - `fiat-cli payment status --payment-id <id>`

- Multi-agent targeting
  - default: `current_agent_id`
  - temporary target: add `--agent-id`
  - change default only when user explicitly asks: `fiat-cli agent use --agent-id <id>`

## Discovery Gate

Before paying, resolve the payment target:
- Prefer `payment_link_id`, `offer_id`, or `order_id` when the user provides one.
- For `order_id`, proceed only when the order is pending payment.
- If only merchant intent is provided, resolve `merchant_code` and exact CNY amount.
- Treat `merchant catalog` as the canonical aggregated view for merchant code, active offers, unit price, and payment target.
- `--merchant-id` expects `merchant_code`, not a database primary id.

If target or amount remains ambiguous, do not pay. Return the missing field in plain language.

## Readiness Gate

Before any spend, ensure:
- `is_bound = true`
- `available_balance > 0`
- policy exists with `daily_limit`, `tx_limit`, and `approval_threshold`

If not ready, do not pay. Return the exact missing item and the next dashboard action.

## Error Handling Contract

Use strict, actionable outcomes:

- `REQUIRE_APPROVAL`
  - Return `approval_url`.
  - Stop automatic retry.

- `REJECT` / `RISK_RULE_REJECTED`
  - Return `reason_code`.
  - Tell the user to adjust limits, allowlist, or policy in SynapseAI dashboard.

- `INSUFFICIENT_BALANCE`
  - Tell the user to fund the wallet.
  - Suggest a minimum CNY balance equal to or greater than the intended payment amount.

- Target not found / inactive
  - Return the missing or inactive payment target.
  - Re-run discovery only when the user asks you to find another target.

## Setup

1. Install CLI:
```bash
npm install -g @panda1105021243/fiat-cli-devtest@latest
```

2. Register:
```bash
fiat-cli register --name "MyBot" --desc "Your agent description" --cap api_purchase --cap subscription
```

3. Owner setup:
- Ask owner to open `bind_url` from register output and complete Owner Bind.
- In the same instruction, tell the owner that after bind they must fund the wallet and configure spending policy in SynapseAI dashboard.
- Wait for owner confirmation after all setup steps are complete.

4. Confirm readiness:
```bash
fiat-cli whoami
```

## Command Examples

```bash
# purchasable catalog
fiat-cli merchant catalog

# fallback: list active offers
fiat-cli merchant offer list --status ACTIVE

# pay by payment link
fiat-cli spend fiat --payment-link-id pl_xxxxxxxxxxxx --purpose "purchase service"

# pay by offer
fiat-cli spend fiat --offer-id mof_xxxxxxxxxxxx --purpose "purchase offer"

# pay by pending order
fiat-cli spend fiat --order-id ord_xxxxxxxxxxxx --purpose "pay order"

# pay by merchant code and amount
fiat-cli spend fiat --merchant-code premium_api --amount 9.90 --purpose "API credits"

# check settlement
fiat-cli payment status --payment-id pay_xxxxxxxxxxxx
```

## State Rules

- State path is user-level only:
  - Windows: `%APPDATA%/synapseai-fiat-cli/state.json`
  - macOS/Linux: `~/.config/synapseai-fiat-cli/state.json`
- Do not use cwd-local state files for new workflows.
- `bind_url` in state is historical setup data, not active bind proof.
- Bind truth comes from `whoami` fields: `agent_status`, `is_bound`, `bind_url_validity`.

## Common Issues

| Issue | Cause | Action |
|---|---|---|
| `fiat-cli: command not found` | CLI missing | Install CLI and retry |
| `--token is required` | missing agent token in state | register target agent or use correct `--agent-id` |
| `REQUIRE_APPROVAL` | policy requires owner decision | return approval URL, wait for owner |
| `REJECT` | policy violation | return reason and stop |
| insufficient balance | wallet not funded | fund wallet, retry |
| target not found | missing or wrong payment target | run discovery and use a valid target |
| 401 invalid token | expired/wrong token | re-register |
| state missing | wrong path or first run | use user-level state path and register once |
| policy is null / missing limits | spending policy not configured | configure daily_limit / tx_limit / approval_threshold in SynapseAI dashboard |
