# Host-retaining admission patch

Base: natpate/ninfer-windows v0.9.0. This local patch targets unnecessary loss of
inactive conversations when admission planning stops early.

## Behavior

- Before the combinatorial pressure search, establish a verified incumbent by
  proposing host demotion of eligible private and shared owners. Already-host-only
  owners remain unchanged. The selected source and active owners remain protected
  by the existing Program and ResourceManager eligibility rules.
- Build the collective demotion proposal with one action per physical replica.
  Shared logical KV pages and state replicas already proposed by an earlier owner
  are excluded from subsequent proposals. Exact projection continues rejecting
  aliased actions; no allocator or publication safety check is relaxed.
- If that does not fit, assess targets sacrificing an increasing oldest-first set
  (1, 2, 4, ...), then narrow a feasible interval. Recency includes both publication
  and subsequent hits; publishing a new conversation does not count as a cache hit.
- Try the candidate with the best optimistic request cost and the root candidate.
  At most 24 seed assessments use at most half the remaining admission boundary,
  subject to indivisible-operation overrun. No monotonic-feasibility assumption is
  used to admit a request: every adopted target passes exact physical projection
  and logical publication checks, then normal sealing.
- Rank verified seed targets by owner evictions, then checkpoint drops, then cost.
  Optional refinement cannot increase either destructive count above the current
  verified incumbent. This protects retention even when eviction is estimated to
  be faster. It can spend more time transferring cache or choose a slower source.
  Search remains bounded and is not a proof of globally minimal eviction.
- Retain the original all-release fallback only if no feasible retention incumbent
  is found. The ordinary search can improve the incumbent within its retention ceiling.
- Keep idle admission allowance at 50 ms and increase busy allowance from 10
  to 50 ms. Scale the initial optional grant with incumbent cost, retaining the
  economic cap, affected-request divisor, cancellation and control deadlines.
  Cheap requests do not receive an unconditional 5 ms floor.

The host-demotion proposal is adapted from Ian Ranson's public ninfer-custom
commit 6fe061d3b60d73067a851b0f72c4bfd6b59288a9. Budget scaling is informed by
e6a9922b8c35e2ef9947551f7bb7d2cd068f6b27 and upstream issue 229. This is a focused
adaptation, not a merge of that fork's other scheduler, model, or speculation work.

## Validation on 2026-09-28

Built the full server with CUDA 13.1.80/MSVC 14.44 for RTX 5090. Both affected CTest
targets passed: materialization budget and resource manager. Tests cover deadlines,
source protection, oldest-first necessary sacrifice, and preservation despite cheaper
eviction; the independent small-problem oracle uses the new retention preference.
The same preservation regressions fail against stock headers.

Two final 150-request GPU replays used Qwen3.8-27B, 235200 shared context tokens,
concurrency 2, 46 GiB host KV, 160 host states, and 12 shared-prefix entries. Each
replay includes six 55K-token conversations over three passes, a 64-conversation
fleet over two passes, two XHigh turns, and two concurrent streaming requests.

| Metric | Stock repetitions | Final patched repetitions |
| --- | --- | --- |
| Maximal fallbacks | 1 / 2 | 0 / 0 |
| Cache owners evicted | 81 / 89 | 0 / 0 |
| Warm-prompt token reuse | 69.864% / 78.002% | 99.800% / 99.800% |
| 64-session revisit reuse | 0% / 43.502% | 99.432% / 99.432% |
| Host reads | 11.51 / 9.04 GiB | 17.26 / 16.73 GiB |
| Median planning time | 6.09 / 5.50 ms | 49.92 / 49.95 ms |
| Median TTFT, all requests | 0.928 / 0.823 s | 0.459 / 0.496 s |

Final artifacts: `../upgrade/results/planner-20260928-010809`; stock controls:
`../upgrade/results/planner-20260928-004054` (use only its stock rows). Earlier patched
iterations in those results were rejected. Counting maximal fallbacks alone missed
bulk destruction, so adoption also requires actual owner-eviction and revisit metrics.
The collective proposal needed replica exclusions to avoid rejected aliasing targets.

XHigh continuation and concurrent streaming completed correctly in both final runs.
Streaming duration varied materially between repetitions; these measurements do not
establish a general decode-speed improvement. Final state occupancy peaked at 154
of 160 slots; host KV peaked at 14.18 / 19.28 GiB, so this does not validate a full
46-GiB arena or guarantee retention under a larger checkpoint working set.

## Operation and rollback

`../upgrade/Start-Ninfer.ps1 -Version new` selects the custom build only when
`../upgrade/planner-deployment.json` is enabled and its executable hash matches.
The stock release and original launcher backup remain intact. To roll back after
normal service use, stop Ninfer normally, then run:

```powershell
python C:\Users\Austi\ninfer-investigation\upgrade\deploy_planner_patch.py --rollback
```

This disables the custom profile and starts stock v0.9.0. It refuses to interrupt
a production process that has received requests. Restarting discards volatile cache.

This is not persistence across restarts or an indefinite retention guarantee.
Host-state slots have no fixed TTL; their contents can survive the entire engine
uptime, until replacement, eviction, or shutdown. A slot stores a checkpoint, not
necessarily a whole conversation; one conversation can use several slots.
Both host state and KV can still fill; allocator geometry and logical catalog
limits may require eviction even when aggregate host KV bytes appear available.
