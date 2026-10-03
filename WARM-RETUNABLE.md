# Strata 0.1.37 — warm-retunable + AVX1 floor

A fork of [Strata](https://github.com/Niko1221/Strata) **v0.1.37** carrying two things upstream does not:

1. a vendored-llama.cpp **runtime retune** patch — change a running llama-server's slot count, per-slot
   context and KV pool **without reloading the weights** (this section), and
2. an **AVX1 floor** build — SSE4.2 + AVX, `STRATA_ISA_FLOOR=ON` — so the engine also runs on an AVX-only
   CPU (a Sandy Bridge Xeon), which upstream's AVX2 baseline refuses with an illegal instruction
   (see the AVX1 floor section below).

Upstream's vendored llama.cpp (`3cf0325`, the revision v0.1.37 still pins) answers `POST /props` with a stub —
`{"success": true}` and nothing else. With this patch it actually re-shapes a running server:

| request | effect |
|---|---|
| `{"parallel": N, "ctx_per_slot": T}` | re-slice the unified KV pool across N slots, weights resident |
| `{"ctx": N}`, `{"cache_type_k": "q8_0"}`, `{"flash_attn": "on"}`, `{"parallel_max": N}`, `{"kv_unified": true}` | rebuild the context in place (the weights stay loaded) |
| `{"allow_oversubscribe": true}` | let slots share a pool that cannot back them all |

Measured on an RX 7900 XTX serving a 35B-A3B MoE (Vulkan, KV q8_0):

* in-place re-slice: **0–0.4 ms**
* context rebuild (pool 32k → 262k): **~0.8 s**, weights never reloaded
* re-slicing *while other slots are decoding*: 0.35 ms
* a request that would evict a busy slot: **HTTP 400 `slot N is busy, cannot take it out of service`**
  (never a hang)
* `parallel_max` rebuild with 4 slots reserved: 814 ms

## The AVX1 floor (SSE4.2 + AVX)

Upstream cannot start on a CPU without AVX2 + FMA/F16C: the CPU expert kernels are AVX2 at least, and the
released ggml-cpu is compiled for the *build host*, so such a machine gets an illegal instruction instead of
an error message. With `STRATA_ISA_FLOOR=ON` this fork compiles ggml-cpu **once** for ggml's own
`sandybridge` feature set (SSE4.2 + AVX — no FMA, no F16C, no AVX2) and relaxes the startup gate to
"AVX2 with FMA/F16C, **or** AVX1". Measured on the machine this exists for: a Xeon E5-2687W (AVX only)
*started* fine and then died in `bf16_rows_dot_multi+0x1d9` (`vpmovzxwd`) on the **first request** — the
router lookahead calls an `-mavx2` translation unit before anything else runs, so that call now has a scalar
fallback, and the AVX2 sign table in `iq_avx2.cpp` is `constexpr` (its runtime constructor had been
vectorised into AVX-2 and ran before `main`).

```sh
STRATA_ISA_FLOOR=1 ./setup.sh --backend hip --family qwen --model IQ3_XXS --context 32768 \
    --kv int8 --gguf-dir /path/to/the/two/shards --build --no-start --yes
```

**`STRATA_ISA_FLOOR` is not a CMake `option()`** in this tree — it is only read by `if(STRATA_ISA_FLOOR)`,
so on a fresh build directory it is undefined (OFF) and you silently get a build-host-native engine. That is
why `setup.py` here passes `-DSTRATA_ISA_FLOOR=ON` explicitly when `STRATA_ISA_FLOOR=1` is set (see
`isa_floor_defs()`), and why a hand-run cmake needs the same flag. Leaving it unset breaks nothing — you
simply never get the floor.

**What it costs:** the floor is ggml-cpu-only. Strata's own AVX2/AVX-512 kernel units (`iq_avx2.cpp`,
`iq_avx512.cpp`) keep their ISA and their run-time dispatch; only the ggml-cpu tier moves down. Our own A/B
on gfx1100 — the same binary on the forced-AVX1 path versus its native AVX-512 path — was a **null result**
(trial-to-trial spread ~5×, because first-touch reads of a 55 GB pack from a rotational drive dominate), and
a tighter controlled test measured decode **21.8 → 19.1 tok/s (~12%) with byte-identical output**. So treat
"no cost" as unmeasured, not proven: the floor's reason for existing is that an AVX-only CPU otherwise
cannot run at all. Both raw arms and the caveats are in `bench/results/2026-10-01-avx1-floor/`; the curated
entry is `docs/benchmarks/2026-10-01-gfx1100-avx1-floor.json`.

