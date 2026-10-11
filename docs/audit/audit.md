# Hypertrain audit: gap analysis and improvement plan

Repository: `CortexLM/hypertrain` at commit `8baf7c8` (branch `boat/224a523e5f92`). Audit date: 2026-10-11.
Scope: `docs/`, `src/`, `tests/`, `experiments/`, `scripts/`, `.github/`, `deploy/`, `docker/`.
Every repository citation is `path:line` or `path:start-end`, relative to the repository root. `scripts/validate_audit_refs.py` (in the job directory, not the repo) checks each one against the tree.

Lineage tags: **MEASURED** means a number recorded in the repository or produced by a script in this audit. **DERIVED** means computed from measured inputs; the formula is shown. **ASSUMED** means an input chosen by the auditor, with the reason given.

---

## 0. Executive summary

Hypertrain is a carefully engineered **verification and settlement layer**. It has signed envelopes, drand-gated sampling, bitwise replay, N-ary bisection, integer escrow and hermetic tests. The **training engine** under it is still a correctness prototype. The headline gaps:

1. **MFU is about 0.1-0.3% on the only production-shaped GPU run** (DERIVED, section 4). The decoder uses a hand-written fp32-score attention (`src/hypertrain/trainer/model.py:174-176`). TF32 and reduced-precision reductions are disabled globally (`src/hypertrain/trainer/determinism.py:51-58`). Gradients are synchronised with a per-tensor `all_gather` and a serial sum (`src/hypertrain/trainer/island.py:524-543`). The optimizer loops over tensors in Python (`src/hypertrain/trainer/optim.py:138-153`). Phase B itself recorded that deterministic mode runs at about half the baseline throughput (`experiments/results/phase_b_summary.json:24-43`).
2. **Update compression is lossy-only and has no entropy stage.** The payload is raw packed fields (`src/hypertrain/trainer/compress.py:119-124`, `:170-177`). A scratch run in this audit shows that the same SparseLoCo payload shrinks **4.49x more, losslessly**, with gap-coded indices plus byte shuffling plus zstd (section 5).
3. **The pipeline cannot carry real model sizes yet.**
   - Delta validation is hard-capped at 64 KiB on two service paths (`src/hypertrain/trainer/compress.py:265`, `src/hypertrain/challenge/store.py:7454`, `:7463`, `src/hypertrain/aggregator/capacity_worker.py:436`).
   - The aggregator norm math converts every coordinate to a Python float (`src/hypertrain/aggregator/core.py:104-117`).
   - Relay object reads must finish in 90 s (`src/hypertrain/data/stream_store.py:48`).
   - Rosters are capped at 16/32 (`src/hypertrain/challenge/store.py:4384-4393`).
4. **Only two model families exist**: a causal decoder (dense or MoE), and the OpenDecision encoder via an external package (`src/hypertrain/trainer/config.py:35`, `src/hypertrain/models/opendecision/__init__.py:52`). The production shard format is u16 (`src/hypertrain/datasets/build.py:103-104`). There is no tensor or pipeline execution parallelism, no document masking, no multimodal sample type and no SFT/RL loss.
5. **Trust is still centralised in places.**
   - A single auditor's MATCH mints reward (no quorum).
   - A miner's own rerun can classify a mismatch as TRANSIENT (`src/hypertrain/auditor/replay.py:857-858`).
   - There is one referee and one coordinator key.
   - The public API cache grows without bound (`src/hypertrain/public_api/__init__.py:49`, `:66`).
6. **The quality claims are unproven.** All 150 parity jobs are CENSORED (`docs/operator.md:153`). Docs contain arithmetic and factual inconsistencies (section 2.8).

Quality gates on this tree (MEASURED, logs in `logs/`): `ruff format --check` exit 0, `ruff check` exit 0, `mypy` exit 0 (113 files). The pytest outcome is recorded in `build.json`.

---

## 1. Everything missing

