# Telemetry branch

Remotes: `upstream` = `engeldlgado/toshllm` (canonical), `origin` =
`malzzz/toshllm` (our fork). This fork is the canonical place for two kinds
of work: the telemetry patch series (this branch) and fixes to current
ToshLLM meant for upstream PRs (see "Upstream fixes" below).

This branch (`telemetry`, tracks `upstream/main`) carries the MXP telemetry
instrumentation plus our not-yet-upstream fixes for the vendored llama.cpp
engine.

## Series layout (v0.87.0)

v0.86 collapsed upstream's ~65 area-dir patches into **7 monolithic patches
in a flat layout** (`patches/llama/0001-metal-kernels.patch` …
`0007-tools-and-build.patch`; the old `core/metal/model/server/
experimental-moe` dirs are gone). v0.87.0 runs the flat series 0001-0048
(monoliths + our folded 0008/0009 + the v0.86.2-0.86.6 mgpu/MoE work +
0041-0048 GCN-native kernels) and moved the vendored llama.cpp pin to
`465e49b9cea78a68b9c244ffb48d0ee24a82873d`. Metal kernels live in
`ggml/src/ggml-metal/kernels/*.metal`; his 64-lane decode/expert/fused
matvecs are in `kernels/mul_mv_w64.metal`.

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

What remains ours, at the series tail (0050/0051 measured on v0.87.0, see
below):

- `0049-metal-queue-depth-scale.patch` — scale MTL queue depth with the
  registered device count. **FORK-LOCAL**: the maintainer's bar was a
  measured reproducible case of the 64-buffer limit backpressuring; we ran
  the measurement and it is a **clean negative** (recorded in
  model-experiments `vega-duo-diag/queue-depth-backpressure.md`). The patch
  stays as harmless insurance for the 4-die rig — do not PR it.
- `0050-metal-ext-mv-wave64-tuning.patch` — wave64 q6_K ext matvec kernel
  (`kernel_mul_mv_ext_q6_K_f32_w64_disp` in `kernels/mul_mv.metal`) + pipeline
  picker and dispatch gate. His v0.87 64-lane work covers `mul_mv`,
  `mul_mv_id` (experts) and `qvk` (fused) but **not** the multi-token ext
  path (`ne11` 2-12), which these kernels fix — verified in his 0032-0048
  (zero `mul_mv_ext` hits) and measured additive on v0.87.0: DFlash2
  single-die 12.5 t/s with vs 9.7 t/s with `TOSH_EXT_W64_DISABLE=1` (+29%).
  Pending upstream submission.
- `0051-metal-ext-mv-wave64-single-pass.patch` — eight src1 rows in one
  weight pass (`kernel_mul_mv_ext_q6_K_f32_w64x8`), active from ne11 6-12.
- `0052-metal-telemetry.patch` — ggml-metal telemetry glue.
- `0053-cuda-telemetry.patch` — ggml-cuda telemetry glue.
- `0054-metal-moe-off-active-stub.patch` — `tosh_moe_off_active` was added to
  the MoE compact-assemble path in v0.86.3+ without an OFF stub; breaks
  `TOSH_ENABLE_DYNAMIC_MOE=OFF` builds. Same class as the old 0078/0081 —
  upstream PR candidate.
- `0055-metal-fa-dk384-640-guard.patch` — `supports_op` and the FA dispatch
  name list admitted dk 384/640 for non-Turbo K/V, but the .metal side
  instantiates the standard pairs only at dk 64/128/256/512 (384/640 are
  Turbo-only): the pipeline miss left a null pipeline that the max-threads
  guard dereferenced — SIGSEGV on AMD. Gates 384/640 to Turbo KV in the AMD
  branch, rejects dk 384/640 in the generic tail too (the 0002 head-size
  widening had also opened the generic Apple-Silicon path, which likewise
  lacks the instantiation — upstream never admits 384/640), and null-guards
  the vec/tile pipeline lookups (miss falls through to the generic kernels).
  Validated on gfx906: pre-fix exit 139 repro, post-fix the configs skip to
  CPU; full FA suite otherwise unchanged. Upstream PR candidate.
- `0056-qwen4exp-mtp.patch` — MTP/NextN draft-head support for qwen4exp
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
- `0057-linux-meta-threads.patch` — ggml-backend-meta: std::thread twin of
  the Apple GCD multi-backend subgraph dispatch. Captures a long-standing
  in-tree edit from the Linux bring-up that was never in the series (the
  v0.87 round-trip diff caught it). FORK-LOCAL: Linux-only, upstream is
  macOS+AMD.
- `0058-metal-xdev-exchange-channel.patch` — ggml-metal-context.m: the fused
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

Dropped across the v0.86/v0.86.1/v0.87 syncs:

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

The telemetry glue sources and patchers live in the MXP repo at
`/data/alchemical-rabbit/model-experiments`:

- glue: `mxp-telemetry/ggml-metal-telemetry.{h,m}` and
  `mxp-telemetry/ggml-cuda-telemetry.{h,cpp}`
