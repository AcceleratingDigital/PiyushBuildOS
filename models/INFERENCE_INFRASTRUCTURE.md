# Inference Infrastructure — Staged Self-Host Plan

> **Decision date:** 2026-08-18
> **Status:** Approved — external-chassis path, staged capex
> **Owner:** Piyush
> **Full research:** `~/Downloads/hermes/used-gpu-buildout-aug2026.md` (chassis-grounded BOMs, used pricing, benchmark citations)

## Decision Summary

Replace the long-term dependency on free `algolia/*` models (LiteLLM cloud-proxied) with
**self-hosted inference on a dedicated external GPU chassis**, layered in over time.
Capex-funded (current-year project income, Section 179/bonus depreciation), zero monthly burn.

### Options evaluated and rejected

| Option | Verdict | Why |
|---|---|---|
| 2× NVIDIA DGX Spark ($9.4K) | ❌ | 273 GB/s bandwidth wall. ~5–10 concurrent streams on 30B-class models; team needs 60–70 peak streams on frontier-class models. Order of magnitude short. |
| 2× ASUS GX10 / other GB10 clones | ❌ | Identical GB10 silicon, same wall, just $800 cheaper. |
| Mac Studio M3 Ultra 512GB | ❌ | Prefill collapses on big models (227s for a 16K prompt on DeepSeek V3); MLX serving is effectively single-user; high-mem SKUs discontinued (refurb only). |
| Retrofit R740 with 3× GPUs (option-16 risers) | ❌ | Requires riser conversion + 2400W PSUs + 208V + thermal risk on the primary hypervisor. Son's call confirmed: **no more cards inside the R740.** |
| Hosted open-model APIs (~$1.4–2K/mo) | ⚠️ Interim | Best $/quality (MiniMax M2.5 = 75.8% SWE-bench) but recurring opex — kept as escalation/fallback tier, not the base. |
| **External used GPU chassis, cards added in stages** | ✅ | R740/R740xd untouched. One chassis purchase = 8-slot runway. Every stage's hardware stays useful in the endgame. |

## Sizing Basis (measured, not guessed)

From real agentic-coding trace research (TraceLab arXiv:2606.30560, vLLM/Mooncake, viberank):

| Metric | 20 devs + 20 background agents |
|---|---|
| Metered tokens/day | ~1B fleet-wide (median dev 10–30M, heavy 50–80M) |
| Input:output ratio | 25:1–131:1 — **prefill-dominated**; 95% should be prefix-cache hits |
| Concurrent decode streams | 15–25 sustained, 60–70 peak |
| Aggregate decode | ~600 tok/s sustained, ~2,500 peak |
| Aggregate prefill | 20–50K tok/s peak (TTFT-critical) |
| Warm KV cache | 120–500GB for ~40 sessions @ 80K ctx |

**Implication:** prioritize prefix caching (vLLM APC), session-affinity routing, and VRAM
headroom for KV — the cluster is prefill/KV-bound, not decode-bound.

## Target Architecture

```
Devs / agents → LiteLLM (10.1.2.13:4000, unchanged) → Model Router policy
                   ├── local vLLM replicas  (external GPU box — new)
                   ├── hosted open APIs     (escalation / overflow)
                   └── frontier APIs        (Claude/GPT per MODEL_SELECTION_POLICY.md)
```

- **R740 / R740xd: NEVER modified.** No new cards, risers, or PSUs. They remain
  Proxmox/SAN workhorses. (Constraint set 2026-08-18.)
- External box runs **bare-metal Ubuntu + vLLM** — deliberately NOT in Proxmox
  (VFIO passthrough kills PCIe P2P; vLLM falls back to slow allreduce).
- Model weights on SAN/unraid via NFS; hot models cached on local NVMe.
- Failure blast radius: zero — LiteLLM falls back to hosted routes.

## Staged Build Plan

### Phase 0 — $0 — Telemetry ✅ ALREADY LIVE
LiteLLM has logged every request to Postgres (`LiteLLM_SpendLogs`, db `litellm` @ 10.1.128.8)
since 2026-03-22. Measured baseline (as of 2026-08-18, current fleet — NOT yet 20 devs):

| Metric | Measured |
|---|---|
| Total since March | 79K requests, 6.02B input / 29.5M output tokens |
| Recent daily (Aug 4–18) | 8M–363M input tokens/day (active days: 200–360M) |
| Input:output ratio | **~200:1** — confirms prefill-dominated agentic profile |
| Peak request rate | 101 req/min (14-day window) |
| Dominant models | xlarge (1.7B in/14d), large (450M), Claude opus/sonnet (460M) |

