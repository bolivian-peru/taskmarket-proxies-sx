# EXT-002: Network Access Hook (proposal)

**Status:** Proposed
**Type:** Daydreams Extension (not a TMP protocol standard)
**Depends on:** ERC-8195 `ITMPHook`, `rev003-hooks-evaluator-registry`, `rev010-multi-hook`

Modelled on [EXT-001: Token Reward Hook](https://github.com/daydreamsai/taskmarket-contracts/blob/main/docs/extensions/ext-001-token-reward-hook.md).
Like EXT-001, it is an opt-in ERC-8195 hook — no core protocol changes, tasks
without the hook are unaffected, and it holds no escrowed funds.

---

## Overview

A task whose brief touches the web (scraping, geo-restricted content,
bot-protected targets) needs clean mobile/residential egress the worker's own IP
can't provide. EXT-002 lets a requester attach a **per-task egress budget** at
task creation. The hook records the budget on-chain and signals when a worker is
entitled to draw egress; an off-chain minting service issues a metered,
GB-capped proxy credential scoped to the task.

The hook itself moves no money and makes no external calls — it validates and
emits events. All value transfer stays in the worker's wallet (the egress is
paid via x402/USDC, the same rail TMP settles on), and the credential is
metered + hard-capped so a task can never exceed its declared budget.

---

## Why a hook (and why nothing else changes)

- **checkFund** is the only call carrying `hookData`, and per the hooks
  reference it runs inside `createTask` after the PGTR forwarder has moved USDC.
  EXT-002 decodes the egress budget from `hookData` (a `uint32`, GB-tenths,
  4 big-endian bytes — same convention as the TWAP window in the hooks doc) and
  stores `budget[taskId]`.
- **checkClaim / checkSelectWorker** (`(taskId, ctx, worker)`) run after state
  commit, before payout. EXT-002 returns `true` and emits
  `NetworkAccessGranted(taskId, worker, budgetTenthsGb)`. It cannot mint — no
  HTTP from Solidity — so an off-chain service subscribes to this event and
  issues the credential to `worker`.
- **onComplete / onForfeit / onCancel / onExpire** emit
  `NetworkAccessClosed(taskId)`. These are best-effort (swallowed on failure per
  the interface), so the off-chain service must also reconcile by polling task
  status — it never relies on the event alone.

No Diamond storage is touched; the hook is wired via `setDefaultHooks()` /
per-task `--hook`, exactly like EXT-001.

---

## Credential lifecycle (off-chain, keyed by taskId)

| TMP status | Off-chain action |
|---|---|
| `worker_selected` / `claimed` (event) | mint task-scoped credential, hard GB cap = budget, session id = taskId |
| delivery window | metered; over-cap requests are refused at the gateway |
| `pending_approval` / `review` | freeze (no new sessions) |
| `completed` | settle usage, burn credential, emit signed usage receipt |
| `expired` / `cancelled` | burn, report unused GB for refund |

**Credential delivery** uses the existing confidential-artifact path (encrypt to
the worker's published secp256k1 key); if no pubkey is published, the worker
mints its own credential at claim time instead.

---

## What this unlocks for TMP

- **Web-touching tasks stop being blocked on worker infrastructure.** A worker on
  a cheap VPS can complete geo/bot-protected briefs without owning a proxy stack.
- **Attested exit country in the receipt** turns geo-dependent acceptance ("did
  they really check the German price?") from a judgement call into something the
  evaluator/dispute layer can verify. This strengthens TMP's trust surface, not
  just the worker's capability.

---

## Open design questions (yours to decide)

1. **Escrow-funded egress.** Should a declared slice of the requester's escrow be
   an egress allowance the worker draws against, rather than the worker fronting
   USDC? Hooks hold no funds, so the accounting would live off-chain against the
   usage receipt — but the escrow split is a core-side decision.
2. **Who implements.** We can contribute the EXT-002 doc + a reference
   `NetworkAccessHook.sol` (Foundry-tested against `ITMPHook`), or the extension
   can live under Daydreams and we supply only the off-chain minting service +
   the worker skill. Either works for us.

---

## Already live (verifiable today)

- Worker skill (uses only shipped endpoints): PR to `daydreamsai/skills-market`.
- Reference + full spec: https://github.com/bolivian-peru/taskmarket-proxies-sx
- The egress rail, account-less: `curl -i https://api.proxies.sx/v1/x402/pool`
  returns HTTP 402 with an `accepts[]` catalog (scheme `exact`, base + solana,
  USDC); `GET /v1/x402/.well-known` returns `x402Version 1`.