### 1.1 Training engine
| Gap | Evidence |
|---|---|
| No fused/flash attention in the decoder; scores materialised in fp32 `[B,H,T,T]`, mask rebuilt every layer | `src/hypertrain/trainer/model.py:174-176` |
| GQA implemented by physically copying K/V | `src/hypertrain/trainer/model.py:171-173` |
| No tensor parallelism, no executed pipeline parallelism (`pp` only partitions leaf hashes) | `src/hypertrain/trainer/loop.py:90-102`, `src/hypertrain/trainer/island.py:3-8` |
| No `torch.compile`, CUDA graphs, activation checkpointing or fused optimizer (lane grep returned none) | `src/hypertrain/trainer/optim.py:138-153` |
| Only fp32/bf16 compute; no FP8 | `src/hypertrain/trainer/config.py:12` |
| Only two architectures; any non-`decoder` arch is routed to OpenDecision | `src/hypertrain/trainer/config.py:35`, `src/hypertrain/trainer/model.py:24-28` |
| Loss is causal CE only; no loss mask, no SFT/RL/diffusion objective | `src/hypertrain/trainer/model.py:271-273` |
| Batches must be exactly `u32[seq_len+1]`; no variable-length samples or multimodal inputs | `src/hypertrain/trainer/loop.py:117-119` |
| Layout invariance across GPU counts is out of scope, so there is no elastic resharding | `src/hypertrain/trainer/island.py:16` |

### 1.2 Data
| Gap | Evidence |
|---|---|
| Production shards are u16; vocab + 3 must be < 65,535 (rules out Llama-3/Qwen vocabularies) | `src/hypertrain/datasets/build.py:103-104` |
| No document-boundary masking in packing (explicit ponytail) | `src/hypertrain/data/shards.py:57` |
| `tulu-3-sft` adapter raises, though it is part of `od-b-v1`; the od-c sources have no adapters | `src/hypertrain/teacher/adapters.py:78-92` |
| Builder loads every parquet fully into memory | `src/hypertrain/datasets/build.py:68` |
| No train-train dedup; pretrain mixes use exact 13-gram decontam only | `src/hypertrain/datasets/build.py:107` |

### 1.3 Network, transport and storage
| Gap | Evidence |
|---|---|
| No entropy coding of updates | `src/hypertrain/trainer/compress.py:119-124` |
| Client uploads chunks strictly sequentially, never resumes from relay acks, and has no retries | `src/hypertrain/relay/client.py:159-166` |
| Relay re-downloads every chunk and re-uploads the whole object (2x storage and egress) | `src/hypertrain/relay/core.py:401-404` |
| Whole receive must finish in 90 s regardless of size | `src/hypertrain/data/stream_store.py:48` |
| R2 single PUT only, 5 GiB cap, no multipart | `src/hypertrain/storage/r2.py:137-145` |
| No streaming/fragmented outer sync (Streaming DiLoCo) | `src/hypertrain/aggregator/core.py:9-10` |

### 1.4 Protocol, economics and operations
| Gap | Evidence |
|---|---|
| No per-item auditor quorum. v2 manifests must list at least two auditor identities (`src/hypertrain/challenge/store.py:2142-2143`), but each reward item is accepted on one verdict from any listed auditor | `src/hypertrain/ledger/escrow_v2.py:399-401` |
| Miner self-reported rerun yields TRANSIENT forgiveness | `src/hypertrain/auditor/replay.py:857-858` |
| Audit only possible with anchor age <= 1; carry-mode miners unaudited for 2+ rounds become unauditable | `src/hypertrain/auditor/replay.py:394-395` |
| Rollback only before round w+2; later fraud stays in the model | `src/hypertrain/aggregator/core.py:846-847` |
| Roster hard caps 16/32 | `src/hypertrain/challenge/store.py:4384-4393` |
| No metrics endpoint; no logging in the challenge package | `src/hypertrain/challenge/app.py:810-816` (health only) |
| Single SQLite connection and global RLock; all writes serialised | `src/hypertrain/challenge/store.py:364-370` |
| Envelope expiry effectively infinite; replay protection rests only on the dedup table | `src/hypertrain/miner/core.py:51` |

---

## 2. What can be improved (bugs and weaknesses, by severity)

