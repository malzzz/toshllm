# Experimental v0.87.3 Mac application patch set

The user explicitly requested an installed v0.87.3 application that includes the
retained experimental candidates, to observe their behavior on the Vega II Duos.
The five patches below now follow the preserved custom tail through 0068 and are
included by the build script's ordinary numeric patch discovery. This is the
default series for that requested experimental build, not a claim of production
mathematical correctness, measured combined speedup, or upstream acceptance.

The baseline is telemetry merge `ae66a14972a82818f71cf329b84c54897880c384`
(upstream v0.87.3), with llama.cpp pin
`465e49b9cea78a68b9c244ffb48d0ee24a82873d`. Existing patches through 0068 are
unchanged. Preserve a recoverable installed-app backup before deployment.

## Exact appended manifest

Original source deltas are in the sibling `model-experiments` repository under
`vega-duo-diag/fixes/`. Each packaged patch names that retained source in its
header. Patch bodies are unchanged except that 0070's historical
`a/source/` / `b/guard-source/` filenames are normalized to standard `a/` / `b/`.
All five use ordinary `git apply` (`-p1`).

| Patch | Retained source delta | Purpose |
| --- | --- | --- |
| [0069-metal-xdev-preflight-cache.patch](patches/llama/0069-metal-xdev-preflight-cache.patch) | `2026-09-11-0063/fix-delta-cached.patch` | Prepare all fused exchange resources before execution; fail-stop unsafe late failures; cache the successful ADD pipeline |
| [0070-metal-gdn-guarded-rows2.patch](patches/llama/0070-metal-gdn-guarded-rows2.patch) | `2026-09-11-0065-0061/guard-delta.patch` | Share q/k loads across two GDN state rows on the measured Vega/shape subset |
| [0071-metal-gdn-k-reuse.patch](patches/llama/0071-metal-gdn-k-reuse.patch) | `2026-09-11-0065-0061/0065-k-reuse.delta.patch` | Hoist the repeated k-vector load in the original GDN kernel |
| [0072-mtp-request-cap-accounting.patch](patches/llama/0072-mtp-request-cap-accounting.patch) | `2026-09-11-0065-0061/0061-shared-mtp-cap.delta.patch` | Respect per-request draft limits and count offered MTP tokens |
| [0073-qwen4exp-gather-before-hc.patch](patches/llama/0073-qwen4exp-gather-before-hc.patch) | `2026-09-11-0065-0061/gather-delta.patch` | Gather target logits rows before the final HC mixer while retaining the complete NextN stream |

Packaged SHA256 hashes:

```text
b02a1799e341858cee458eb919bd68846c8b8801bef7a87234b2c1ea12e26354  0069-metal-xdev-preflight-cache.patch
eb51254d2c7eb6d51bc0b1f81f0652523866772b67f3423cb2b8b98467fa5add  0070-metal-gdn-guarded-rows2.patch
b34066aba9482b9964db190298b92dd126768e9c69df211db68f6a593f81fad8  0071-metal-gdn-k-reuse.patch
8cd450bf2f4e53f27f20db97efa3c51dff5fe437a4e5963750271a4c7839f7b2  0072-mtp-request-cap-accounting.patch
540291d02294c6515e117f424ba17e390a9380673a1b6ca5aacf1061ff2b3394  0073-qwen4exp-gather-before-hc.patch
```

## Known limits, retained rather than waived

- **0069:** the cached candidate passed its bounded performance noninferiority
  gate and 129 native checks, but intermittent GPU timeouts during injected
  failure recovery remain unexplained. The uncached version also reproduced
  that timeout; the cache is not established as its cause.
- **0070:** targeted CPU-reference correctness passed. The guarded H12 case
  gained roughly 7% at kernel level, but only 70/72 matrix performance gates
  passed and the resident-model comparison found no resolved whole-model gain.
  This is the guarded variant, not the broader row-pairing screen that regressed
  the H48/K2 shape.
- **0071:** targeted correctness passed; the approximately 2% multi-token
  performance observation was an initial screen. Combining it with 0070 has
  not previously been qualified. No additional SSM optimization is added here.
- **0072:** 21 host cases and short live comparisons passed. Broader MTP
  state/parity and performance qualification remain open; a baseline-only
  control reproduced the longer-request divergence.
- **0073:** NextN rows and selected tokens matched, but the retained 32-token
  prefill comparison changed 248,246/248,320 logits, maximum absolute difference
  0.00131905. Its exact-parity gate failed; this experimental inclusion does not
  relabel that result or establish lossless MTP behavior.

The five-way combination has not had a native correctness/performance campaign.
Do not add the uncached 0063 delta or its cache-only follow-on alongside 0069.
Do not add historical full replacement patches at the new tail. Fault injection,
benchmark switches/counters, live diagnostic copies, the broad fallback-sync
experiment and extra error-observer overlays are excluded. Existing frozen
Stage 0 controls and reports remain separate and unchanged.

## Packaging verification and integration boundary

All five patches were checked and applied sequentially to a fresh temporary
copy of the nine affected files from the current canonical vendor. All passed;
no canonical vendor file was changed by packaging. This is a combined patch
application check, not a full-series rebuild, native test, or app installation.

The ordinary fork patches and the separate frozen Stage 0 v8 telemetry overlay
are not interchangeable bases. A read-only check against the local frozen v8b
source found 0069 context conflicts in `ggml-metal-context.m` and
`ggml-metal-device.m`; 0070-0073 individually apply there. An app build retaining
v8 telemetry needs a separately recorded reconciliation of those existing hook
contexts. Preserve the hooks and candidate behavior; do not use rejected hunks
or overwrite the telemetry files with ordinary-fork copies. No new recorder,
rounding rule or instrumentation framework is introduced by this patch set.

Rebuild source-matched embedded and precompiled shaders for the selected final
tree. The prior 25-library diagnostic bundle lacks the rows2 entrypoint and must
not be reused as if it matched these experimental kernels. The installed app
and runtime activation options are handled separately by the designated operator.
