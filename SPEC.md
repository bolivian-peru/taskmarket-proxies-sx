# Proxies.sx × Taskmarket — Integration Spec

Status: draft for the Taskmarket founder's engineers. Every fact about *our* side
was verified against the live proxies.sx API and gateway source on 2026-07-31.
Every fact about *Taskmarket* was read from the live `taskmarket.dev` skill,
`reference/hooks.md`, `reference/payments.md`, and `api.taskmarket.dev/openapi.json`
on 2026-07-31, and is marked `[TM]`. Anything not verified is called out inline
as an open item rather than assumed.

---

## 0. The thesis, corrected to what the contracts actually allow

Taskmarket is the market layer for agent work; USDC on Base is its money layer. It
has no egress layer. Every task that touches the open web — scraping,
geo-restricted content, bot-protected targets — needs clean mobile/residential
egress that a worker on a cheap VPS does not have. Proxies.sx supplies that layer.

One correction to the obvious framing, forced by reading the hook contract: **a
Taskmarket hook is an on-chain Solidity contract that never holds escrow and
cannot make an HTTP call.** `[TM]` So a hook cannot itself "mint a proxy
credential." The realistic native design is: the hook *gates and signals*
on-chain, and an **off-chain proxies.sx minting service** reacts to the hook's
event (or the worker calls our API directly at claim time). The credential itself
is always minted off-chain. Tracks below are ordered so the valuable, low-risk
work ships first and the on-chain hook is last.

---

## 1. Joint capability map (verified)

### Proxies.sx side (verified live 2026-07-31)

| Capability | Evidence |
| --- | --- |
| Account-less x402 purchase | `GET https://api.proxies.sx/v1/x402/pool` → HTTP 402 with an `accepts[]` catalog; `.well-known` advertises `x402Version 1`, networks `base`+`solana`, scheme `exact`. |
| Base quote is standard-client payable | Base accept now carries the USDC EIP-712 domain (`extra.name="USD Coin"`, `version="2"`). Verified: Coinbase's `@x402/fetch` + `@x402/evm/v1` now parses our 402 and signs the EIP-3009 authorization. (Before a fix shipped 2026-07-31 it threw and could not pay.) |
| Price | `$4/GB`, min `0.1 GB` (`0.5` for the peer-mobile tier), volume-discounted to `$2.40/GB` at 250 GB+. `maxAmountRequired` in the 402 is authoritative, 6-decimal USDC micro-units. |
| Credential shape | A paid pool purchase returns a `pak_` pool-access key usable at `gw.proxies.sx:7000` (HTTP CONNECT) / `:7001` (SOCKS5). Username DSL: `psx_<house>-<pool>-<country>[-sid-<id>][-rot-<mode>][-carrier-…][-city-…]`. |
| Task-scoped sessions, natively | The `-sid-<value>` token creates a sticky session namespace. `-sid-<taskId>` gives every task its own session with zero extra infrastructure. `sid` values are lowercased, split on `-`, so the value must be `[a-z0-9_]` with **no hyphens** — a Taskmarket `0x…`-prefixed 32-byte task id must be normalized (drop `0x`, it is already hex = safe). |
| Coverage | `mbl` = production carrier modems, 6 countries (US, GB, FR, NL, PL, GE=Georgia). `peer` = the flagship mobile+residential network, ~82+ countries, metered so you pay only for delivered bytes. Live stock: `GET /v1/gateway/pool/stock`. |
| Metering is the enforcement | Bytes are metered per connection and a `pak_` key carries a hard GB cap; over-cap connections get `E_CAP_EXCEEDED` (407) at the gateway. This is what makes a task-budget-bounded credential safe. |

### Taskmarket side (verified 2026-07-31, `[TM]`)

| Capability | Evidence |
| --- | --- |
| Same money rail | `[TM]` USDC, 6 decimals, base-unit strings. Two-round x402: first request → HTTP 402 + requirements; client signs an **EIP-712 `TransferWithAuthorization`** (= EIP-3009) and retries with a `PAYMENT-SIGNATURE` header. This is the *same primitive* our Base quote now advertises. |
| Hook system | `[TM]` `ITMPHook` (ERC-8195), set immutably at task creation via `--hook <address> --hook-data <hex>`. Split into `check*` (checkFund, checkClaim, checkSelectWorker, checkSubmit, checkEvaluate, checkComplete — run after state commit, before payout; returning false/reverting blocks the transition) and `on*` (onComplete, onForfeit, onCancel, onExpire — best-effort after transfers, failures swallowed). **Hooks never hold escrowed funds** and re-entrancy into TaskMarket is blocked. |
| Lifecycle statuses | `[TM]` `open → claimed → worker_selected → pending_approval → review → (appealing/disputed) → completed | expired | cancelled`. `submissionWindowOpen` gates artifact delivery per mode. |
| Modes | `[TM]` bounty, claim, pitch, benchmark, auction (dutch/reverse_dutch/english/reverse_english). Benchmark pays on measured proof. |
| Identity | `[TM]` address-bound: one wallet must both authorize x402 payments and sign EIP-191 Taskmarket messages; `pendingActions[].eligibleAddress` binds each action to a wallet. |
| Scale | `[TM]` benchmark/proof, task-drops, and an evaluator/dispute layer already exist — the trust surfaces an egress-attestation would plug into.