- patchers: `scripts/apply-telemetry-edits.py` (metal),
  `scripts/apply-telemetry-edits-cuda.py` (cuda)
- regenerators: `scripts/regen-metal-telemetry.sh` (metal, default number
  0052), `scripts/regen-cuda-telemetry.sh` (cuda, default number 0053)

Never hand-edit the telemetry patch files here; change the glue or patcher
in MXP and regenerate. (The non-telemetry patches 0049-0051 and 0054 are hand-carried
— re-diff them per sync, below.)

## Regenerating the telemetry patches

Prereqs: vendor tree at the pin (`LLAMA_COMMIT` in
`scripts/build-engines.sh`, currently
`465e49b9cea78a68b9c244ffb48d0ee24a82873d`):

```sh
git clone --filter=blob:none https://github.com/ggml-org/llama.cpp vendor/llama.cpp
git -C vendor/llama.cpp checkout "$LLAMA_COMMIT"
```

Each regen script refreshes its throwaway clone under /tmp (bootstrapped on
first run), regenerates the patch into the flat `patches/llama/`, and
re-applies it to the vendor tree. The scripts restore only their own files
before applying, so the vendor tree must be at the patch's series position
when they run. The metal telemetry patch (0052) diffs against pin + 0001-0051;
the cuda one (0053) against the pristine pin (no other patch touches
ggml-cuda). Sequence:

```sh
MXP=/data/alchemical-rabbit/model-experiments
cd vendor/llama.cpp

# 1. everything except the telemetry patches (upstream 0001-0048 + our 0049-0051)
find ../../patches/llama -maxdepth 1 -name '*.patch' -printf '%f\t%p\n' | sort | cut -f2- \
  | awk -F/ '$NF !~ /telemetry/' | while read -r p; do git apply "$p"; done

# 2. metal telemetry (regenerates + applies 0052)
$MXP/scripts/regen-metal-telemetry.sh

# 3. cuda telemetry (regenerates + applies 0053)
$MXP/scripts/regen-cuda-telemetry.sh
```

## Re-diffing the hand-carried patches (0049-0051, 0054) on a sync

When upstream moves the pin or the monoliths, our non-telemetry patches must
be re-created against the monolith-applied tree. The procedure used for the
v0.86 sync (works for any sync):

1. Save the old patch files as semantic specs (`git show
   <pre-sync-telemetry>:patches/llama/...`).
2. Fresh scratch clone of the vendor repo at the new pin; apply the upstream
   monoliths in order; commit as "base".
3. For each surviving fix, in series order: `git apply` the old patch (most
   hunks land with offsets); where the layout moved (e.g. the 0089 kernel:
   `ggml-metal.metal` → `kernels/mul_mv.metal`, instantiations after the
   `kernel_mul_mv_ext_q6_K_f32_r1_5_nr0_2` template block) or context
   drifted, hand-port the identical semantic change. Commit each fix
   separately.
4. `git diff -U8 <prev> <cur>` per commit → the new patch files.
5. Validate: fresh scratch at the pin, apply ALL patches (monoliths + ours)
   in numeric order with `git apply --check`, then `diff -rq` against the
   real vendor tree — must be byte-identical.
6. If the patcher anchors drifted, fix them in the MXP patchers
   (`apply-telemetry-edits*.py`), never in the generated patches; verify by
   running the patcher against the pre-telemetry stage and diffing against
   the committed telemetry stage.

## Validating the series

From a clean vendor checkout at the pin, every patch must apply in global
numeric order (this is what `build-engines.sh` does):

```sh
cd vendor/llama.cpp
git checkout -- . && git clean -fdq
find ../../patches/llama -maxdepth 1 -name '*.patch' -printf '%f\t%p\n' | sort | cut -f2- \
  | while read -r p; do git apply --check "$p" && git apply "$p"; done
```

## Syncing with upstream

```sh
git fetch upstream
git checkout telemetry
git rebase upstream/main   # or: git merge upstream/main
```

Expected conflict shape (v0.86): upstream deletes/restructures patch files
we also touched. Resolution: accept upstream's layout, drop our old-layout
files from every rebased commit, then re-create our series on top with the
re-diff procedure above. (For v0.86 the cleanest end state was a hard reset
to `upstream/main` plus one fresh sync commit — the rebase artifacts were
empty commits and rename-detection noise.)

After the rebase:

0. Number collisions / superseded patches: if upstream took a number or
   absorbed a fix (like 0087/0088 at v0.86, or 0008-0012 at v0.86.1 when his
   PR #79 fold and the PR #80 patches landed), drop ours and renumber the
   tail. Always renumber ours, never upstream's.
1. Re-checkout the vendor tree at the new pin (`git checkout -- . &&
   git clean -fdq` is fine there once `git status` shows nothing outside
   the patch series).
2. Delete the throwaway clones (`/tmp/llama-tele`, `/tmp/llama-tele-cuda`)
   so they re-bootstrap against the new series.
3. Re-diff 0049-0051/0054 (procedure above), re-run both telemetry regen
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
