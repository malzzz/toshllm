# Telemetry branch

Remotes: `upstream` = `engeldlgado/toshllm` (canonical), `origin` =
`malzzz/toshllm` (our fork). This fork is the canonical place for two kinds
of work: the telemetry patch series (this branch) and fixes to current
ToshLLM meant for upstream PRs (see "Upstream fixes" below).

This branch (`telemetry`, tracks `upstream/main`) carries the MXP telemetry
instrumentation plus our not-yet-upstream fixes for the vendored llama.cpp
engine.

## Series layout (v0.87.1)

v0.86 collapsed upstream's ~65 area-dir patches into **7 monolithic patches
in a flat layout** (`patches/llama/0001-metal-kernels.patch` …
`0007-tools-and-build.patch`; the old `core/metal/model/server/
experimental-moe` dirs are gone). v0.87.0 runs the flat series 0001-0048
(monoliths + our folded 0008/0009 + the v0.86.2-0.86.6 mgpu/MoE work +
0041-0048 GCN-native kernels) and moved the vendored llama.cpp pin to
`465e49b9cea78a68b9c244ffb48d0ee24a82873d`. Metal kernels live in
`ggml/src/ggml-metal/kernels/*.metal`; his 64-lane decode/expert/fused
matvecs are in `kernels/mul_mv_w64.metal`.

The v0.87.1 sync merges upstream `6c18507` and keeps that llama.cpp pin.
Upstream's series now runs through 0057; gaps at 0049, 0052 and 0053 are
normal. Our nine surviving patches follow it as 0058-0066, preserving their
relative order. Upstream owns its original patch names and numbers.

v0.86.1 folded our upstreamed fixes into the flat series (PRs #77-80, all
closed/merged):

- his `0002-metal-backend-host.patch` now carries our mmap gate
  (`buffer_from_host_ptr = has_unified_memory || use_shared_buffers`) and our
  metal MoE-OFF stubs — our 0009/0010 dropped.
- his `0005-llama-core.patch` now carries the `tosh_moe_fixed_of` stub —
  our 0008 dropped.
- his `0008-metal-comm-allreduce-fail-loud.patch` +
  `0009-metal-sync-flush-pending-waits.patch` are our PR #80, merged as-sent
  — our 0011/0012 dropped.

v0.87.1 absorbed four of our patches byte-for-byte:

- `0050-metal-ext-mv-wave64-tuning.patch` and
  `0051-metal-ext-mv-wave64-single-pass.patch` — PR #94, wave64 Q6_K ext
  matvecs for the multi-token path, including the eight-row single pass.
  The v0.87.0 measurement was DFlash2 single-die 12.5 t/s versus 9.7 t/s
  with `TOSH_EXT_W64_DISABLE=1` (+29%).
- `0054-metal-moe-off-active-stub.patch` — PR #95, the missing
  `tosh_moe_off_active` stub for `TOSH_ENABLE_DYNAMIC_MOE=OFF` builds.
- `0055-metal-fa-dk384-640-guard.patch` — PR #96, restrict dk384/640 to
  instantiated Flash Attention kernels and null-guard missing pipelines.

Keep those upstream copies; they are no longer part of our custom tail.
Upstream also fixes padded sparse-attention reads in 0048, adds pre-Vega
aligned matvec loads in 0056, and mirrors the output head for DFlash tensor
splits in 0057. None supersedes the remaining local changes below.

What remains ours, at the series tail (old numbers refer to v0.87.0):

- `0058-metal-queue-depth-scale.patch` (was 0049) — scale MTL queue depth with the
  registered device count. **FORK-LOCAL**: the maintainer's bar was a
  measured reproducible case of the 64-buffer limit backpressuring; we ran
  the measurement and it is a **clean negative** (recorded in
  model-experiments `vega-duo-diag/queue-depth-backpressure.md`). The patch
  stays as harmless insurance for the 4-die rig — do not PR it.