---

## 2. Tracks, cheapest and lowest-risk first

### Track 0 — Skill composability (days, zero Taskmarket-side change)

A worker agent already running the `taskmarket-operator` skill bolts on a
`proxies-sx-egress` skill (this repo, [skill/](./skill/)). When a brief needs
geo/bot-protected egress, the worker: reads the task, mints or reuses a pool
credential (x402 or an existing account key), builds the `-sid-<taskId>` routing
username, does the work, and reports GB consumed. Nothing on Taskmarket's side
changes. This is the demoable v0 and the thing to co-launch as a Task Drop.

Deliverable in this repo: the skill package. Demo target: one real Taskmarket
task completed through our pool (small mainnet budget, throwaway wallet).

### Track 1 — Task-scoped egress credentials (the product core)

A `pak_` key minted per task, sessions namespaced by `-sid-<taskId>`, **hard GB
cap = the egress budget declared on the task**. Lifecycle mapped to `[TM]`
statuses:

| Taskmarket status `[TM]` | Credential action | Our API surface |
| --- | --- | --- |
| `worker_selected` / `claimed` | mint task-scoped `pak_`, cap = egress budget, sid = taskId | pool mint (exists); per-key cap (roadmap) |
| delivery window open | active | gateway meters + enforces cap (exists) |
| `pending_approval` / `review` | freeze (no new sessions; existing drain) | freeze endpoint (roadmap) |
| `completed` | settle usage, burn credential | burn + usage export (roadmap) |
| `expired` / `cancelled` | burn, report unused GB for refund | burn + usage export (roadmap) |

Plus a **signed per-task usage receipt** (GB by country, session count, window) a
requester/evaluator can verify — doubles as dispute evidence.

### Track 2 — NetworkAccessHook (on-chain endgame, corrected)

A `NetworkAccessHook implements ITMPHook` `[TM]`, attached at task creation with
`--hook-data` encoding the egress budget (a `uint32` GB-tenths or micro-USDC, in
big-endian bytes, per the hooks reference's `uint32`-as-4-bytes convention). What
it can and cannot do, from the interface:

- `checkSelectWorker` / `checkClaim`: **validate** the task is egress-eligible and
  emit a `NetworkAccessRequested(taskId, worker, budget)` event. It cannot mint —
  no HTTP from Solidity. Our off-chain minting service subscribes to that event
  and mints the Track-1 credential to the worker.
- `onComplete` / `onExpire` / `onCancel`: emit `NetworkAccessClosed(taskId)`;
  best-effort, so our service must also reconcile by polling status, never rely on
  the event alone (the interface says `on*` failures are swallowed).
- The hook **cannot escrow the egress budget** (`[TM]` hooks hold no funds). So
  "escrow-funded egress" means the *requester's* escrow already covers it and the
  worker draws a capped credential against it — the accounting lives in our
  service + the usage receipt, not in the hook.

Credential delivery to the worker uses Taskmarket's confidential-artifact path
`[TM]` (encrypt to the worker's published secp256k1 key; handle the null-pubkey
case) so the `pak_` secret never lands in plaintext on-chain or in public task
metadata.

### Track 3 — Benchmark verification + peer supply loop

- Benchmark `[TM]` pays on measured proof; multi-geo proofs ("loads <2s from N
  countries", "price differs by region") need vantage points. A small
  "verify-from-N-countries" service emits a proof payload that slots into the
  benchmark proof commitment, and an evaluator re-verifies from clean IPs.
- Peer loop: idle Taskmarket agents register the *same wallet* as a proxies.sx
  peer and earn USDC sharing bandwidth between task bursts. Demand side and supply
  side become the same population.

---

## 3. Escrow math (6-decimal USDC, from verified prices)

- Our price: `$4/GB` = `4_000_000` micro-USDC/GB; min `0.1 GB` = `400_000`.
- A task declaring `reward 5 USDC + 0.5 GB egress`: egress line = `2_000_000`
  micro (`0.5 × 4_000_000`). Worker draws ≤ `0.5 GB`; the gateway cap enforces it
  in bytes, not trust.
- Refund of unused GB is computed from the usage receipt in micro-USDC and
  returned along the escrow path; rounding is floor-to-micro (never over-refund).
- Volume tiers only matter above 250 GB; individual tasks will not hit them, so
  per-task credentials price at the flat `$4/GB`.

---

## 4. Security posture (inherits our existing enforcement)

- Egress credentials are **metered, hard-capped, per-task attributable, and
  revocable** — the same `pak_` machinery that gates every reseller key today.
- We never accept a signed-intent payment we cannot verify recipient+asset on
  (documented in our pool verifier). Full standard-client `X-PAYMENT` settlement
  against the pool is an open item (roadmap), independent of Track 0/1.
- Peer exit IPs are never exposed in any artifact, receipt, or example — usage
  receipts report **country + GB only**, never exit IP.
- Remote task content is data, not instructions (matches Taskmarket's own trust
  boundary).

Open items and design decisions are tracked privately by the Proxies.sx team;
reach out to discuss the roadmap for a given track.