**The router dot — the reason the floor is usable at all.** The lookahead that warms the file tier runs one
router dot per layer, and the floor's first version did it scalar with `std::fma` — which on a CPU with no
FMA instruction is a **libm call per element**, not an instruction. Measured on the Xeon E5-2665 this fork
exists for, one layer's router `[512 experts x 5120 embd]`:

| | ms/layer | per 48-layer lookahead pass | vs scalar `std::fma` |
|---|---|---|---|
| scalar `std::fma` (the first version) | 214.3 | 10.3 s | 1x |
| scalar mul+add | 3.33 | 160 ms | 64x |
| **`bf16_rows_dot_multi_avx1`** (this fork) | **0.81** | **39 ms** | **264x** |

For a 4- or 6-token window it is 1.91 ms / 2.63 ms per layer (450x / 489x, 92 ms / 126 ms per pass), and
the kernel agrees with the scalar reference to `max |diff| = 1.1e-4` on values ~56 (the mul+add vs FMA
rounding). Ten seconds to forty milliseconds per pass is the difference between a floor build that serves
and one that does not. The remaining AVX1 cost is the expert rows themselves: `native_gu_rows` falls
through to ggml-cpu's **single-token** `vec_dot`, so a 4-6 token verify window re-reads each weight row
4-6 times — a multi-token AVX1 kernel is the next win, and a real port (AVX1 has neither FMA nor 256-bit
integer ops).

**Pre-existing, not this fork:** the project's own GPU-free setup tests have two failures on this tree
(`tools/test_setup_amd.py` 1, `tools/test_setup_golden.py` 46) with identical counts on upstream v0.1.37
without the fork work.

## What this fork changes

1. `warm-retune/retune-llama-server-3cf0325.patch` and `warm-retune/apply.sh` — the patch itself, plus an
   idempotent `apply` / `undo` / `check` script (undo reverse-applies the patch; nothing else is needed
   because the vendored tree is regenerated by setup).
2. `setup.py` — three changes, all inert on a checkout with no vendored tree yet:
   * `get_llama_cpp()` calls `apply_warm_retune()` once the tree is present (freshly extracted or
     already on disk), so a setup run leaves the tree warm-retunable; a failure warns instead of
     aborting the setup.
   * the extraction no longer drops `tools/ui` **on non-Windows platforms**. Upstream excludes it for
     Windows' 260-character path limit (#206), but `tools/CMakeLists.txt` adds it unconditionally when
     `LLAMA_BUILD_SERVER` is on — without it, no llama-server can even be configured from this tree.
   * `isa_floor_defs()` adds `-DSTRATA_ISA_FLOOR=ON` to a local HIP/CUDA build when `STRATA_ISA_FLOOR=1`.
3. the AVX1 floor itself: the `STRATA_ISA_FLOOR` block in `CMakeLists.txt`, the relaxed startup gate plus
   the scalar router fallback (`src/program/generate.cpp`, `src/core/expert_source.cpp`), the `constexpr`
   sign table and the AVX1 expert-row path (`src/kernels/cpu/`). Measured on a Xeon E5-2687W and on
   gfx1100 — see the section above.
4. `src/kernels/cpu/kq_avx1.cpp` (+ `kq_avx1.hpp`), built with `-mavx -msse4.2`: the AVX1 router dot, 264x
   to 489x faster than the scalar fallback on the CPU this floor is for (see the table above). It is
   dispatched from `expert_source.cpp` only when the CPU has AVX1 but not AVX2, so AVX2/AVX-512 hosts keep
   the existing `-mavx2` kernel, and the scalar path that remains (no AVX at all) no longer uses
   `std::fma`.