- `0059-metal-telemetry.patch` (was 0052) — ggml-metal telemetry glue.
- `0060-cuda-telemetry.patch` (was 0053) — ggml-cuda telemetry glue.
- `0061-qwen4exp-mtp.patch` (was 0056) — MTP/NextN draft-head support for qwen4exp
  (Qwen3.8-Flash-Next): five `blk.N.nextn.*` tensor types (fc_embedding,
  fc_hidden, hc_mix_norm/down/up), mtp_only/load_mtp wiring in
  `qwen4exp.cpp::load_arch_tensors`, conditional last-layer row drop +
  wide pre-final-mixer `t_h_nextn` on the target graph (the staging
  plumbing was already n_embd_out-wide), dense `graph_mtp` (no QSA indexer
  — the GLM_DSA/DEEPSEEK32 draft precedent), `hc_combine` null-inject
  unit-weight path, QWEN4EXP added to the `mtp_on_hybrid` memory branch
  (draft ctx = plain KV cache over the nextn layer only), converter MTP
  export + gguf-py tensor map entries. Semantics pinned from vLLM
  `qwen4_exp/nvidia/mtp.py` (only public MTP implementation; HF transformers
  ignores ^mtp.*). Measured on the rig with a hand-built mtp-only draft
  GGUF (MXP `scripts/make-qwen4exp-mtp-gguf.py`): engages, 54.2% acceptance,
  correct output. Upstream PR candidate (large; coordinate with maintainer).
- `0062-linux-meta-threads.patch` (was 0057) — ggml-backend-meta: std::thread twin of
  the Apple GCD multi-backend subgraph dispatch. Captures a long-standing
  in-tree edit from the Linux bring-up that was never in the series (the
  v0.87 round-trip diff caught it). FORK-LOCAL: Linux-only, upstream is
  macOS+AMD.
- `0063-metal-xdev-exchange-channel.patch` (was 0058) — ggml-metal-context.m: the fused
  butterfly exchange (`ggml_metal_exchange_reduce`) shared the `xdev_link`
  seq counter, ready/done events, and host wrap buffers with the unfused
  cpy_xdev_events/peer copy paths. Runs mixing both schemes on one link
  (4-die tensor split: unfused at prefill sizes, fused at decode sizes)
  desynchronize the event waits and silently corrupt the collective. The
  exchange gets its own events/counter/wrap buffers. Repro: ANY arch corrupt
  under 4-die `-sm tensor` (dense 1B first-token junk, qwen35moe zeros,
  qwen4exp babble); post-fix 3/3 exact matches incl. qwen4exp == layer-split
  control, and 2-die tensor 105.3 -> 116.9 t/s (the collision was stalling
  it too). Note: `-sm tensor` is the ToshLLM app default, so this bit every
  multi-die AMD user. Upstream PR candidate (strong).
- `0064-spec-timing-instrumentation.patch` (was 0059) — TOSH_SPEC_TIMING phase timers in
  the speculative-simple example (draft/ckpt/verify/process/accept per-round
  breakdown). FORK-LOCAL: profiling tool, not for upstream.
- `0065-metal-gdn-wave64.patch` (was 0060) — gated_delta_net float4 state IO under
  NSG==4 (2.83x decode / 4.13x verify at qwen4exp shapes, 700 GB/s ~= the
  practical HBM ceiling), plus new kernel_ssm_scan_f32_dec wave64 decode
  variant (2.78x, mamba-family; fires only at n_seq_tokens==1 &&
  simd_width==64 && d_state==2*simd_width). Note: the in-model GDN op is
  gated_delta_net, not ssm_scan/conv — verified via dispatch telemetry.
  E2E: 22.9 -> 23.4 t/s baseline, MTP verify wall 59.2 -> 50.0 ms.
  Correctness: full GATED_DELTA_NET + new K=2 cases pass, E2E byte-identical.
  Upstream PR candidate.
- `0066-metal-fa-wg256-gate.patch` (was 0061) — ggml-metal-ops.cpp: suppress the wg32
  KV-split + separate reduce pass at simd_width==64 && dk==256 && heads>=8
  (3x slower than the plain walk at the qwen4exp QSA shape in harness;
  e2e-neutral). Adds TOSH_FA_WG_NB_MAX test knob. Also records: the
  TOSH_FA_AMD_NSG_WIDE env knobs are invalid-work generators (dispatch nsg
  must match the template-baked NSG). Fork-local hygiene; revisit for
  upstream only with in-model evidence.

Dropped across the v0.86/v0.86.1/v0.87.0 syncs:

- `0087/0088-dflash2-*` — the ca3d5a3e1 pin has DFlash2/DSpark natively.
- `0084-metal-peer-gate-pair-only` (peerCount==2 gate) — **REJECTED
  upstream**: the W6800X Duo reports peerCount=4 across two modules joined
  by the A2667 bridge, which IS supported fabric — peer measured +74%
  prefill there (that's what closed upstream issue #51 and made mgpuPeer
  default-on). The gate is a no-op on our 2x Vega II Duo rig (peerCount=2
  per module is correct there) but bites exactly where the bridge is
  supported. Do not resurrect it.