Sizing query (rerun anytime):
`SELECT "startTime"::date, count(*), sum(prompt_tokens), sum(completion_tokens) FROM "LiteLLM_SpendLogs" GROUP BY 1;`
(user `litellm`, password in `~/litellm/.env` on 10.1.2.13)

**Scale-up math:** current ~300M in/day → 20 devs + agents ≈ 1B+/day. The 200:1 ratio
means prefix caching is worth ~20× effective capacity — non-negotiable in vLLM config.

### Phase 1 — ~$5.5–6.5K — The chassis (foundation)
| Item | Est. cost |
|---|---|
| Used EPYC PCIe-**Gen4** 8-GPU barebones (Supermicro 4124GS-TNR ~$3.8K / ASUS ESC8000A-E11 ~$3.6K) | $3.6–3.8K |
| EPYC 7443-class CPU + 512GB DDR4 | $1.5–2.5K |
| 2× NVMe (model cache) + 10GbE NIC | $300–500 |

Gen4 (not a cheaper Gen3 4029GP) so the chassis never needs replacing.
**Pre-purchase check:** rack space (4U), circuit capacity — 1–2 cards fine on 110V/15A;
3–4× Pro 6000 needs ~2.5–3.2kW → 208V or dedicated 20A. Confirm with datacenter owner (son).

### Phase 2 — first card — **preference: RTX PRO 6000 Blackwell Max-Q 96GB**
- **Primary target:** 1× RTX PRO 6000 Max-Q (~$10–15K used/gray; new/OEM ~$14.8K).
  Serves gpt-oss-120b fast; current-gen silicon; part of the 4-card endgame.
- **Supply fallback:** if no Pro 6000 at sane price (supply is tight, Aug 2026;
  sub-$3.3K listings are scams), start with 1× **RTX A6000 48GB** (~$3.6–4.2K executed)
  → Qwen3-Coder-30B FP8 replica; A6000 later becomes the QA/utility-tier card.
- Source as **tested pulls** from ITAD dealers (ITCreations, UnixSurplus, Bargain
  Hardware, ETB) with 30-day returns. Avoid SXM→PCIe converted cards.
- **Gate:** vLLM + LiteLLM routing holds under real dev traffic; measure % of
  pipeline traffic absorbed locally.

### Phase 3+ — one Pro 6000 at a time (buy on gate, not on schedule)
| Cards | VRAM | Unlocks |
|---|---|---|
| 2× | 192GB | MiniMax M2.5 AWQ (75.8% SWE-bench) — first frontier-class local model |
| 3× | 288GB | M2.5 comfortable: 15–25 streams + large KV |
| 4× | 384GB | **M2.5 FP8 — public benchmark on this exact config: 15.7K tok/s prefill, ~41 concurrent @8K ctx, <10s TTFT** (millstoneai.com) |
| 5–8× | — | Open runway (GLM-5-class @4-bit someday) |

**Acceleration trigger:** algolia announces pricing → telemetry says exactly how many
cards the real workload needs. Prices on Pro 6000s drift down; waiting is paid.

### Explicitly rejected hardware
- A100 80GB (5+ yrs old, –20–30%/yr resale, no service-life thesis)
- HGX A100 SXM servers (208V/6kW, $60K+, old silicon)
- MI300X (OAM-only; needs $150K+ UBB chassis)
- L4 (poor $/GB at $3.2K used)

## Model Policy Mapping (when local tiers come online)

| MODEL_SELECTION_POLICY tier | Today | Endgame |
|---|---|---|
| QA / medium | algolia/medium | local Qwen3-Coder-30B replica |
| Core coding / xlarge | algolia/xlarge | local MiniMax M2.5 (2+ Pro 6000s) |
| Docs/design | claude-sonnet-4-6 | unchanged (hosted) |
| Deep security review | gpt-5.6-sol | unchanged (hosted, model diversity) |
| Escalation | Claude | hosted open APIs first, then frontier |

S-S-D holds: specs never name models; the Model Router decides. This document only
changes what the router can route TO.

## Gates & Kill Criteria

| After | Continue if | Stop/hold if |
|---|---|---|
| Phase 0 | telemetry shows sustained multi-M tok/day demand | usage trivially covered by hosted plans |
| Phase 2 | local replica absorbs meaningful traffic, ops burden acceptable | vLLM ops overhead not worth it → stay hosted |
| Each card | telemetry shows saturation (queue depth, TTFT p95) | headroom remains |