1. `warm-retune/retune-llama-server-3cf0325.patch` and `warm-retune/apply.sh` — the patch itself, plus an
   idempotent `apply` / `undo` / `check` script (undo reverse-applies the patch; nothing else is needed
   because the vendored tree is regenerated by setup).
2. `setup.py` — two changes, both inert on a checkout with no vendored tree yet:
   * `get_llama_cpp()` calls `apply_warm_retune()` once the tree is present (freshly extracted or
     already on disk), so a setup run leaves the tree warm-retunable; a failure warns instead of
     aborting the setup.
   * the extraction no longer drops `tools/ui` **on non-Windows platforms**. Upstream excludes it for
     Windows' 260-character path limit (#206), but `tools/CMakeLists.txt` adds it unconditionally when
     `LLAMA_BUILD_SERVER` is on — without it, no llama-server can even be configured from this tree.

## Use it

```sh
./setup.sh --family coder --model IQ1_M --context 65536 --kv int8 --backend hip \
           --gguf-dir /path/to/the/two/shards --build --no-start --yes
```

`setup.py` in this fork applies the patch automatically right after it extracts
`third_party/llama.cpp` (it runs `warm-retune/apply.sh`; the step is a no-op when the patch is already
present or the tree is absent, and it never fails the setup). To do it by hand:

```sh
warm-retune/apply.sh check     # is it present?
warm-retune/apply.sh apply     # idempotent
warm-retune/apply.sh undo      # reverse-applies the patch
```

### Build a server from the vendored tree

Strata's own build compiles ggml/gguf/mtmd — **not `tools/server`** — so the patch is inert until you
build a server yourself:

```sh
cmake -G Ninja -B build-llamacpp -S third_party/llama.cpp -DCMAKE_BUILD_TYPE=Release \
      -DLLAMA_CURL=OFF -DGGML_VULKAN=ON          # or -DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1100
cmake --build build-llamacpp -j8 --target llama-server
```

**Known wart:** the vendored tree is *pruned* — `tools/ui/` is missing, and `tools/CMakeLists.txt`
adds it unconditionally when `LLAMA_BUILD_SERVER` is on, so the configure step above fails with
`add_subdirectory given source "ui" which is not an existing directory` until you restore it from the
same upstream revision:

```sh
cp -r /path/to/a/full/llama.cpp-at-the-pinned-commit/tools/ui third_party/llama.cpp/tools/
```

Then serve it with the retune endpoint enabled:

```sh
llama-server -m model.gguf -ngl 99 -fa on --kv-unified --props -np 4 -c 131072 -a my-model
curl -s localhost:8080/props | jq '{total_slots, slots_max, kv_pool_n_ctx, kv_unified}'
curl -s -X POST localhost:8080/props -d '{"parallel":2,"ctx_per_slot":65536}'
```

`--props` is what enables the write path; `--kv-unified` makes the slot count a server-side policy
instead of a load-time property.

## The patch

`warm-retune/retune-llama-server-3cf0325.patch` (1107 lines, 6 files, 626 added):

* `tools/server/server-context.cpp` — the `SERVER_TASK_TYPE_RECONFIGURE` task, the in-place slot
  policy, the context-rebuild path, the busy-slot refusal and the lazy KV release
* `tools/server/server-context.{h,server-task.h}` — the runtime-state struct and task/result types
* `common/common.{cpp,h}` — `reload_context()` plus a threadpool reset/init so a rebuild reuses the
  process instead of restarting it
* `tools/server/README.md` — `POST /props` documented

It targets llama.cpp `3cf0325` (the revision v0.1.37 pins — unchanged since v0.1.31) and was forward-ported from a tree built
against `7fe450e19` by `git apply --3way`; the added/removed content is **identical** to the original
(verified by diffing the two patches' content lines and by an independent audit).

## Licence

Upstream Strata is MIT (see `LICENSE`); llama.cpp is MIT. The patch is a derivative of llama.cpp and
carries the same terms.