Patch apply order is the numeric prefix of the basename (see
`patch_series()` in `scripts/build-engines.sh`). With
`GGML_METAL_TELEMETRY` / `GGML_CUDA_TELEMETRY` OFF the telemetry patches
compile to nothing; with `TOSH_TELEMETRY` unset at runtime they are inert.

## Patch format

- **`-U8` context** (upstream's rule for the flat series, documented in
  `patches/README.md` since v0.86.1).
- **Round-trip verify** every regeneration: apply to the pre-change copy,
  confirm byte-identical.

## Sources of truth

The Linux checkout is `/data/alchemical-rabbit/libs/toshllm`. On the Mac
Pro, use `~/toshllm-exp/toshllm`; its `vendor/llama.cpp` symlink points to
`../../vendor-86`, preserving the existing experiment and build paths.
Both checkouts carry the same v0.87.1 custom series. The Mac-only
`test-backend-ops.cpp` diagnostic overlay stays local and is excluded from
the canonical patch series. Preserve it separately when replacing or
reconstructing the Mac vendor tree.

The telemetry glue sources and patchers live in the MXP repo at
`/data/alchemical-rabbit/model-experiments`:

- glue: `mxp-telemetry/ggml-metal-telemetry.{h,m}` and
  `mxp-telemetry/ggml-cuda-telemetry.{h,cpp}`
- patchers: `scripts/apply-telemetry-edits.py` (metal),
  `scripts/apply-telemetry-edits-cuda.py` (cuda)
- regenerators: `scripts/regen-metal-telemetry.sh` (metal, default number
  0059), `scripts/regen-cuda-telemetry.sh` (cuda, default number 0060)

Never hand-edit the telemetry patch files here; change the glue or patcher
in MXP and regenerate. (The non-telemetry patches 0058 and 0061-0066 are hand-carried
— re-diff them per sync, below.)

## Regenerating the telemetry patches

Prereqs: vendor tree at the pin (`LLAMA_COMMIT` in
`scripts/build-engines.sh`, currently
`465e49b9cea78a68b9c244ffb48d0ee24a82873d`):

```sh
git clone --filter=blob:none https://github.com/ggml-org/llama.cpp vendor/llama.cpp
git -C vendor/llama.cpp checkout "$LLAMA_COMMIT"
```

The regen scripts use version-scoped throwaway clones
`/tmp/llama-tele-v0.87.1` and `/tmp/llama-tele-cuda-v0.87.1` (bootstrapped on
first run), regenerate into the flat `patches/llama/`, and re-apply to the
vendor tree. Set `TC` to a fresh path whenever the pin or preceding patches
change; an existing cached base is not rebuilt automatically.

The vendor tree must be at the corresponding series position when each
script runs, with no local edits in the files it restores. Running Metal
regen on a fully patched vendor tree would discard later changes in those
files, including the exchange-channel fix. Metal telemetry 0059 diffs
against the pin plus every patch below 0059; CUDA telemetry 0060 diffs
against the pristine pin because no other patch touches ggml-cuda.
Sequence on Linux, starting with a clean disposable vendor checkout at
the pin:

```sh
MXP=/data/alchemical-rabbit/model-experiments
export FORK=/path/to/disposable/toshllm
cd "$FORK/vendor/llama.cpp"

# 1. upstream's patches through 0057, then our queue-depth patch 0058
find "$FORK/patches/llama" -maxdepth 1 -name '*.patch' -printf '%f\t%p\n' | sort | cut -f2- \
  | awk -F/ '$NF < "0059"' | while read -r p; do git apply "$p"; done

# 2. metal telemetry (regenerates + applies 0059)
$MXP/scripts/regen-metal-telemetry.sh

# 3. cuda telemetry (regenerates + applies 0060)
$MXP/scripts/regen-cuda-telemetry.sh

# 4. the remaining custom patches 0061-0066
find "$FORK/patches/llama" -maxdepth 1 -name '*.patch' -printf '%f\t%p\n' | sort | cut -f2- \
  | awk -F/ '$NF >= "0061"' | while read -r p; do git apply "$p"; done
```

## Re-diffing the hand-carried patches (0058, 0061-0066) on a sync

When upstream moves the pin or the monoliths, our non-telemetry patches must
be re-created against the monolith-applied tree. The procedure used for the
v0.86 sync (works for any sync):

1. Save the old patch files as semantic specs (`git show
   <pre-sync-telemetry>:patches/llama/...`).
