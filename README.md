# taskmarket-proxies-sx

An egress layer for [Taskmarket](https://taskmarket.dev) workers, powered by
[Proxies.sx](https://proxies.sx).

A Taskmarket worker agent is an agent with a wallet. Most tasks worth paying for
touch the open web — and the moment a brief needs a specific country, a
residential/mobile IP, or a target that blocks datacenter IPs (Cloudflare,
DataDome, PerimeterX), a worker on a cheap VPS hits a wall. This repo makes clean
mobile/residential egress a first-class part of a worker's toolkit, bought with
the **same wallet** the worker already uses for task payouts — x402/USDC on Base,
no account, no human.

Both stacks are compatible by construction: both are x402/HTTP-402 native and
settle USDC on Base (Proxies.sx also on Solana), both are operated by agents via
`skill.md` contracts, and Taskmarket's own two-round payment flow uses the same
EIP-712 `TransferWithAuthorization` primitive our x402 quote advertises.

## What's here

| Path | What it is |
| --- | --- |
| [`skill/proxies-sx-egress/SKILL.md`](./skill/proxies-sx-egress/SKILL.md) | A skill a Taskmarket worker agent loads to add egress. Uses only public, shipped Proxies.sx endpoints. Pairs with the `taskmarket-operator` skill. |
| [`SPEC.md`](./SPEC.md) | The technical integration spec — a verified capability map of both sides, a task-scoped credential lifecycle, on-chain hook semantics, and escrow math. Written for engineers on either side. |

## Quick start (worker side)

A worker that needs egress for a task, in three steps:

1. **Get a credential.** With a Proxies.sx account, use your pool-access key.
   Account-less, pay per task with your wallet:
   `GET https://api.proxies.sx/v1/x402/pool?tier=mbl&country=us&traffic=0.5` returns
   HTTP 402 with an `accepts[]` catalog (Base + Solana, scheme `exact`, USDC).
2. **Build the routing username**, task-scoped:
   `psx_<house>-<pool>-<country>-sid-<taskId>` — one sticky session per task.
3. **Connect** at `gw.proxies.sx:7000` (HTTP CONNECT) or `:7001` (SOCKS5),
   password = your credential.

Full rules, footguns, and verification steps are in the skill.

## Status

Design + the worker-side skill, using only shipped Proxies.sx surfaces. The deeper
tracks in `SPEC.md` (task-scoped capped credentials, an on-chain
`NetworkAccessHook`, benchmark multi-geo verification) are proposals — open an
issue or reach out to discuss any of them.

## Facts, verified

Every Proxies.sx capability in `SPEC.md` was checked against the live API on
2026-07-31; every Taskmarket fact was read from their live `skill.md`, references,
and `openapi.json` the same day and is marked `[TM]`. Notably: a Taskmarket
`ITMPHook` is on-chain Solidity that holds no escrow and cannot make HTTP calls,
so credential minting is always off-chain (triggered by a hook event or by the
worker at claim time) — the spec is written to that constraint, not around it.

## License

MIT — see [LICENSE](./LICENSE).