### 2.1 High
1. **64 KiB delta cap on service paths.** `validate_payload` defaults to `max_payload_bytes=65536` (`src/hypertrain/trainer/compress.py:265`). The service input path reads at most 65,536 bytes and validates with the same cap (`src/hypertrain/challenge/store.py:7454`, `:7463`), as does the capacity worker (`src/hypertrain/aggregator/capacity_worker.py:431-436`). Dense-int8 costs about 1.016 B/param (DERIVED from the body layout at `src/hypertrain/trainer/compress.py:70-74`), so these paths reject any model above about 64k parameters. This is acceptable for the toy capacity geometry, but it must become `profile.max_object_bytes` as on the other path (`src/hypertrain/challenge/store.py:5921-5925`) before production sizes.
2. **Aggregator norm math is O(d) in Python objects.** `sqnorm` and `dot` call `.tolist()` and then `math.fsum` (`src/hypertrain/aggregator/core.py:104-117`). CenteredClip calls `sqnorm` once per delta per iteration (`src/hypertrain/aggregator/core.py:244-249`). At 1B parameters that is about 1e9 Python floats per call. Replace it with a deterministic vectorised fixed-order reduction, such as float64 pairwise sums over fixed-size blocks or exact integer accumulation of the float64 mantissa, specified in the protocol.
3. **Single-party trust points.** These are: TRANSIENT self-classification (`src/hypertrain/auditor/replay.py:857-858`), one verdict per reward item (`src/hypertrain/ledger/escrow_v2.py:399-401`), a single referee, and one coordinator key. Require k-of-n auditors with distinct coldkeys, and have the referee confirm TRANSIENT.
4. **Unbounded public cache.** The cache is a `dict` keyed by arbitrary request path and is never evicted (`src/hypertrain/public_api/__init__.py:49`, `:66`). Random paths therefore exhaust memory. Use an LRU or cache only known routes.

### 2.2 Medium
5. **drand `latest()` crash.** A relay body without an int round maps to 0, and `get(0)` raises `ValueError` rather than `BeaconError` (`src/hypertrain/beacon/drand.py:164-167`, `:155`).
6. **Escrow remainder bias.** Leftover units always go to the lexicographically first hotkeys (`src/hypertrain/ledger/escrow_v2.py:449-454`). Rotate the order using the beacon. Also guard `total == 0` explicitly (`src/hypertrain/ledger/escrow_v2.py:450-451`).
7. **`assert` in production guards.** These disappear under `python -O` (`src/hypertrain/challenge/store.py:4380`).
8. **Relay timeout.** `fail_after(90)` wraps whole-object `get` calls (`src/hypertrain/data/stream_store.py:48`). A 2 GiB object would need about 190 Mbit/s to finish in time (DERIVED: 2 GiB x 8 / 90 s). Scale the deadline with the declared size.
9. **Teacher label/build key mismatch.** `iter_questions` keys on `text_field`, while the record builder keys on `_eval_text` and decontaminates before `limit` (`src/hypertrain/datasets/build.py:114`). With `--limit`, the builder therefore selects rows that were never labelled.
10. **MoE wasted compute.** Dropped tokens are still pushed through the experts and only zero-weighted (`src/hypertrain/trainer/model.py:212-219`). The input is also tiled `k` times (`src/hypertrain/trainer/model.py:214`).

### 2.3 Low and maintainability
11. `src/hypertrain/challenge/store.py` is 9,662 lines (MEASURED, `wc -l`). Split it by domain (admission, rounds, disputes, weights, objects).
12. `/health` writes to the DB under the global lock (`src/hypertrain/challenge/store.py:425`).
13. The k8s relay runs a single uvicorn worker per pod and serves plain HTTP on port 8000 (`deploy/k8s/base/deployment.yaml:42`). The docs describe 443/8443, which is inconsistent.
14. The relay image digest pinned in `deploy/k8s/base/kustomization.yaml:13-14` is never bumped by the release workflows.