2. Fresh scratch clone of the vendor repo at the new pin; apply every upstream
   patch in order; commit as "base".
3. For each surviving fix, in series order: `git apply` the old patch (most
   hunks land with offsets); where the layout moved (e.g. the 0089 kernel:
   `ggml-metal.metal` → `kernels/mul_mv.metal`, instantiations after the
   `kernel_mul_mv_ext_q6_K_f32_r1_5_nr0_2` template block) or context
   drifted, hand-port the identical semantic change. Commit each fix
   separately. Regenerate and apply telemetry at its numeric position
   before continuing with the later hand-carried patches.
4. `git diff -U8 <prev> <cur>` per commit → the new patch files.
5. Validate: fresh scratch at the pin, apply ALL patches (monoliths + ours)
   in numeric order with `git apply --check`, then `diff -rq` against the
   real vendor tree — must be byte-identical.
6. If the patcher anchors drifted, fix them in the MXP patchers
   (`apply-telemetry-edits*.py`), never in the generated patches; verify by
   running the patcher against the pre-telemetry stage and diffing against
   the committed telemetry stage.

## Validating the series

From a clean disposable vendor checkout at the pin, every patch must apply in global
numeric order (this is what `build-engines.sh` does):

```sh
FORK=/path/to/disposable/toshllm
cd "$FORK/vendor/llama.cpp"
find "$FORK/patches/llama" -maxdepth 1 -name '*.patch' -printf '%f\t%p\n' | sort | cut -f2- \
  | while read -r p; do git apply --check "$p" && git apply "$p"; done
```

## Syncing with upstream

```sh
git fetch upstream
git checkout telemetry
git merge upstream/main
```

Preserve existing edits before merging, and use an isolated worktree for
patch reconciliation. Upstream may delete, restructure, or absorb patches
we also touched. Accept upstream's layout and regenerate the surviving
custom series with the re-diff procedure above. The v0.87.1 sync preserves
both histories with a merge rather than rewriting the telemetry branch.

After the merge:

0. Number collisions / superseded patches: if upstream took a number or
   absorbed a fix (like 0087/0088 at v0.86, or 0008-0012 at v0.86.1 when his
   PR #79 fold and the PR #80 patches landed), drop ours and renumber the
   tail. Always renumber ours, never upstream's.
1. Prepare a clean disposable vendor tree at the new pin. Preserve local
   vendor edits and the Mac diagnostic overlay before replacing the live
   vendor contents with the validated result.
2. Give each regen script a fresh version-scoped `TC` path so its base is
   rebuilt against the new series; preserve the previous scratch clones.
3. Re-diff 0058/0061-0066 (procedure above), re-run both telemetry regen
   scripts, re-validate the full series.
4. Commit the regenerated patches on this branch.

## Upstream fixes (PR workflow)

Bug fixes intended for upstream live on their own branches off
`upstream/main` — never on `telemetry` (it carries our patch series, which
upstream does not want):

```sh
git fetch upstream
git checkout -b fix/<short-name> upstream/main
# ... minimal fix, commit message style per upstream log ...
git push -u origin fix/<short-name>
gh pr create --repo engeldlgado/toshllm --base main --head malzzz:fix/<short-name>
```

Layout for PR-bound patches: flat `patches/llama/NNNN-name.patch` at `-U8`.

Precedent: `fix-qvk-matcher-order` on origin (docs in MXP `pr-qvk-matcher/`),
PRs #77-80 (merged at v0.86.1 — outcomes recorded in the series list above).
If a fix touches a file the telemetry patches also touch, re-run the series
validation above after it lands upstream.

Scope rule: upstream ToshLLM targets macOS + AMD only. PRs are for fixes
that matter there. Linux/CUDA-only fixes ride in our own series instead —
e.g. the former `0008-moe-fixed-of-stub.patch` (merged upstream at v0.86.1):
upstream's MoE cache code called `tosh_moe_fixed_of()` in `llama-graph.cpp`
without adding it to the non-`TOSH_ENABLE_DYNAMIC_MOE` stub block, which only
broke builds with MoE disabled (i.e. non-macOS).

Note: the vendored llama.cpp tree carries ggml-org's AGENTS.md, which bans
automated PR submissions and agent-written PR text for ggml-org/llama.cpp
(private forks are exempt — ours is one). If a change ever targets
ggml-org upstream, the human submits it.

## Keeping the fork's main current

`origin/main` should track `upstream/main` (fast-forward only — never commit
on main):

```sh
git fetch upstream
git push origin upstream/main:main
```