### 2.4 Documentation inconsistencies (DERIVED checks)
- `docs/parity.md:33` claims "100 GB per sync, about 80 Gbit/s at `H_r = 10` and a 10 s step". By the stated formula `bytes / (H * t_step)`, the rate is 100e9 B / 100 s = 1 GB/s = **8 Gbit/s**. SparseLoCo's 3.9 GB / 100 s is **0.31 Gbit/s**, not "about 3". Both figures are 10x too high.
- `docs/operator.md:151` says the Phase B profile pins `reset`, but `experiments/gpu_phase_b/phase_b.json:35` is `"state_policy": "carry"`.
- `docs/parity.md:83-86` tags numbers against `experiments/results/summary_smoke.json`. That file is not tracked; `git ls-files experiments/results` lists 9 files and it is not among them (MEASURED).

---

## 3. Tests that should be run

### 3.1 Existing gates (run in this audit; results in `build.json`)
| Command | Purpose |
|---|---|
| `uv run --frozen ruff format --check` | format |
| `uv run --frozen ruff check` | lint, including bandit `S` rules (`pyproject.toml:69-74`) |
| `MYPYPATH=src:scripts:. uv run --frozen mypy --explicit-package-bases --package hypertrain ...` | types (`scripts/check.sh:7-9`) |
| `uv run --frozen pytest -q --ignore=tests/sim --ignore=tests/e2e` | main suite (about 960 test functions across 23 directories) |
| `uv run --frozen pytest -q tests/sim tests/e2e` | slow suite (CI job `test-slow`) |

### 3.2 Gated tests that never run automatically
- The genuine CUDA cases skip without `HT_GENUINE_CUDA_INDEX` (`tests/challenge/test_backend_authority_v2.py:132`, `tests/miner/test_island_launch_v2.py:807`).
- The gitleaks check skips without the binary, and CI never installs it (`tests/test_publish_exclusions.py:25`).
- `.github/workflows/gpu-vast.yml` runs only after merge or manually. `images.yml` publishes the GPU image without a GPU test.

### 3.3 Tests to add (prioritised)
| # | Category | Concrete test |
|---|---|---|
| T1 | Property/fuzz | Hypothesis round-trip and rejection for `compress`/`decompress` and `_payload_metadata`: random shapes, NaN/Inf, truncation, reordered names, padding bits. Add `hypothesis` to the dev group in `pyproject.toml`. |
| T2 | Fuzz | JCS canonicalisation and envelope parsing against the RFC 8785 vectors; bounded-size malformed JSON (`src/hypertrain/protocol/jcs.py`, `src/hypertrain/protocol/envelope_v2.py`). |
| T3 | Size regression | Real-size payload path: a 10M-param dense-int8 delta through `store.py:7454-7466` and `capacity_worker.py:436`. It fails today, which locks in the 64 KiB finding. |
| T4 | Cross-hardware determinism | A matrix of {RTX 5090, H100, A100, L40S} x {driver N, N-1} replaying the same round. Assert identical leaves, or an explicit qualified-set rejection. |
| T5 | MFU/perf regression | A CI GPU job that records tok/s, MFU and peak memory for fixed profiles, with a threshold gate. Extend `experiments/gpu_phase_b/run_phase_b.py` with FLOPs/token. |
| T6 | Adversarial e2e | Copy, interpolation, scaled, sign-flip, low-rate poisoning and Sybil miners against CenteredClip + copy screen on GPU. Measure catch rate and false positives against honeypots (`src/hypertrain/auditor/honeypot.py`). |
| T7 | Chaos/network | netem latency/loss/partition between miner, relay and service; relay pod kill mid-upload; resume from `/acks`. |
| T8 | Load/soak | 32-miner rosters for 100 rounds; service SQLite write latency; public API under random-path flood (finding 2.1.4). |
| T9 | Economics invariants | Property tests: escrow conservation, zero-token work, vesting/clawback with Python-int vs SQLite-int64 bounds (`src/hypertrain/ledger/escrow_v2.py:449-454`). |
| T10 | Mutation testing | mutmut on `protocol/`, `ledger/`, `auditor/replay.py`. Surviving mutants mark missing assertions. |
| T11 | Supply chain | `pip-audit` against `uv.lock`, gitleaks in CI, an SBOM for both images, and hash-pinning the opendecision tarball in `docker/Dockerfile.gpu`. |
| T12 | Docs numbers | Run `scripts/check_doc_numbers.py` in CI. It must fail on the missing `summary_smoke.json` (section 2.4). |
| T13 | Direct unit tests | `src/hypertrain/gpu_ops/budget.py`, `src/hypertrain/gpu_ops/verdict.py` and `src/hypertrain/miner/island_worker.py` have no direct test imports (lane-tests-ci grep). |
| T14 | Quality parity | Rerun the pre-registered TOST grid with at least 8 seeds; the smoke power analysis requires 8 (`docs/parity.md:86`). |

---

## 4. Raising MFU

### 4.1 Baseline (DERIVED)
- FLOPs/token = `6 * active_params + 12 * L * d * T`.
- Peak = **ASSUMED** 209.5 TFLOPS dense BF16 per RTX 5090 (vendor dense tensor figure).
- tok/s is MEASURED wall-clock, including leaf hashing (`experiments/results/phase_b_summary.json:24-43`, `experiments/results/phase_a_summary.json:132`).

| Run | Active params | tok/s | MFU |
|---|---|---|---|
| Phase B h1, deterministic, 8 GPUs | 201.9M | 2000.84{ev:experiments/results/phase_b_summary.json#b2.per_host.h1.tok_per_s} | **0.16%** |
| Phase B h1, baseline (non-deterministic) | 201.9M | 4125.753{ev:experiments/results/phase_b_summary.json#b2.per_host.h1.base_tok_per_s} | 0.34% |
| Phase B h0, deterministic | 201.9M | 1072.125{ev:experiments/results/phase_b_summary.json#b2.per_host.h0.tok_per_s} | 0.09% |
| Phase A, 1 GPU, deterministic | 33.8M | about 43,890 (200 x 8 x 1024 / 37.33{ev:experiments/results/phase_a_summary.json#runs.0.result.seconds} s) | 5.3% |

Sensitivity: a different peak assumption scales MFU linearly. Even at 100 TFLOPS, Phase B stays below 0.4%. The full computation is in `research/verify-mfu.json`. Reference points: torchtitan reports 33-39% MFU for Llama 3.1 8B on H100s, and INTELLECT-1 reports 36-41% MFU under DiLoCo (section 7 sources).

### 4.2 Plan (ordered by expected gain per unit of determinism risk)
Every change must stay bitwise-reproducible on qualified hardware. "Deterministic" here means *fixed reduction order*, not *slow*.

1. **Fused deterministic attention.**
   - Replace `model.py:174-176` with FlashAttention forward, which is deterministic. For the backward, use FA2/FA3 `deterministic=True`, or a batch-invariant Triton kernel with a fixed split.
   - Expected effect: removes the O(T²) fp32 materialisation and lifts attention from memory-bound to compute-bound.
   - Native GQA support also removes the K/V copy at `model.py:171-173`.
   - Qualify it with the existing op-level trace parity tests (`tests/auditor/test_trace_parity.py`).
2. **Allow TF32/bf16 tensor-core paths with pinned kernels.**
   - `determinism.py:54-58` disables TF32 and reduced-precision reduction globally.
   - Determinism comes from fixed kernel selection and split-K, not from the precision mode. Pin cuBLASLt algorithms (or use a CUTLASS / batch-invariant matmul with fixed tiles) and allow bf16 accumulation where the qualified kernel set fixes the order.
   - Phase B shows that these flags plus the gather-based reduction cost about 2x (MEASURED: 1072.125{ev:experiments/results/phase_b_summary.json#b2.per_host.h0.tok_per_s} vs 2111.303{ev:experiments/results/phase_b_summary.json#b2.per_host.h0.base_tok_per_s} tok/s on h0).
3. **Replace all-gather + serial sum with a deterministic reduce-scatter.**
   - `island.py:524-543` moves `world x` bytes per tensor and starts only after the full backward (`src/hypertrain/trainer/island.py:698-699`).
   - Use a fixed-order ring or tree reduce-scatter on flat bucketed buffers (custom NCCL-free kernel or a fixed-topology NCCL `ReduceScatter` with `NCCL_ALGO=Ring`, `NCCL_PROTO=Simple`, pinned channels).
   - Overlap it with backward per bucket. ZeRO-1 then needs only a reduce-scatter plus an all-gather of params, instead of re-gathering theta every step (`island.py:727-731`).
4. **Kill per-step host syncs.**
   - `float(ce.detach())` (`island.py:520`).
   - Per-tensor `float()` norms (`optim.py:118`, `island.py:549-550`).
   - `bincount(...).tolist()` per MoE layer (`model.py:215`).
   - Keep these values on device and sync once per leaf (every J steps).
5. **Fused, foreach optimizer.** Turn `optim.py:138-153` into a single multi-tensor AdamW/Muon kernel with a fixed per-tensor order; this is deterministic by construction. Batch Muon Newton-Schulz across same-shape matrices.
6. **Stop re-casting weights every use.** `p[...].to(dt)` runs on every layer and microbatch (`model.py:166-168`, `:217`, `:266`). Keep a bf16 shadow copy that is refreshed once per optimizer step.
7. **`torch.compile` / CUDA graphs** on the fixed-shape forward and backward, with Inductor's deterministic mode, and the compiled graph hash added to the qualified-kernel manifest.
8. **MoE.**
   - Grouped-GEMM experts instead of a Python loop (`model.py:216-219`).
   - Skip dropped tokens instead of zero-weighting them (`model.py:212`).
   - Avoid `flat.repeat(k, 1)` (`model.py:214`).
   - Send bf16 in `all_to_all` instead of upcasting to fp32 (`island.py:175-176`); the upcast doubles EP bytes.
9. **Leaf cost.** `state_hash` copies theta, m and v device-to-host at every leaf (`compress.py:37-44`). Hash on GPU with a fixed-order tree hash (e.g. BLAKE3 over fixed chunks) and copy only the digest. Write checkpoints asynchronously.
10. **Data loading.** `_load_batch` is synchronous numpy with no pinned memory or prefetch (`loop.py:108-123`). Use a double-buffered pinned-memory prefetcher; the sample order is already deterministic.
11. **Activation checkpointing** (deterministic recompute) to raise micro-batch size. MFU on small micro-batches (2 per rank in Phase B) is dominated by launch overhead.
12. **Fix the model shapes used for qualification.** Vocab 259 and d=1024 are too small to saturate tensor cores (`experiments/gpu_phase_b/phase_b.json:19-28`). Qualify on shapes from the target ladder (0.5B → 7B, per `docs/RESULTS.md`).
13. **FP8** (Transformer Engine / DeepSeek-V3-style blockwise scaling) as a later tier, with deterministic amax history.

Expected outcome (ASSUMED, not measured): items 1-6 bring a dense 1B-class model into the 25-40% MFU band reported by the deterministic-friendly stacks above. Each item needs a Phase-B-style A/B with bitwise replay.

---

## 5. Lossless compression and connection

### 5.1 Current state
- The codecs are `dense-int8` (8.125 bits/value) and `sparseloco` (top-k with u32 indices plus 2-bit codes, about 34 bits per kept value). Both are lossy with error feedback (`src/hypertrain/trainer/compress.py:1-11`).
- No entropy coder exists anywhere in the update path (lane-transport grep).

### 5.2 Measured headroom (this audit)
Command: `uv run --frozen --with zstandard python ../research/verify_lossless.py`. Output is in `logs/verify_lossless.log`.

Proxy input (ASSUMED): Student-t (df=3) pseudo-gradients, 4,194,304 params, top-k 3.125%.

| Codec payload | Raw bytes | zlib-9 | lzma | zstd-19 | gap-coded indices + byte-shuffle + zstd-19 |
|---|---|---|---|---|---|
| sparseloco | 557,284 | 229,981 | 162,724 | 198,639 | **124,152 (4.49x)** |
| dense-int8 | 4,260,024 | 3,457,124 | 3,408,028 | 3,426,748 | 3,411,796 (1.25x) |

The sparse gain comes almost entirely from the ascending u32 indices. Gaps are small, so their high bytes are zero and byte-plane shuffling exposes that. Real pseudo-gradients have structured top-k sets (rows and columns), so they should compress at least as well. This is unverified; measure it on Phase B deltas.

### 5.3 Plan ("maximum lossless")
1. **Codec v2 = existing quantiser + lossless container.** Keep the lossy step bit-identical, which preserves EF semantics and replay. Add these stages:
   - **Indices:** delta (gap) encode, then Elias-Fano or rANS over the gap distribution. Elias-Fano is near the information-theoretic bound of about `log2(n/k) + 2` bits per index: about 7 bits at 3.125% density versus 32 bits today (DERIVED).
   - **2-bit codes and int8 values:** adaptive rANS or zstd with a per-tensor histogram. int8 values from block-amax scaling have a peaked distribution.
   - **Scales:** byte-plane shuffle, then zstd. Keep scales in a stream separate from the values; today they are interleaved (`compress.py:70-74`).
   - **Framing:** stream per tensor so encode and decode never hold the whole payload (`compress.py:177`, `b"".join`).
   - **Determinism:** pin the encoder version and level, and define the canonical encoding (e.g. zstd frames with a fixed level and no dictionary, or a spec'd rANS). The payload hash stays a function of the inputs. Validation decodes into the canonical form and rejects non-canonical encodings.
2. **Reuse the checkpoint path.** For model/state transfers (late joiners, rollback), use ZipNN-style byte grouping with Huffman/zstd on bf16/fp32. Published ratios are about 1.5x on bf16 and 1.17x on fp32. XOR-delta against the previous round's checkpoint should help further on slowly changing weights.
3. **GPU-side codec** (DietGPU/nvCOMP ANS at 250+ GB/s) so compression never costs MFU.
4. **Connection:**
   - Upload chunks in parallel up to the relay's limiter of 4 (`src/hypertrain/relay/core.py:156` per the lane), instead of the sequential loop at `src/hypertrain/relay/client.py:159-166`.
   - Resume from `GET .../acks` after a failure, and retry with jittered backoff.
   - Enable HTTP/2 in httpx.
   - Scale `fail_after` by size (`src/hypertrain/data/stream_store.py:48`).
   - Fail over across `fallback_ids` in the client.
   - Stop the relay from re-assembling the whole object (`src/hypertrain/relay/core.py:401-404`). Store chunks plus a signed chunk manifest, and let consumers stream them.
5. **Protocol-level bandwidth** (lossy, but the largest win): Streaming DiLoCo fragment sync overlapped with compute cuts *peak* bandwidth about 100x at similar quality. This changes the commitment unit to a fragment, so the audit tape needs per-fragment leaves.

---

## 6. What a complete network for training every model type needs

### 6.1 Model zoo and engine
- A model registry with an `arch` dispatch table, replacing "anything not decoder means OpenDecision" (`src/hypertrain/trainer/model.py:24-28`).
- Per architecture: param shapes, init, forward, the qualified kernel set, and op-trace hooks for bisection.
- Required families:
  - Decoder LM (dense and MoE with shared experts, MLA)
  - Encoder (MLM)
  - Encoder-decoder (cross-attention)
  - Vision (ViT patch embed)
  - Multimodal (vision tower + projector + LM)
  - Diffusion / flow matching (timestep conditioning, noise schedule, a deterministic RNG stream already exists in `src/hypertrain/trainer/rng.py`)
  - SSM / hybrid (deterministic chunked scan)
  - Long-context variants (RoPE scaling, sliding window, context parallelism)
- Execution parallelism: TP and PP (1F1B with fixed microbatch order), plus context parallelism, all with fixed reduction order. Layout-invariant checkpoints are needed so islands with different GPU counts can join (`src/hypertrain/trainer/island.py:16`).
- Objectives: CE with a loss mask, MLM, contrastive, diffusion MSE, distillation (KL against cached teacher logits), and preference/RL losses (DPO, GRPO) with verified rollouts (TOPLOC-style).

### 6.2 Data
- A shard format v3 with:
  - a dtype field (u16/u32);
  - a variable-length record type with sidecars for doc boundaries, loss mask, segment ids and modality blobs (image/audio bytes with a content hash);
  - per-unit Merkle roots as today.
- The `fetch.unit_ranges` byte width must come from the manifest instead of being hardcoded.
- Streaming builder, train-train dedup (MinHash LSH), curriculum and mixing schedules in the manifest, and long-context packing.

### 6.3 Network and protocol
- Elastic membership: mid-run join with verified state sync, using a checkpoint plus a compressed state stream. This replaces fixed per-round rosters and the 16/32 caps.
- Fragmented and asynchronous outer sync (Streaming DiLoCo), heterogeneous islands, and an outer LR schedule.
- Auditor quorum with stake, auditor slashing driven by the honeypot catch rate, multi-referee dispute courts, and a threshold coordinator key (or on-chain commitments) instead of one key.
- Rollback beyond w+2 via checkpoint lineage plus replay, so poisoning detected late can be excised.
- Horizontal scale for the service: Postgres or another replicated store instead of one SQLite file with a global lock, object store per run, metrics and tracing, rate limiting on all routes.

### 6.4 Evidence
- The scale ladder (0.5B → 1-3B → 7B) with loss curves.
- The TOST parity grid with at least 8 seeds.
- A compression ablation (codec x density x EF) reporting bytes/round and loss impact.
- Cross-vendor determinism qualification, and a public honeypot catch-rate dashboard.

---

## 7. External sources (accessed 2026-10-11)
1. DiLoCo, arXiv 2311.08105. https://huggingface.co/papers/2311.08105
2. Streaming DiLoCo, arXiv 2501.18512. https://huggingface.co/papers/2501.18512
3. DiLoCo scaling laws, arXiv 2503.09799. https://huggingface.co/papers/2503.09799
4. SparseLoCo, arXiv 2508.15706. https://huggingface.co/papers/2508.15706
5. INTELLECT-1 (36.2-41.4% MFU, int8 all-reduce), arXiv 2412.01152. https://huggingface.co/papers/2412.01152
6. OpenDiLoCo, arXiv 2407.07852. https://huggingface.co/papers/2407.07852
7. ZipNN (bf16 about 1.5x lossless). https://github.com/zipnn/zipnn ; arXiv 2411.05239
8. DFloat11. https://huggingface.co/papers/2504.11651
9. DietGPU (GPU rANS). https://github.com/facebookresearch/dietgpu
10. PyTorch reproducibility notes. https://github.com/pytorch/pytorch/blob/main/docs/source/notes/randomness.md
11. FlashAttention deterministic backward. https://github.com/Dao-AILab/flash-attention
12. Thinking Machines, "Defeating Nondeterminism in LLM Inference". https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/
13. torchtitan H100 benchmarks. https://github.com/pytorch/torchtitan/blob/main/docs/benchmarks/llama3_h100_202412_torchtitan.md
14. Gensyn Verde/RepOps, arXiv 2502.19405. https://huggingface.co/papers/2502.19405
15. Templar Gauntlet, arXiv 2505.21684. https://huggingface.co/papers/2505.21684
16. INTELLECT-2 / TOPLOC, arXiv 2505.07291. https://huggingface.co/papers/2505.07291
17. Decentralized Diffusion Models, arXiv 2501.05450. https://huggingface.co/papers/2501.05450

## 8. Method
- Research team: 7 read-only codebase lanes (trainer, transport, aggregator/auditor/ledger, challenge/miner, data, docs/experiments, tests/CI) and 1 web lane. Raw lane outputs are in `research/wave-1-*.md`.
- The auditor re-read every high-severity citation before writing it.
- Executed verifications: `research/verify_lossless.py` and `research/verify-mfu.json`.
- Not done: no GPU run (none available on this host); no web claim beyond the cited pages; the MuLoCo, Psyche and nvCOMP figures were not retrieved.
