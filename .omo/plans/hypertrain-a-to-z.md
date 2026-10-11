# Hypertrain A-to-Z improvement plan

Job 3924dab79a27. Source audit: job 224a523e5f92 (`audit-audit.md`, `audit-audit.json`). Base: `8baf7c869f80074455fbcfcf894c302e198e8ce5` on `boat/3924dab79a27`. Phase: plan only; nothing in `src/` changes in this phase.

Machine-readable twin: `plan.json` (same content, plus the ordered backlog). Line citations are `path:line` at the base SHA and were spot-checked against the tree in this job.

## Targets and how each is proven

| Goal | Pass condition | Proof |
|---|---|---|
| MFU > 40% on RTX 5090 | `mfu-dense-1b` (~1.1B dense, seq 2048, vocab 32768, bf16) min MFU over 3 independent Vast hosts > 40%, 1x and 8x GPU, determinism ON | `run_mfu.py` JSON with raw step timings, FLOPs/token `6*N_active + 12*L*d*T`, peak 209.5 TFLOPS per GPU x n_gpus (ASSUMED vendor) and measured GEMM peak, replay leaf-root match; recomputed by `check_gpu_evidence.py --kind mfu` |
| Maximum lossless update compression | v2 container output within 5% of the summed per-stream empirical entropy bound; >= 4x smaller than v1 sparseloco on real deltas; `ef_out` bitwise identical to v1 | entropy-bound JSON on real Phase-B/1B deltas + `test_codec_v2.py` |
| Complete network | 3 regional CPU aggregators agree 2-of-3 on every round; join/leave mid-run with verified state sync; k-of-n auditor quorum; registry trains every listed family | 20-round live run journal (WS6.6) + per-family CPU bitwise tests and 5090 smoke JSON |

Baseline today: 0.16% MFU (Phase B h1, deterministic, DERIVED in audit 4.1). The jump to 40% depends on WS1 items 1.1-1.11 together; no single item gets there.

## Owner decisions (APPROVED 2026-10-12, see decisions.md: D1 USD 200 per run, D2 deps, D3 Vast CPU, D4 additive versioning)

- **D1 Total GPU/CPU rental budget.** Sum of workstream ceilings is USD 440. Last recorded Vast credit was USD 20.95 (docs/RESULTS.md section 7, read 2026-10-08). Every rental task is blocked until the owner tops up or lowers scope; the harness refuses to start a rental whose worst case exceeds max_usd or credit-5 (scripts/vast_e2e.py:214-258). Default if declined: none - spend is owner-only.
- **D2 New dependencies.** zstandard (codec back-end), hypothesis (dev tests), h2 (httpx HTTP/2), flash-attn (only if 1.1 spike selects backend b). Each is pinned in pyproject.toml/uv.lock or docker/gpu-requirements.*. Default if declined: if refused: stdlib lzma back-end (2.4), stdlib random-based property tests, HTTP/1.1, SDPA or Triton attention.
- **D3 CPU node provider for regional aggregators.** Vast.ai CPU offers are sparse per region; alternative is any hourly cloud VM with the spec in WS6. Needs an account the agent can verify. Default if declined: Vast.ai CPU offers if available in EU/US/APAC, else blocked until owner names a provider.
- **D4 Protocol version bumps.** ht-leaf-v2 (1.8), codec v2 MAGICs (2.2), ht-norm-v2 (4.2), precision_policy and activation_checkpointing fields (1.2, 1.10), auditor_quorum (5.1), max_roster (4.3). Old runs keep v1 paths; new run_ids change. Default if declined: additive versioning, v1 retained for replay of existing runs.

## Global rules

- Determinism is non-negotiable: every engine change must keep bitwise replay on qualified hardware; a speedup without replay-match is rejected.
- No change to the lossy quantiser output: ef_out and dequantised deltas stay bitwise identical between codec v1 and v2.
- Every rental uses scripts/vast_e2e.py-style guards: --max-usd hard ceiling, --max-minutes deadline, delete + absence verification, journal of charges; API keys only via key file, never printed.
- Every number in docs carries an {ev:path#key} tag checked by scripts/check_doc_numbers.py.
- Quality gates per task: uv run --frozen ruff format --check; ruff check; mypy (scripts/check.sh flags) after `uv sync --locked --all-extras --group dev`; pytest for touched packages; slow suite for e2e-touching tasks.
- Must NOT: merge, deploy, push to main, weaken forbid_reductions, enable nondeterministic algorithms, add per-PR paid GPU CI, store evidence with credentials in git.

## Budget summary

| Workstream | Rental | Task sum USD | Ceiling USD |
|---|---|---|---|
| WS0 Measurement foundation (MFU harness, FLOPs model, evidence schema) | Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand for calibration only | 8 | 10 |
| WS1 Training engine to MFU > 40% on RTX 5090 (deterministic) | Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand for A/B iterations; Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py) for the final island run; Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py) (two hosts) for cross-host bitwise check | 107 | 135 |
| WS2 Maximum lossless compression of updates (codec v2 container) | Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py) (one Phase-B-shape round to capture real deltas); rest local CPU | 15 | 20 |
| WS3 Connection and transport | local kind cluster (deploy/k8s/local) + 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3) for cross-region throughput test | 10 | 15 |
| WS4 Production-size limits and aggregator scale | local CPU; 1 CPU node for 1B-param aggregation timing | 1 | 5 |
| WS5 Trust: auditor quorum and single-party fixes | local CPU | 0 | 0 |
| WS6 Complete network: multi-region aggregators and elastic membership | 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3); plus Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py) x2 hosts as miners for the live run | 60 | 75 |
| WS7 Model registry and every model family | Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand (1 smoke run per family) | 70 | 90 |
| WS8 Data format v3 | local CPU | 0 | 0 |
| WS9 Docs correctness and CI gates | GitHub-hosted runners; GPU via gpu-vast.yml | 70 | 90 |
| **Total** | | **341** | **440** |

Cost basis (ASSUMED, re-priced from live offers at rental time): RTX 5090 about USD 0.40-0.70/GPU-h on Vast; regional CPU node about USD 0.15-0.30/h. Ceilings are hard `--max-usd` limits, not estimates.

## Workstreams

### WS0. Measurement foundation (MFU harness, FLOPs model, evidence schema)

Goal: Every later MFU claim is computed by one reviewed script from raw timings, with a pinned formula and both peak denominators.

Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand for calibration only. Cost ceiling: USD 10.

Exit evidence: Baseline MFU JSON for both dense profiles with replay-match, recomputed by check_gpu_evidence.py.

#### 0.1 FLOPs/token model and MFU calculator

- Files: `src/hypertrain/trainer/perf.py (new)`, `tests/trainer/test_perf.py (new)`
- Changes:
  - Add flops_per_token(cfg: ModelConfig) -> int implementing 6*N_active + 12*L*d*T (PaLM convention, no causal halving; N_active counts top_k_experts/n_experts of expert weights, excludes embedding gather, includes head).
  - Add mfu(tokens_per_s, flops_per_token, peak_flops) -> float and MFU_PEAK_RTX5090_BF16 = 209.5e12 with a docstring naming the vendor source and labelling it ASSUMED.
  - Pure Python, no torch import at module load.
- Tests:
  - test_perf.py: hand-computed FLOPs for phase_b profile (654,965,760 total / ~201.9M active) and for a dense 1.1B profile; MoE active-param accounting; rejects non-positive inputs.
  - Cross-check: flops_per_token(phase_b) * 2000.84 / 209.5e12 reproduces audit MFU 0.16% (+-0.01).
- Evidence: pytest log for tests/trainer/test_perf.py exit 0; reproduction of audit section 4.1 table within 0.01 pp
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 0.2 Achievable-peak microbenchmark

- Files: `experiments/mfu/peak_matmul.py (new)`, `experiments/mfu/README.md (new)`
- Changes:
  - bf16 GEMM sweep (M=N=K in {4096, 8192, 16384}), fp32 accumulate, CUDA events timing, 50 warmup + 200 timed iters, report max TFLOPS and nvidia-smi clocks/power/temp before and after.
  - Output JSON: {gpu_name, driver, cuda, torch, clocks, tflops_by_shape, measured_peak}.
- Tests:
  - tests/test_mfu_scripts.py (new): CPU dry-run mode (--dry-run) validates JSON schema without CUDA.
- Evidence: experiments/results/mfu/peak-<host>.json committed; Measured peak reported next to 209.5 TFLOPS vendor peak
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 3
- Depends on: 0.1

#### 0.3 Reproducible MFU benchmark runner

- Files: `experiments/mfu/run_mfu.py (new)`, `experiments/mfu/profiles.json (new)`, `scripts/check_gpu_evidence.py`, `tests/test_mfu_scripts.py`
- Changes:
  - run_mfu.py drives hypertrain.trainer.island.run (the production path, determinism ON) for a profile and records: steps, wall seconds per step (CUDA-synchronised), tokens/s, flops_per_token, MFU vs vendor peak and vs measured peak, peak memory, leaf-hash time share, git SHA, image digest, env_record from experiments/gpu_phase_b/run_phase_b.py:214.
  - Steady-state window: discard first 10 steps, time next >=50 steps; tokens counted from the assignment, not estimated.
  - Two modes per run: --deterministic (production flags) and --replay that re-runs the same steps and asserts identical leaf roots (bitwise reproducibility proof attached to every MFU number).
  - profiles.json: mfu-dense-1b (n_layers=16, d_model=2048, n_heads=16, n_kv_heads=8, d_ff=5632, vocab=32768, seq_len=2048, bf16), mfu-dense-0.5b, mfu-moe-1b (8 experts top-2), phase_b (regression).
  - check_gpu_evidence.py: add `mfu` evidence kind that recomputes MFU from raw step timings and rejects any reported MFU that differs by >0.1 pp or lacks the replay leaf-root match.
- Tests:
  - tests/test_mfu_scripts.py: CPU tiny-profile run end-to-end (gloo, 1 rank) produces schema-valid JSON; checker rejects tampered MFU and missing replay match.
- Evidence: experiments/results/mfu/<profile>-<sha>.json schema-valid; check_gpu_evidence.py exit 0 on it
- Rental: none (local CPU); cost USD 0
- Depends on: 0.1, 0.2

#### 0.4 Baseline MFU on rented 5090 before any engine change

- Files: `experiments/results/mfu/baseline-*.json (new evidence)`, `docs/RESULTS.md (new MFU section)`
- Changes:
  - Run run_mfu.py for mfu-dense-0.5b and mfu-dense-1b on 1x 5090 at the current HEAD; record numbers as the A in every later A/B.
- Tests:
  - n/a (evidence task)
- Evidence: Baseline JSON + replay match + Vast journal (instance id, dph, total charge) under the job evidence dir; Charge <= ceiling read back from Vast GET
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 5
- Depends on: 0.3

### WS1. Training engine to MFU > 40% on RTX 5090 (deterministic)

Goal: mfu-dense-1b reaches MFU > 40% vs the 209.5 TFLOPS vendor peak on 1x and 8x RTX 5090 with determinism ON and bitwise replay of every leaf.

Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand for A/B iterations; Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py) for the final island run; Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py) (two hosts) for cross-host bitwise check. Cost ceiling: USD 135.

Exit evidence: final-*.json: min MFU > 40% on 3 hosts for 1x and 8x 5090, replay bitwise, checker exit 0; Cross-host leaf equality log.

#### 1.1 Fused deterministic causal attention with native GQA

- Files: `src/hypertrain/trainer/attention.py (new)`, `src/hypertrain/trainer/model.py:162-178`, `src/hypertrain/trainer/determinism.py`, `docker/gpu-requirements.in`, `docker/gpu-requirements.txt`, `tests/trainer/test_attention.py (new)`, `tests/auditor/test_trace_parity.py`
- Changes:
  - attention(q, k, v, causal=True) -> Tensor dispatching to: (a) torch SDPA flash backend when torch.are_deterministic_algorithms_enabled() permits it on the device, else (b) flash-attn `flash_attn_func(..., deterministic=True)` pinned in gpu-requirements, else (c) the current math path (CPU and fallback). Selection is a pure function of (device capability, torch version, installed kernel) and its name is written into the qualified-kernel manifest so auditors replay the same backend.
  - Decision rule (no judgment left): spike runs tests/trainer/test_attention.py::test_backward_bitwise on 5090; the first backend in order (a),(b) that is bitwise-stable over 20 repeated backward passes and matches math path within bf16 tolerance (max abs err < 2e-2 on logits) is chosen; if neither passes, implement (d) a Triton fixed-split flash kernel in attention.py.
  - Remove K/V expand/reshape at model.py:171-173 (pass n_kv_heads to kernel), remove per-layer mask rebuild at model.py:175.
- Tests:
  - test_attention.py: CPU math-path equivalence vs current implementation (bitwise on fp32), GQA shapes, causal mask, T in {1, 17, 2048}.
  - GPU-gated (HT_GENUINE_CUDA_INDEX) test_backward_bitwise: 20 repeated fwd+bwd identical bytes.
  - test_trace_parity.py: op-trace hook names unchanged ('attn').
- Evidence: Spike log with backend chosen + bitwise result; A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded; Cross-host replay identical leaves (2 hosts)
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 10
- Depends on: 0.4

#### 1.2 Pinned tensor-core matmul policy (TF32/bf16 reductions under fixed kernels)

- Files: `src/hypertrain/trainer/determinism.py:51-58`, `src/hypertrain/gpu_ops/network_qualification.py`, `tests/trainer/test_determinism_policy.py (new)`
- Changes:
  - Introduce precision_policy in reference_spec.env: 'strict' (current flags) and 'pinned-tc' (TF32 allowed for fp32 matmuls, bf16 inputs with fp32 accumulate; allow_bf16_reduced_precision_reduction and allow_fp16_reduced_precision_reduction stay False; CUBLAS_WORKSPACE_CONFIG pinned so split-K choice is fixed; cuBLASLt heuristics cache disabled).
  - Policy name signed into run_id via protocol messages_v2 reference_spec (schema bump, both docs/schemas updated).
  - Qualification (network_qualification.py) records policy and rejects mixed-policy rounds.
- Tests:
  - test_determinism_policy.py: setup returns exact flag dict per policy; unknown policy rejected; protocol round-trip of new field; old manifests default to 'strict'.
- Evidence: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded; Bitwise replay under pinned-tc on 2 hosts
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 6
- Depends on: 1.1

#### 1.3 Kill per-step host syncs

- Files: `src/hypertrain/trainer/island.py:520,546-560`, `src/hypertrain/trainer/optim.py:115-119`, `src/hypertrain/trainer/model.py:215`, `src/hypertrain/trainer/loop.py`
- Changes:
  - Keep loss sum, grad-norm partials as device tensors; single .item() per leaf (every J steps).
  - grad norm: fixed-order float64 device reduction torch.stack([g.double().pow(2).sum() for g in sorted]) summed with a fixed pairwise tree; clip computed on device.
  - MoE counts: replace bincount(...).tolist() + torch.split with grouped kernel (see 1.7); until then counts are computed once per microbatch, not per layer copy.
- Tests:
  - tests/trainer/test_trainer_core.py: leaf roots for tiny profile bitwise unchanged on CPU (golden digest captured before the change).
  - New test: no host sync inside step via torch.cuda.set_sync_debug_mode('error') on GPU-gated run.
- Evidence: Golden-digest equality on CPU; A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded
- Rental: none (local CPU); cost USD 0
- Depends on: 0.4

#### 1.4 Deterministic bucketed reduce-scatter overlapped with backward

- Files: `src/hypertrain/trainer/island.py:522-543,727-731`, `src/hypertrain/trainer/collectives.py (new)`, `experiments/gpu_phase_b/nccl_bench.py`, `tests/layout/test_reduce_scatter.py (new)`
- Changes:
  - Flat 25 MiB buckets in sorted-name order; per bucket a fixed-order ring reduce-scatter implemented over point-to-point send/recv in rank order (no NCCL reduction ops, so forbid_reductions stays valid), fp32 accumulation in ring order; launched from backward hooks per bucket.
  - ZeRO-1: owners step their shard, then one all_gather of updated params per bucket; drop full theta re-gather at island.py:727-731.
  - Keep _reduce('all_gather') as reference implementation for the emulator; ThreadComm gains send/recv so emulate reproduces the ring order bitwise.
- Tests:
  - test_reduce_scatter.py: ThreadComm world sizes 2/4/8, results bitwise equal to a numpy reference that sums in the same ring order; differs from all_gather order only within documented fp32 rounding (asserted to be a different-but-fixed digest).
  - tests/layout existing suite green.
- Evidence: Emulator vs 8-GPU leaves bitwise; A/B on 8x 5090: tok/s and MFU; Cross-host bitwise (2 x 8 GPU optional, else 2 x 2 GPU)
- Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py); cost USD 20
- Depends on: 1.3

#### 1.5 Multi-tensor fused optimizer (AdamW + Muon batched NS)

- Files: `src/hypertrain/trainer/optim.py:122-160`, `tests/trainer/test_optim_fused.py (new)`
- Changes:
  - AdamW via torch._foreach_* in sorted-name order (deterministic: elementwise only).
  - Muon: group same-shape matrices, run newton_schulz batched (bmm) - elementwise-identical per matrix since bmm on independent batch entries uses identical kernels; verify by test.
  - Weight decay and clip folded into the foreach chain.
- Tests:
  - test_optim_fused.py: fused vs current loop bitwise equal on CPU fp32 for 50 steps (AdamW, Muon, derived/carry policies); GPU-gated bitwise check.
- Evidence: CPU bitwise equality log; A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded
- Rental: none (local CPU); cost USD 0
- Depends on: 1.3

#### 1.6 bf16 weight shadow refreshed once per optimizer step

- Files: `src/hypertrain/trainer/model.py:166-168,217,266,270`, `src/hypertrain/trainer/island.py`
- Changes:
  - forward() accepts an optional pre-cast params dict; island keeps p_bf16 updated after each step with one foreach copy; fp32 master unchanged (leaves still hash fp32 masters).
- Tests:
  - tests/trainer/test_trainer_core.py golden digests unchanged; new test asserts no .to(dt) inside forward when shadow supplied (op counter).
- Evidence: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded
- Rental: none (local CPU); cost USD 0
- Depends on: 1.5

#### 1.7 MoE: grouped GEMM, drop skipped tokens, no k-fold repeat, bf16 all_to_all

- Files: `src/hypertrain/trainer/model.py:201-225`, `src/hypertrain/trainer/island.py:170-180`, `tests/trainer/test_moe_grouped.py (new)`
- Changes:
  - Gather only kept (token, slot) rows via index_select on flat without repeat; experts run with torch._grouped_mm (or a padded bmm fallback with fixed padding) in expert order; combine via existing fixed-order sum.
  - all_to_all carries compute dtype (bf16) instead of fp32.
- Tests:
  - test_moe_grouped.py: forward/backward equal to current _moe on CPU fp32 (bitwise for kept tokens; dropped tokens contribute exactly zero); capacity edge cases; EP emulator bitwise.
- Evidence: CPU equivalence log; A/B on mfu-moe-1b
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 5
- Depends on: 1.3

#### 1.8 GPU leaf hashing and async checkpoint writes

- Files: `src/hypertrain/trainer/compress.py:37-44`, `src/hypertrain/trainer/gpu_hash.py (new)`, `src/hypertrain/protocol/hashing.py`, `tests/trainer/test_gpu_hash.py (new)`
- Changes:
  - Define ht-leaf-v2: fixed 1 MiB chunk tree hash; chunk digests are SHA-256 (or BLAKE3 if D2-approved) computed by a GPU kernel validated against hashlib known-answer vectors, or on host from pinned async D2H copies overlapped with compute; root = sha256 over ordered chunk digests. A non-cryptographic digest is forbidden (commitments must stay collision-resistant). Protocol version bump; v1 kept for old runs.
  - Checkpoint writes moved to a background thread with bounded queue; leaf commit waits only on digest.
- Tests:
  - test_gpu_hash.py: CPU reference implementation equals device implementation on CUDA (gated); known-answer vectors; v1 path unchanged.
- Evidence: Leaf time share in run_mfu.py drops below 2% of step time
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 3
- Depends on: 1.3

#### 1.9 Pinned-memory double-buffered data prefetch

- Files: `src/hypertrain/trainer/loop.py:108-123`, `src/hypertrain/trainer/island.py:509`
- Changes:
  - Prefetcher thread builds next microbatch into pinned host buffer, non_blocking H2D on a side stream with an event; order identical to assignment.
- Tests:
  - tests/trainer/test_trainer_core.py golden digests unchanged; new unit test for prefetch ordering and error propagation.
- Evidence: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded
- Rental: none (local CPU); cost USD 0
- Depends on: 1.3

#### 1.10 Deterministic activation checkpointing

- Files: `src/hypertrain/trainer/model.py:252-268`, `src/hypertrain/trainer/config.py`, `src/hypertrain/protocol/messages_v2.py`
- Changes:
  - Per-layer torch.utils.checkpoint(use_reentrant=False) toggled by reference_spec.env.activation_checkpointing (signed); RNG not used inside layers so recompute is bitwise.
- Tests:
  - CPU: gradients bitwise equal with/without checkpointing on tiny profile.
- Evidence: Memory headroom allows micro_batch >= 8 at seq 2048 for mfu-dense-1b on 32 GB
- Rental: none (local CPU); cost USD 0
- Depends on: 1.6

#### 1.11 torch.compile of fixed-shape fwd/bwd with kernel manifest

- Files: `src/hypertrain/trainer/island.py`, `src/hypertrain/trainer/compile.py (new)`, `src/hypertrain/gpu_ops/network_qualification.py`
- Changes:
  - compile(forward, fullgraph=True, dynamic=False) with inductor deterministic config (torch._inductor.config.deterministic=True or the 2.14 equivalent); hash of generated Triton sources + autotune choices disabled (max_autotune=False) recorded in qualified-kernel manifest; auditors refuse to replay with a different hash.
- Tests:
  - GPU-gated: compiled vs eager leaves differ only if manifest says so; repeated compile on two hosts yields identical source hash.
- Evidence: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded; Two-host identical compiled-source hash
- Rental: Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py); cost USD 8
- Depends on: 1.1, 1.2, 1.6

#### 1.12 MFU qualification shapes and final 40% proof

- Files: `experiments/mfu/profiles.json`, `experiments/results/mfu/final-*.json (evidence)`, `docs/RESULTS.md`, `README.md (performance section)`
- Changes:
  - Run mfu-dense-1b on 1x 5090 and 8x 5090 island (dp=8, zero1) with final engine; 3 independent rentals (different hosts) to show reproducibility; publish median and min.
  - Claim text: 'MFU > 40% (vendor peak 209.5 TFLOPS BF16, PaLM FLOPs convention, deterministic mode, bitwise replay verified)' only if min over 3 hosts > 40%.
- Tests:
  - check_gpu_evidence.py mfu kind passes on every final JSON
- Evidence: 3 hosts x {1 GPU, 8 GPU} JSON; replay-match in each; Vast charges reconciled
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py); cost USD 45
- Depends on: 1.4, 1.5, 1.7, 1.8, 1.9, 1.10, 1.11

#### 1.13 FP8 tier (post-40% stretch, deterministic amax)

- Files: `src/hypertrain/trainer/fp8.py (new)`, `src/hypertrain/trainer/config.py:12`, `src/hypertrain/protocol/messages.py:101`
- Changes:
  - compute_dtype 'fp8-block' with DeepSeek-V3 blockwise scaling (1x128 activations, 128x128 weights) and deterministic amax computed per step with fixed reduction; only enabled after 1.12 passes.
- Tests:
  - CPU emulation of quantize/dequantize bitwise; GPU-gated loss-curve parity vs bf16 over 2k steps within 1%.
- Evidence: A/B MFU vs bf16, loss curve overlay
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 10
- Depends on: 1.12

#### 1.14 Perf regression gate in GPU CI

- Files: `.github/workflows/gpu-vast.yml`, `scripts/vast_e2e.py`, `scripts/vast_e2e_remote.sh`
- Changes:
  - After e2e, run run_mfu.py mfu-dense-0.5b for 60 steps; fail if MFU drops >3 pp below committed experiments/results/mfu/threshold.json; upload JSON as artifact.
- Tests:
  - tests/test_vast_e2e.py: dry-run includes mfu step and threshold comparison logic (fake provider).
- Evidence: Workflow run artifact with MFU JSON
- Rental: none (local CPU); cost USD 0
- Depends on: 1.12

### WS2. Maximum lossless compression of updates (codec v2 container)

Goal: Same lossy quantiser bit-for-bit, plus a canonical lossless container that reaches within 5% of the empirical entropy bound per stream.

Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py) (one Phase-B-shape round to capture real deltas); rest local CPU. Cost ceiling: USD 20.

Exit evidence: ef_out bitwise identical v1 vs v2; sparseloco-v2 >= 4x smaller than v1 on real deltas and within 5% of summed per-stream entropy bound; all service paths accept production-size payloads.

#### 2.1 Entropy-bound analyser and real-delta corpus

- Files: `experiments/compression/entropy_bound.py (new)`, `experiments/compression/capture_deltas.py (new)`
- Changes:
  - Capture SparseLoCo + dense-int8 payloads from 3 consecutive real rounds (phase_b profile and mfu-dense-1b) on 8x 5090; store under evidence dir (not git).
  - Compute per-stream empirical entropies: index gaps (order-0 and Elias-Fano bound log2(n/k)+2), 2-bit codes, int8 values (order-0 per tensor), scales (byte planes).
- Tests:
  - tests/test_compression_scripts.py (new): synthetic Student-t corpus reproduces audit 5.2 sizes within 2%.
- Evidence: bounds JSON per stream, per tensor class
- Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py); cost USD 10
- Depends on: 0.4

#### 2.2 Codec v2 container: framing and canonical form

- Files: `src/hypertrain/trainer/codec_v2.py (new)`, `src/hypertrain/trainer/compress.py:1-30,160-285`, `src/hypertrain/trainer/config.py:15`, `src/hypertrain/protocol/messages.py:169-175`, `docs/schemas/DeltaManifest.json`, `docs/protocol.md`
- Changes:
  - New MAGIC values 'sparseloco-v2' and 'dense-int8-v2'; payload = header + per-tensor frames; each frame holds separate streams (indices, codes, values, scales) each with stage id + length.
  - v2 is a lossless transform of the exact v1 payload: compress() still produces the v1 bytes (ef_out bitwise identical by construction) and encode_v2(v1) splits them into streams; decode_v2(v2) must return the identical v1 bytes, which then pass the existing v1 validation.
  - Validation decodes, re-encodes, and rejects if bytes differ (canonical-encoding check) before any tensor allocation beyond declared sizes; per-frame streaming decode (no b''.join of the whole payload).
- Tests:
  - tests/trainer/test_codec_v2.py (new): ef_out v1==v2 bitwise; round-trip; non-canonical encodings rejected; truncated/oversized frames rejected; declared-size bomb rejected before allocation.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none (2.1 real-delta corpus gates only the ratio claims in 2.8)

#### 2.3 Elias-Fano index stage

- Files: `src/hypertrain/trainer/codec_v2.py`, `tests/trainer/test_codec_v2.py`
- Changes:
  - Vectorised numpy Elias-Fano for strictly ascending u32 indices: low bits l=floor(log2(n/k)), packed low array + unary high array; decode with np.cumsum/np.flatnonzero; canonical by construction.
- Tests:
  - Property test (hypothesis, T1): random (n,k) incl. k=1, k=n, n=2^32-1 boundary; size == EF formula exactly.
- Evidence: Index stream bits/index within 2% of log2(n/k)+2 on real corpus
- Rental: none (local CPU); cost USD 0
- Depends on: 2.2

#### 2.4 Entropy back-end for codes, values and byte-shuffled scales

- Files: `src/hypertrain/trainer/codec_v2.py`, `pyproject.toml`, `uv.lock`, `docker/gpu-requirements.in`
- Changes:
  - Back-end per decision D2: default zstandard pinned (exact version, level 19, no dictionary, single-threaded, content-size flag on, checksum off); canonical form enforced by re-encode compare; if D2 refuses the new dependency, stdlib lzma (FORMAT_RAW, FILTER_LZMA2 preset 9, fixed dict size) is used instead.
  - Scales: split into 4 byte planes then back-end; int8 values: per-tensor back-end; 2-bit codes: back-end on packed bytes.
- Tests:
  - test_codec_v2.py: back-end determinism across two processes; version pin asserted at import (refuse if mismatch).
- Evidence: Per-stream ratio vs entropy bound table
- Rental: none (local CPU); cost USD 0
- Depends on: 2.3

#### 2.5 rANS ablation for value/code streams (keep only if it wins)

- Files: `experiments/compression/rans_ablation.py (new)`
- Changes:
  - Static order-0 rANS with per-tensor 12-bit frequency table (numpy reference); compare vs back-end on corpus. Adopt into codec_v2 only if it saves >=3% total payload; otherwise record negative result.
- Tests:
  - Ablation script test on synthetic corpus.
- Evidence: ablation JSON + decision note in docs/protocol.md
- Rental: none (local CPU); cost USD 0
- Depends on: 2.4

#### 2.6 Wire v2 into miner, aggregator, auditor, capacity worker and service

- Files: `src/hypertrain/miner/core.py`, `src/hypertrain/aggregator/core.py`, `src/hypertrain/aggregator/capacity_worker.py:431-436`, `src/hypertrain/auditor/replay.py`, `src/hypertrain/challenge/store.py:7454-7466`, `src/hypertrain/trainer/config.py:CODECS`
- Changes:
  - CODECS gains v2 names; manifest outer.codec_version selects; all validators call one validate_payload that dispatches by MAGIC; max_payload_bytes = profile.max_object_bytes everywhere (closes audit 2.1.1).
- Tests:
  - tests/challenge, tests/aggregator, tests/auditor suites green; new size-regression test T3 (10M-param dense-int8 v1 and v2 delta accepted).
- Evidence: Full pytest log; e2e test_service_network_v2_e2e green with v2
- Rental: none (local CPU); cost USD 0
- Depends on: 2.4, 4.1

#### 2.7 Checkpoint / state-transfer lossless path (ZipNN-style + XOR delta)

- Files: `src/hypertrain/aggregator/checkpoint.py`, `src/hypertrain/trainer/codec_v2.py`
- Changes:
  - Byte-group bf16/fp32 tensors (exponent plane separate), back-end compress; optional XOR against previous round's checkpoint for state streams used by late joiners (WS6.3).
- Tests:
  - Round-trip bitwise; ratio report on real checkpoint.
- Evidence: Ratio table (expect ~1.5x bf16, ~1.17x fp32 per ZipNN; measured)
- Rental: none (local CPU); cost USD 0
- Depends on: 2.4

#### 2.8 Compression ablation report

- Files: `docs/RESULTS.md`, `experiments/results/compression/*.json`
- Changes:
  - Codec x density {1%,1.56%,3.125%} x EF beta table: bytes/round raw vs v2, ratio vs entropy bound, encode/decode MB/s on CPU node and on 5090 host CPU.
- Tests:
  - n/a
- Evidence: Report JSON checked by scripts/check_doc_numbers.py tags
- Rental: reuse 2.1 corpus; cost USD 0
- Depends on: 2.5, 2.6, 2.7

#### 2.9 GPU-side codec feasibility (nvCOMP ANS / DietGPU)

- Files: `experiments/compression/gpu_codec_bench.py (new)`
- Changes:
  - Benchmark encode throughput; adopt only if CPU encode time > 5% of round wall time at 1B params (measured in 2.8).
- Tests:
  - n/a
- Evidence: Bench JSON + adopt/reject note
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 5
- Depends on: 2.8

### WS3. Connection and transport

Goal: Uploads are parallel, resumable and size-aware; relays stop double-storing objects.

Rental: local kind cluster (deploy/k8s/local) + 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3) for cross-region throughput test. Cost ceiling: USD 15.

Exit evidence: Resume test passes; Relay egress counters show 1x not 2x; Region-pair throughput JSON.

#### 3.1 Parallel resumable chunk upload with retries

- Files: `src/hypertrain/relay/client.py:159-166`, `tests/relay/test_transport_v2.py`
- Changes:
  - anyio task group with CapacityLimiter(4) (matching relay/core.py:156); before upload GET .../acks and skip acknowledged chunks; per-chunk retry with jittered exponential backoff (max 5); HTTP/2 enabled on httpx client (h2 extra pinned) ; failover across fallback_ids.
- Tests:
  - test_transport_v2.py: kill mid-upload then resume uploads only missing chunks; out-of-order acks validated; retry exhaustion surfaces error (no silent success).
- Evidence: pytest log; kind smoke (scripts/relay_kind_smoke.py) pass log
- Rental: none (local CPU); cost USD 0
- Depends on: 4.1

#### 3.2 Size-scaled transfer deadline

- Files: `src/hypertrain/data/stream_store.py:48`, `src/hypertrain/data/store.py`
- Changes:
  - Deadline = 30 s + size / min_rate where min_rate is a signed profile value (default 25 MB/s), plus idle timeout 30 s between chunks.
- Tests:
  - Unit tests for deadline formula and idle timeout with an anyio fake stream.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 3.3 Relay stores chunks + signed chunk manifest (no reassembly)

- Files: `src/hypertrain/relay/core.py:395-420`, `src/hypertrain/relay/app.py`, `src/hypertrain/protocol/relay_messages.py`, `src/hypertrain/data/stream_store.py`
- Changes:
  - Completion signs a ChunkManifest (chunk sha256 list + object sha256 computed streaming); consumers fetch chunks and verify; whole-object put removed.
- Tests:
  - test_lifecycle_v2.py: storage and egress counters halve; consumer rejects swapped chunk.
- Evidence: pytest log; kind smoke log
- Rental: none (local CPU); cost USD 0
- Depends on: 3.1

#### 3.4 R2 multipart upload

- Files: `src/hypertrain/storage/r2.py:137-145`, `tests/storage/test_r2.py`
- Changes:
  - S3 multipart (64 MiB parts) when len > 5 GiB or stream input; immutable semantics preserved via sha256 metadata check.
- Tests:
  - Fake S3 server test for multipart init/part/complete/abort.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 3.5 Cross-region throughput and chaos test (T7)

- Files: `scripts/net_chaos.py (new)`, `tests/e2e/test_chaos.py (new, slow)`
- Changes:
  - netem latency/loss/partition between miner, relay and service in kind; relay pod kill mid-upload; then real 3-region CPU nodes measuring MB/s for a 1 GiB object.
- Tests:
  - test_chaos.py in tests/e2e (CI test-slow).
- Evidence: Throughput JSON per region pair; chaos test log
- Rental: 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3); cost USD 10
- Depends on: 3.1, 3.2, 3.3

### WS4. Production-size limits and aggregator scale

Goal: Every hard cap that blocks real model sizes is removed or made a signed profile value; aggregation is O(d) vectorised and deterministic.

Rental: local CPU; 1 CPU node for 1B-param aggregation timing. Cost ceiling: USD 5.

Exit evidence: T3 passes; 1B-element norm < 3 s; roster at profile cap works.

#### 4.1 Replace 64 KiB caps with profile.max_object_bytes

- Files: `src/hypertrain/trainer/compress.py:265`, `src/hypertrain/challenge/store.py:7454,7463`, `src/hypertrain/aggregator/capacity_worker.py:431-436`, `tests/challenge/test_service_network_v2.py`, `tests/aggregator/test_hardening.py`
- Changes:
  - validate_payload requires explicit max_payload_bytes (no default); callers pass profile.max_object_bytes as store.py:5921-5925 does; bounded_bytes/_read_at use the same value.
- Tests:
  - T3 size regression: 10M-param dense-int8 delta accepted; profile.max_object_bytes+1 rejected.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 4.2 Deterministic vectorised sqnorm/dot

- Files: `src/hypertrain/aggregator/core.py:104-117`, `docs/protocol.md`, `tests/aggregator/test_norms.py (new)`
- Changes:
  - Spec ht-norm-v2: per tensor, float32 values squared (or multiplied) in float64 (exact), zero-padded to a power of two, then reduced by a fixed pairwise tree of elementwise adds (x = x[0::2] + x[1::2] until length 1). Elementwise IEEE adds are correctly rounded, so the result is bitwise independent of numpy SIMD dispatch and thread count (np.sum/np.add.reduce are NOT used because their internal blocking is build-dependent). Tensor results combined with math.fsum in tensor-name order. v1 retained for old runs.
- Tests:
  - test_norms.py: equals math.fsum reference within 1 ulp-of-total bound and is bitwise stable across 3 runs and across numpy thread settings; 1B-element timing < 3 s on CPU node.
- Evidence: Timing JSON on CPU node; pytest log
- Rental: 1 CPU node (16 vCPU); cost USD 1
- Depends on: none

#### 4.3 Roster caps to signed profile values

- Files: `src/hypertrain/challenge/store.py:4378-4395`, `src/hypertrain/protocol/messages_v2.py`
- Changes:
  - Replace 16/32 literals with profile.max_roster (signed, default 32, ceiling 1024); replace assert at 4380 with explicit ChallengeError.
- Tests:
  - Tests at max_roster and max_roster+1; python -O run of the guard test.
- Evidence: pytest log incl. -O run
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 4.4 Service store concurrency (split store.py, WAL, reader pool)

- Files: `src/hypertrain/challenge/store.py (split into store/admission.py, rounds.py, disputes.py, weights.py, objects.py)`, `src/hypertrain/challenge/app.py`, `src/hypertrain/challenge/store.py:364-370,425`
- Changes:
  - Mechanical split first (no behaviour change, imports re-exported from store.py), then WAL + read-only connection pool for GETs; /health no longer writes under global lock.
- Tests:
  - Full tests/challenge suite unchanged; T8 soak: 32-miner x 100 rounds write latency p99 recorded.
- Evidence: pytest log; soak JSON
- Rental: none (local CPU); cost USD 0
- Depends on: 4.3

### WS5. Trust: auditor quorum and single-party fixes

Goal: No single party can mint reward, forgive a fault or exhaust the public API.

Rental: local CPU. Cost ceiling: USD 0.

Exit evidence: All trust tests green; mechanism.md documents quorum.

#### 5.1 k-of-n auditor quorum per reward item with distinct coldkeys

- Files: `src/hypertrain/ledger/escrow_v2.py:395-402`, `src/hypertrain/protocol/messages_v2.py`, `src/hypertrain/challenge/trust_v2.py`, `src/hypertrain/auditor/worker.py`, `docs/mechanism.md`, `tests/ledger/test_escrow_quorum.py (new)`
- Changes:
  - Run manifest gains auditor_quorum {k, n} (signed; default k=2); reward item carries k ReplayVerdicts from distinct auditor coldkeys all MATCH; any MISMATCH escalates to dispute; auditor assignment drawn from drand beacon.
- Tests:
  - test_escrow_quorum.py: k-1 verdicts rejected; same coldkey twice rejected; mixed MATCH/MISMATCH escalates; property test of escrow conservation (T9).
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 5.2 Referee-confirmed TRANSIENT

- Files: `src/hypertrain/auditor/replay.py:842-860`, `src/hypertrain/challenge/disputes_v2.py`
- Changes:
  - TRANSIENT requires an independent referee rerun matching the auditor root; miner rerun alone yields CONTEST.
- Tests:
  - Unit tests for each branch of classification table.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 5.3 Audit window beyond anchor age 1 and rollback beyond w+2

- Files: `src/hypertrain/auditor/replay.py:394-395`, `src/hypertrain/aggregator/core.py:846-847`, `src/hypertrain/aggregator/rollback_v2.py`
- Changes:
  - Auditor replays from nearest retained checkpoint (checkpoint lineage, retention signed in profile); rollback via lineage replay excising the faulty delta and re-aggregating forward.
- Tests:
  - Tests: anchor age 3 audit succeeds; rollback at w+5 reproduces the honest-only state bitwise (emulator).
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: 2.7

#### 5.4 Escrow remainder rotation and total==0 guard

- Files: `src/hypertrain/ledger/escrow_v2.py:445-456`
- Changes:
  - Remainder order rotated by beacon-derived offset; explicit EscrowError on total==0.
- Tests:
  - Property tests (T9) conservation and fairness over beacons.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 5.5 Bounded public API cache and drand latest() fix

- Files: `src/hypertrain/public_api/__init__.py:49-66`, `src/hypertrain/beacon/drand.py:150-170`
- Changes:
  - Cache only known route templates via functools.lru_cache-like OrderedDict max 1024; unknown paths bypass cache. drand: non-int round raises BeaconError.
- Tests:
  - Random-path flood test keeps cache size <= 1024; drand malformed body test.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 5.6 Envelope expiry

- Files: `src/hypertrain/miner/core.py:51`, `src/hypertrain/protocol/envelope_v2.py`
- Changes:
  - Default expiry = 4 beacon rounds; Intake rejects expired envelopes.
- Tests:
  - Expired/near-boundary tests.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

### WS6. Complete network: multi-region aggregators and elastic membership

Goal: Three regional aggregators (CPU) compute identical aggregates and agree by quorum; miners join and leave mid-run with verified state sync.

Rental: 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3); plus Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py) x2 hosts as miners for the live run. Cost ceiling: USD 75.

Exit evidence: Live run JSON with 20 finalised rounds, churn, aggregator failover.

#### 6.1 Regional aggregator role with cross-region hash agreement

- Files: `src/hypertrain/aggregator/region.py (new)`, `src/hypertrain/aggregator/cli.py`, `src/hypertrain/protocol/messages_v2.py`, `deploy/k8s/overlays/{eu,us,apac}/kustomization.yaml`, `deploy/k8s/base/aggregator-deployment.yaml (new)`, `docs/network-v2.md`
- Changes:
  - `hypertrain aggregate-region` subcommand: pulls round deltas from nearest relay, runs aggregate-v2 (deterministic), signs AggregateCommit{round, root}; service finalises when >= 2 of 3 regional commits agree; disagreement opens dispute with bisect over tensor list.
- Tests:
  - tests/aggregator/test_region.py: 3 in-process regions agree; one corrupted region is outvoted and flagged.
- Evidence: pytest log; kustomize render of 3 overlays
- Rental: none (local CPU); cost USD 0
- Depends on: 4.2, 3.3

#### 6.2 Elastic membership: join/leave per round, roster from admissions

- Files: `src/hypertrain/challenge/admission.py`, `src/hypertrain/challenge/membership.py (new)`, `src/hypertrain/challenge/store.py (rounds module after 4.4)`
- Changes:
  - Roster for round w+1 = admitted set at beacon cut-off minus leavers; churn bounded per round (signed max_churn); departures are graceful (no slash) when announced before cut-off.
- Tests:
  - Membership state-machine tests incl. churn limit and rejoin.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: 4.3, 4.4

#### 6.3 Verified state sync for late joiners

- Files: `src/hypertrain/miner/core.py`, `src/hypertrain/aggregator/checkpoint.py`, `src/hypertrain/protocol/messages_v2.py (StateServe)`
- Changes:
  - Joiner downloads latest finalised checkpoint (compressed via 2.7) + optimizer state per state_policy; verifies root against finalised AggregateCommit before training.
- Tests:
  - e2e: miner joins at round 3, its round-3 leaves match a replay.
- Evidence: e2e log
- Rental: none (local CPU); cost USD 0
- Depends on: 6.2, 2.7

#### 6.4 Layout-invariant checkpoints (islands with different GPU counts)

- Files: `src/hypertrain/trainer/island.py:16`, `src/hypertrain/aggregator/checkpoint.py`
- Changes:
  - Checkpoint format is full unsharded tensors in sorted order (already true for leaves); loader reshards for any (dp, ep) layout; round deltas are layout-independent; replay still uses the miner's declared layout.
- Tests:
  - Emulator test: save from dp=4 ep=2, load into dp=1 ep=1 and dp=2 ep=4, leaves equal reference.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: 6.3

#### 6.5 Metrics, tracing and rate limits

- Files: `src/hypertrain/challenge/app.py:810-816`, `src/hypertrain/relay/app.py`, `src/hypertrain/public_api/__init__.py`
- Changes:
  - /metrics (Prometheus text, stdlib formatting), structured logging in challenge, per-IP token bucket on all routes.
- Tests:
  - Route tests for /metrics and 429 responses.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 6.6 Live multi-region run

- Files: `experiments/network_live/run.py (new)`, `experiments/results/network_live/*.json`
- Changes:
  - 3 regional CPU aggregators + 2 GPU miner hosts (2x5090 each) + 1 CPU auditor pair; 20 rounds; one miner joins at round 5, one leaves at round 12; one aggregator killed at round 15.
- Tests:
  - n/a (evidence)
- Evidence: Per-round finalised roots agreed by >= 2 regions; join/leave events in journal; auditor quorum verdicts; Vast + CPU charges reconciled
- Rental: 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3); 2 x Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py); cost USD 60
- Depends on: 6.1, 6.2, 6.3, 6.4, 6.5, 5.1, 2.6

### WS7. Model registry and every model family

Goal: One arch registry; each family has shapes/init/forward/objective, kernel manifest, op-trace hooks, CPU bitwise tests and a 5090 smoke run.

Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand (1 smoke run per family). Cost ceiling: USD 90.

Exit evidence: Per-family: CPU bitwise test + 5090 smoke JSON (loss down, replay bitwise).

#### 7.1 Arch registry replacing decoder-vs-OpenDecision branch

- Files: `src/hypertrain/models/registry.py (new)`, `src/hypertrain/trainer/model.py:24-90,232-274`, `src/hypertrain/protocol/messages.py:86-111`, `tests/trainer/test_arch_dispatch.py`
- Changes:
  - ArchSpec protocol (param_shapes, init_params, stage_of, forward, objective, sample_spec, kernel_set); REGISTRY dict keyed by protocol arch literal; 'decoder' and 'od-encoder' registered first with zero behaviour change.
- Tests:
  - test_arch_dispatch.py: unknown arch rejected at manifest validation; decoder/od goldens unchanged.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: 1.6

#### 7.2 Decoder variants: MoE shared experts, MLA

- Files: `src/hypertrain/models/decoder_mla.py (new)`, `src/hypertrain/trainer/model.py`
- Changes:
  - Shared-expert MoE option; MLA (low-rank KV compression with decoupled RoPE) as arch 'decoder-mla'.
- Tests:
  - CPU fp32 bitwise determinism (2 runs), shape tests, MoE shared-expert gradient check.
- Evidence: 5090 smoke: 200 steps, loss decreases, replay bitwise
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 6
- Depends on: 7.1, 1.7

#### 7.3 Encoder (MLM) family

- Files: `src/hypertrain/models/encoder.py (new)`
- Changes:
  - Bidirectional attention, deterministic masking from rng.py Philox stream keyed by (round, sample).
- Tests:
  - CPU bitwise; masking determinism test.
- Evidence: 5090 smoke
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 6
- Depends on: 7.1, 8.1

#### 7.4 Encoder-decoder family

- Files: `src/hypertrain/models/encdec.py (new)`
- Changes:
  - Cross-attention via attention.py; span-corruption objective.
- Tests:
  - CPU bitwise.
- Evidence: 5090 smoke
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 6
- Depends on: 7.3

#### 7.5 Vision (ViT) family

- Files: `src/hypertrain/models/vit.py (new)`
- Changes:
  - Patch embed as matmul over fixed patch order; classification and contrastive objectives.
- Tests:
  - CPU bitwise.
- Evidence: 5090 smoke
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 6
- Depends on: 7.1, 8.1

#### 7.6 Multimodal (vision tower + projector + LM)

- Files: `src/hypertrain/models/multimodal.py (new)`
- Changes:
  - Image tokens spliced at sidecar positions; loss mask over text only.
- Tests:
  - CPU bitwise.
- Evidence: 5090 smoke
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 8
- Depends on: 7.5, 8.1

#### 7.7 Diffusion / flow matching

- Files: `src/hypertrain/models/diffusion.py (new)`, `src/hypertrain/trainer/rng.py`
- Changes:
  - DiT backbone, timestep conditioning, noise drawn from rng.py counter stream (round, sample, step) so replay is bitwise.
- Tests:
  - CPU bitwise; noise stream determinism.
- Evidence: 5090 smoke
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 8
- Depends on: 7.5

#### 7.8 SSM / hybrid

- Files: `src/hypertrain/models/ssm.py (new)`
- Changes:
  - Mamba-2 style chunked scan with fixed chunk size and fixed inter-chunk combine order (pure torch reference; Triton kernel only if bitwise).
- Tests:
  - CPU bitwise; chunked vs sequential reference within tolerance.
- Evidence: 5090 smoke
- Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; cost USD 8
- Depends on: 7.1

#### 7.9 Long context: RoPE scaling, sliding window, context parallel

- Files: `src/hypertrain/trainer/attention.py`, `src/hypertrain/trainer/island.py`, `src/hypertrain/models/registry.py`
- Changes:
  - YaRN/NTK rope scaling, sliding window in attention.py; context parallelism (ring attention with fixed ring order) as layout field cp_size.
- Tests:
  - CPU emulator: cp=2/4 equals cp=1 within fixed-order digest.
- Evidence: 5090 smoke at seq 32k on 8 GPUs
- Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py); cost USD 12
- Depends on: 7.1, 1.4, 8.1

#### 7.10 Objectives: loss mask CE, distillation, DPO, GRPO

- Files: `src/hypertrain/trainer/objectives.py (new)`, `src/hypertrain/teacher/adapters.py:78-92`
- Changes:
  - Objectives registry; distillation from cached teacher logits; DPO pairwise; GRPO with verified rollouts (rollout tokens committed and replayed); fix tulu-3-sft adapter.
- Tests:
  - Objective unit tests vs hand-computed values; adapter test.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: 7.1, 8.1

#### 7.11 Tensor and pipeline parallelism with fixed order

- Files: `src/hypertrain/trainer/island.py`, `src/hypertrain/trainer/collectives.py`, `src/hypertrain/trainer/loop.py:90-102`
- Changes:
  - TP (column/row split with fixed-order ring reduce) and PP 1F1B with fixed microbatch schedule; layout fields tp, pp executed (not just leaf partition).
- Tests:
  - Emulator: tp=2, pp=2 leaves equal reference digest for that layout; layout invariance via 6.4 checkpoints.
- Evidence: 8x 5090 run for 7B-class shape fits and replays
- Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py); cost USD 10
- Depends on: 1.4, 6.4

### WS8. Data format v3

Goal: Shards carry any vocab, variable-length records, masks and modality blobs.

Rental: local CPU. Cost ceiling: USD 0.

Exit evidence: v3 shards with u32 vocab train end-to-end in tiny profile.

#### 8.1 Shard format v3 (dtype field, varlen records, sidecars)

- Files: `src/hypertrain/datasets/build.py:68,98-108`, `src/hypertrain/datasets/shards16.py`, `src/hypertrain/data/shards.py:50-60`, `src/hypertrain/trainer/loop.py:108-123`, `src/hypertrain/datasets/fetch.py`
- Changes:
  - Manifest field token_dtype (u16/u32); record index + sidecars (doc_boundaries, loss_mask, segment_ids, blobs with sha256); unit_ranges width from manifest; document masking in packing (closes shards.py:57 ponytail); streaming parquet reader (pyarrow iter_batches).
- Tests:
  - Round-trip tests; vocab 128k build; doc-mask attention equivalence test.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 8.2 Train-train dedup and fix teacher label/build key mismatch

- Files: `src/hypertrain/datasets/build.py:107-114`, `src/hypertrain/datasets/decontam.py`
- Changes:
  - MinHash LSH train-train dedup (existing decontam MinHash reused); label lookup keyed on the same _eval_text; limit applied before decontam ordering consistently.
- Tests:
  - Test with --limit selects labelled rows.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

### WS9. Docs correctness and CI gates

Goal: Docs numbers are machine-checked and every audit test category runs in CI.

Rental: GitHub-hosted runners; GPU via gpu-vast.yml. Cost ceiling: USD 90.

Exit evidence: CI green with new gates; docs checker exit 0.

#### 9.1 Fix doc inconsistencies

- Files: `docs/parity.md:33,83-86`, `docs/operator.md:151`
- Changes:
  - 8 Gbit/s and 0.31 Gbit/s corrections; Phase B state_policy carry; remove or regenerate summary_smoke.json reference.
- Tests:
  - scripts/check_doc_numbers.py passes; test_check_doc_numbers.py extended for missing-file failure.
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 9.2 Property/fuzz tests T1, T2, T9 + hypothesis dev dep

- Files: `pyproject.toml`, `uv.lock`, `tests/protocol/test_jcs_fuzz.py (new)`, `tests/trainer/test_compress_fuzz.py (new)`, `tests/ledger/test_escrow_props.py (new)`
- Changes:
  - Add hypothesis to dev group (decision D2 covers dev deps).
- Tests:
  - new tests
- Evidence: pytest log
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 9.3 Supply chain T11 and gitleaks in CI

- Files: `.github/workflows/ci.yml`, `docker/Dockerfile.gpu`
- Changes:
  - pip-audit on uv.lock; install pinned gitleaks so tests/test_publish_exclusions.py:25 runs; SBOM via syft for both images; hash-pin opendecision tarball.
- Tests:
  - CI run
- Evidence: CI logs
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 9.4 Cross-hardware determinism matrix T4

- Files: `experiments/xhw/run.py (new)`
- Changes:
  - Same round replayed on 5090 (driver N, N-1) and one other qualified GPU class; outputs qualified-set table.
- Tests:
  - n/a
- Evidence: matrix JSON
- Rental: Vast.ai 1 GPU each: RTX 5090 x2 drivers, H100, L40S; cost USD 20
- Depends on: 1.12

#### 9.5 Adversarial e2e T6 on GPU

- Files: `tests/e2e/test_adversarial_gpu.py (new)`
- Changes:
  - copy, scaled, sign-flip, low-rate poisoning, Sybil miners vs CenteredClip + quorum; catch rate and false positives vs honeypots.
- Tests:
  - gated GPU test
- Evidence: catch-rate JSON
- Rental: Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py); cost USD 10
- Depends on: 6.6

#### 9.6 Mutation testing T10 and direct unit tests T13

- Files: `tests/gpu_ops/test_budget.py (new)`, `tests/gpu_ops/test_verdict.py (new)`, `tests/miner/test_island_worker.py (new)`
- Changes:
  - mutmut over protocol/, ledger/, auditor/replay.py (report only, survivors become issues).
- Tests:
  - new tests
- Evidence: mutmut report
- Rental: none (local CPU); cost USD 0
- Depends on: none

#### 9.7 Parity grid >= 8 seeds (T14) on CPU nodes

- Files: `experiments/sim/parity_grid.py`, `experiments/results/parity_runs.jsonl`
- Changes:
  - Run pre-registered TOST grid with 8 seeds on rented CPU nodes; no promotion claim unless TOST passes.
- Tests:
  - n/a
- Evidence: parity results + TOST verdict
- Rental: 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3); cost USD 40
- Depends on: none

## Todos

Ordered backlog (topological; ties broken by workstream priority WS0, WS4, WS1, WS2, WS5, WS3, WS8, WS7, WS6, WS9). Each row's details are in its workstream section.

- [ ] 1. [WS0 0.1] FLOPs/token model and MFU calculator
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log for tests/trainer/test_perf.py exit 0; reproduction of audit section 4.1 table within 0.01 pp.
  QA: run the tests listed in 0.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws0: FLOPs/token model and MFU calculator`.
- [ ] 2. [WS0 0.2] Achievable-peak microbenchmark
  Recommended task executor category: deep-low
  Depends on: 1. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 3.
  Acceptance: experiments/results/mfu/peak-<host>.json committed; Measured peak reported next to 209.5 TFLOPS vendor peak.
  QA: run the tests listed in 0.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws0: Achievable-peak microbenchmark`.
- [ ] 3. [WS0 0.3] Reproducible MFU benchmark runner
  Recommended task executor category: deep-low
  Depends on: 1, 2. Rental: none (local CPU). Cost: USD 0.
  Acceptance: experiments/results/mfu/<profile>-<sha>.json schema-valid; check_gpu_evidence.py exit 0 on it.
  QA: run the tests listed in 0.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws0: Reproducible MFU benchmark runner`.
- [ ] 4. [WS0 0.4] Baseline MFU on rented 5090 before any engine change
  Recommended task executor category: unspecified-low
  Depends on: 3. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 5.
  Acceptance: Baseline JSON + replay match + Vast journal (instance id, dph, total charge) under the job evidence dir; Charge <= ceiling read back from Vast GET.
  QA: run the tests listed in 0.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws0: Baseline MFU on rented 5090 before any engine change`.
- [ ] 5. [WS4 4.1] Replace 64 KiB caps with profile.max_object_bytes
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 4.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws4: Replace 64 KiB caps with profile.max_object_bytes`.
- [ ] 6. [WS4 4.2] Deterministic vectorised sqnorm/dot
  Recommended task executor category: deep-low
  Depends on: none. Rental: 1 CPU node (16 vCPU). Cost: USD 1.
  Acceptance: Timing JSON on CPU node; pytest log.
  QA: run the tests listed in 4.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws4: Deterministic vectorised sqnorm/dot`.
- [ ] 7. [WS4 4.3] Roster caps to signed profile values
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log incl. -O run.
  QA: run the tests listed in 4.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws4: Roster caps to signed profile values`.
- [ ] 8. [WS4 4.4] Service store concurrency (split store.py, WAL, reader pool)
  Recommended task executor category: deep-high
  Depends on: 7. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log; soak JSON.
  QA: run the tests listed in 4.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws4: Service store concurrency (split store.py, WAL, reader pool)`.
- [ ] 9. [WS1 1.1] Fused deterministic causal attention with native GQA
  Recommended task executor category: deep-high
  Depends on: 4. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 10.
  Acceptance: Spike log with backend chosen + bitwise result; A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded; Cross-host replay identical leaves (2 hosts).
  QA: run the tests listed in 1.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Fused deterministic causal attention with native GQA`.
- [ ] 10. [WS1 1.2] Pinned tensor-core matmul policy (TF32/bf16 reductions under fixed kernels)
  Recommended task executor category: deep-high
  Depends on: 9. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 6.
  Acceptance: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded; Bitwise replay under pinned-tc on 2 hosts.
  QA: run the tests listed in 1.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Pinned tensor-core matmul policy (TF32/bf16 reductions under`.
- [ ] 11. [WS1 1.3] Kill per-step host syncs
  Recommended task executor category: deep-low
  Depends on: 4. Rental: none (local CPU). Cost: USD 0.
  Acceptance: Golden-digest equality on CPU; A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded.
  QA: run the tests listed in 1.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Kill per-step host syncs`.
- [ ] 12. [WS1 1.4] Deterministic bucketed reduce-scatter overlapped with backward
  Recommended task executor category: deep-high
  Depends on: 11. Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py). Cost: USD 20.
  Acceptance: Emulator vs 8-GPU leaves bitwise; A/B on 8x 5090: tok/s and MFU; Cross-host bitwise (2 x 8 GPU optional, else 2 x 2 GPU).
  QA: run the tests listed in 1.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Deterministic bucketed reduce-scatter overlapped with backwa`.
- [ ] 13. [WS1 1.5] Multi-tensor fused optimizer (AdamW + Muon batched NS)
  Recommended task executor category: deep-low
  Depends on: 11. Rental: none (local CPU). Cost: USD 0.
  Acceptance: CPU bitwise equality log; A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded.
  QA: run the tests listed in 1.5, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Multi-tensor fused optimizer (AdamW + Muon batched NS)`.
- [ ] 14. [WS1 1.6] bf16 weight shadow refreshed once per optimizer step
  Recommended task executor category: deep-low
  Depends on: 13. Rental: none (local CPU). Cost: USD 0.
  Acceptance: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded.
  QA: run the tests listed in 1.6, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: bf16 weight shadow refreshed once per optimizer step`.
- [ ] 15. [WS1 1.7] MoE: grouped GEMM, drop skipped tokens, no k-fold repeat, bf16 all_to_all
  Recommended task executor category: deep-low
  Depends on: 11. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 5.
  Acceptance: CPU equivalence log; A/B on mfu-moe-1b.
  QA: run the tests listed in 1.7, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: MoE: grouped GEMM, drop skipped tokens, no k-fold repeat, bf`.
- [ ] 16. [WS1 1.8] GPU leaf hashing and async checkpoint writes
  Recommended task executor category: deep-high
  Depends on: 11. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 3.
  Acceptance: Leaf time share in run_mfu.py drops below 2% of step time.
  QA: run the tests listed in 1.8, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: GPU leaf hashing and async checkpoint writes`.
- [ ] 17. [WS1 1.9] Pinned-memory double-buffered data prefetch
  Recommended task executor category: deep-low
  Depends on: 11. Rental: none (local CPU). Cost: USD 0.
  Acceptance: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded.
  QA: run the tests listed in 1.9, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Pinned-memory double-buffered data prefetch`.
- [ ] 18. [WS1 1.10] Deterministic activation checkpointing
  Recommended task executor category: deep-low
  Depends on: 14. Rental: none (local CPU). Cost: USD 0.
  Acceptance: Memory headroom allows micro_batch >= 8 at seq 2048 for mfu-dense-1b on 32 GB.
  QA: run the tests listed in 1.10, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Deterministic activation checkpointing`.
- [ ] 19. [WS1 1.11] torch.compile of fixed-shape fwd/bwd with kernel manifest
  Recommended task executor category: deep-high
  Depends on: 9, 10, 14. Rental: Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py). Cost: USD 8.
  Acceptance: A/B on 1x 5090 with run_mfu.py, deterministic + replay; tok/s gain recorded; Two-host identical compiled-source hash.
  QA: run the tests listed in 1.11, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: torch.compile of fixed-shape fwd/bwd with kernel manifest`.
- [ ] 20. [WS1 1.12] MFU qualification shapes and final 40% proof
  Recommended task executor category: unspecified-low
  Depends on: 12, 13, 15, 16, 17, 18, 19. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand; Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py). Cost: USD 45.
  Acceptance: 3 hosts x {1 GPU, 8 GPU} JSON; replay-match in each; Vast charges reconciled.
  QA: run the tests listed in 1.12, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: MFU qualification shapes and final 40% proof`.
- [ ] 21. [WS1 1.13] FP8 tier (post-40% stretch, deterministic amax)
  Recommended task executor category: deep-high
  Depends on: 20. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 10.
  Acceptance: A/B MFU vs bf16, loss curve overlay.
  QA: run the tests listed in 1.13, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: FP8 tier (post-40% stretch, deterministic amax)`.
- [ ] 22. [WS1 1.14] Perf regression gate in GPU CI
  Recommended task executor category: unspecified-low
  Depends on: 20. Rental: none (local CPU). Cost: USD 0.
  Acceptance: Workflow run artifact with MFU JSON.
  QA: run the tests listed in 1.14, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws1: Perf regression gate in GPU CI`.
- [ ] 23. [WS2 2.1] Entropy-bound analyser and real-delta corpus
  Recommended task executor category: deep-low
  Depends on: 4. Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py). Cost: USD 10.
  Acceptance: bounds JSON per stream, per tensor class.
  QA: run the tests listed in 2.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: Entropy-bound analyser and real-delta corpus`.
- [ ] 24. [WS2 2.2] Codec v2 container: framing and canonical form
  Recommended task executor category: deep-high
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 2.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: Codec v2 container: framing and canonical form`.
- [ ] 25. [WS2 2.3] Elias-Fano index stage
  Recommended task executor category: deep-low
  Depends on: 24. Rental: none (local CPU). Cost: USD 0.
  Acceptance: Index stream bits/index within 2% of log2(n/k)+2 on real corpus.
  QA: run the tests listed in 2.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: Elias-Fano index stage`.
- [ ] 26. [WS2 2.4] Entropy back-end for codes, values and byte-shuffled scales
  Recommended task executor category: deep-low
  Depends on: 25. Rental: none (local CPU). Cost: USD 0.
  Acceptance: Per-stream ratio vs entropy bound table.
  QA: run the tests listed in 2.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: Entropy back-end for codes, values and byte-shuffled scales`.
- [ ] 27. [WS2 2.5] rANS ablation for value/code streams (keep only if it wins)
  Recommended task executor category: deep-low
  Depends on: 26. Rental: none (local CPU). Cost: USD 0.
  Acceptance: ablation JSON + decision note in docs/protocol.md.
  QA: run the tests listed in 2.5, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: rANS ablation for value/code streams (keep only if it wins)`.
- [ ] 28. [WS2 2.6] Wire v2 into miner, aggregator, auditor, capacity worker and service
  Recommended task executor category: deep-high
  Depends on: 26, 5. Rental: none (local CPU). Cost: USD 0.
  Acceptance: Full pytest log; e2e test_service_network_v2_e2e green with v2.
  QA: run the tests listed in 2.6, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: Wire v2 into miner, aggregator, auditor, capacity worker and`.
- [ ] 29. [WS2 2.7] Checkpoint / state-transfer lossless path (ZipNN-style + XOR delta)
  Recommended task executor category: deep-low
  Depends on: 26. Rental: none (local CPU). Cost: USD 0.
  Acceptance: Ratio table (expect ~1.5x bf16, ~1.17x fp32 per ZipNN; measured).
  QA: run the tests listed in 2.7, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: Checkpoint / state-transfer lossless path (ZipNN-style + XOR`.
- [ ] 30. [WS2 2.8] Compression ablation report
  Recommended task executor category: writing
  Depends on: 27, 28, 29. Rental: reuse 2.1 corpus. Cost: USD 0.
  Acceptance: Report JSON checked by scripts/check_doc_numbers.py tags.
  QA: run the tests listed in 2.8, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: Compression ablation report`.
- [ ] 31. [WS2 2.9] GPU-side codec feasibility (nvCOMP ANS / DietGPU)
  Recommended task executor category: deep-low
  Depends on: 30. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 5.
  Acceptance: Bench JSON + adopt/reject note.
  QA: run the tests listed in 2.9, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws2: GPU-side codec feasibility (nvCOMP ANS / DietGPU)`.
- [ ] 32. [WS5 5.1] k-of-n auditor quorum per reward item with distinct coldkeys
  Recommended task executor category: deep-high
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 5.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws5: k-of-n auditor quorum per reward item with distinct coldkeys`.
- [ ] 33. [WS5 5.2] Referee-confirmed TRANSIENT
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 5.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws5: Referee-confirmed TRANSIENT`.
- [ ] 34. [WS5 5.3] Audit window beyond anchor age 1 and rollback beyond w+2
  Recommended task executor category: deep-high
  Depends on: 29. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 5.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws5: Audit window beyond anchor age 1 and rollback beyond w+2`.
- [ ] 35. [WS5 5.4] Escrow remainder rotation and total==0 guard
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 5.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws5: Escrow remainder rotation and total==0 guard`.
- [ ] 36. [WS5 5.5] Bounded public API cache and drand latest() fix
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 5.5, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws5: Bounded public API cache and drand latest() fix`.
- [ ] 37. [WS5 5.6] Envelope expiry
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 5.6, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws5: Envelope expiry`.
- [ ] 38. [WS3 3.1] Parallel resumable chunk upload with retries
  Recommended task executor category: deep-low
  Depends on: 5. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log; kind smoke (scripts/relay_kind_smoke.py) pass log.
  QA: run the tests listed in 3.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws3: Parallel resumable chunk upload with retries`.
- [ ] 39. [WS3 3.2] Size-scaled transfer deadline
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 3.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws3: Size-scaled transfer deadline`.
- [ ] 40. [WS3 3.3] Relay stores chunks + signed chunk manifest (no reassembly)
  Recommended task executor category: deep-high
  Depends on: 38. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log; kind smoke log.
  QA: run the tests listed in 3.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws3: Relay stores chunks + signed chunk manifest (no reassembly)`.
- [ ] 41. [WS3 3.4] R2 multipart upload
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 3.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws3: R2 multipart upload`.
- [ ] 42. [WS3 3.5] Cross-region throughput and chaos test (T7)
  Recommended task executor category: deep-low
  Depends on: 38, 39, 40. Rental: 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3). Cost: USD 10.
  Acceptance: Throughput JSON per region pair; chaos test log.
  QA: run the tests listed in 3.5, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws3: Cross-region throughput and chaos test (T7)`.
- [ ] 43. [WS8 8.1] Shard format v3 (dtype field, varlen records, sidecars)
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 8.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws8: Shard format v3 (dtype field, varlen records, sidecars)`.
- [ ] 44. [WS8 8.2] Train-train dedup and fix teacher label/build key mismatch
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 8.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws8: Train-train dedup and fix teacher label/build key mismatch`.
- [ ] 45. [WS7 7.1] Arch registry replacing decoder-vs-OpenDecision branch
  Recommended task executor category: deep-low
  Depends on: 14. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 7.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Arch registry replacing decoder-vs-OpenDecision branch`.
- [ ] 46. [WS7 7.2] Decoder variants: MoE shared experts, MLA
  Recommended task executor category: deep-low
  Depends on: 45, 15. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 6.
  Acceptance: 5090 smoke: 200 steps, loss decreases, replay bitwise.
  QA: run the tests listed in 7.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Decoder variants: MoE shared experts, MLA`.
- [ ] 47. [WS7 7.3] Encoder (MLM) family
  Recommended task executor category: deep-low
  Depends on: 45, 43. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 6.
  Acceptance: 5090 smoke.
  QA: run the tests listed in 7.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Encoder (MLM) family`.
- [ ] 48. [WS7 7.4] Encoder-decoder family
  Recommended task executor category: deep-low
  Depends on: 47. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 6.
  Acceptance: 5090 smoke.
  QA: run the tests listed in 7.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Encoder-decoder family`.
- [ ] 49. [WS7 7.5] Vision (ViT) family
  Recommended task executor category: deep-low
  Depends on: 45, 43. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 6.
  Acceptance: 5090 smoke.
  QA: run the tests listed in 7.5, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Vision (ViT) family`.
- [ ] 50. [WS7 7.6] Multimodal (vision tower + projector + LM)
  Recommended task executor category: deep-low
  Depends on: 49, 43. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 8.
  Acceptance: 5090 smoke.
  QA: run the tests listed in 7.6, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Multimodal (vision tower + projector + LM)`.
- [ ] 51. [WS7 7.7] Diffusion / flow matching
  Recommended task executor category: deep-low
  Depends on: 49. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 8.
  Acceptance: 5090 smoke.
  QA: run the tests listed in 7.7, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Diffusion / flow matching`.
- [ ] 52. [WS7 7.8] SSM / hybrid
  Recommended task executor category: deep-high
  Depends on: 45. Rental: Vast.ai 1x RTX 5090 (verified datacenter, >=64 GB host RAM, PCIe 5 x16), on-demand. Cost: USD 8.
  Acceptance: 5090 smoke.
  QA: run the tests listed in 7.8, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: SSM / hybrid`.
- [ ] 53. [WS7 7.9] Long context: RoPE scaling, sliding window, context parallel
  Recommended task executor category: deep-high
  Depends on: 45, 12, 43. Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py). Cost: USD 12.
  Acceptance: 5090 smoke at seq 32k on 8 GPUs.
  QA: run the tests listed in 7.9, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Long context: RoPE scaling, sliding window, context parallel`.
- [ ] 54. [WS7 7.10] Objectives: loss mask CE, distillation, DPO, GRPO
  Recommended task executor category: deep-low
  Depends on: 45, 43. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 7.10, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Objectives: loss mask CE, distillation, DPO, GRPO`.
- [ ] 55. [WS6 6.1] Regional aggregator role with cross-region hash agreement
  Recommended task executor category: deep-high
  Depends on: 6, 40. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log; kustomize render of 3 overlays.
  QA: run the tests listed in 6.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws6: Regional aggregator role with cross-region hash agreement`.
- [ ] 56. [WS6 6.2] Elastic membership: join/leave per round, roster from admissions
  Recommended task executor category: deep-high
  Depends on: 7, 8. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 6.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws6: Elastic membership: join/leave per round, roster from admiss`.
- [ ] 57. [WS6 6.3] Verified state sync for late joiners
  Recommended task executor category: deep-low
  Depends on: 56, 29. Rental: none (local CPU). Cost: USD 0.
  Acceptance: e2e log.
  QA: run the tests listed in 6.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws6: Verified state sync for late joiners`.
- [ ] 58. [WS6 6.4] Layout-invariant checkpoints (islands with different GPU counts)
  Recommended task executor category: deep-low
  Depends on: 57. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 6.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws6: Layout-invariant checkpoints (islands with different GPU cou`.
- [ ] 59. [WS7 7.11] Tensor and pipeline parallelism with fixed order
  Recommended task executor category: deep-high
  Depends on: 12, 58. Rental: Vast.ai 8x RTX 5090 single host (same filter as experiments/gpu_phase_b/select_hosts.py). Cost: USD 10.
  Acceptance: 8x 5090 run for 7B-class shape fits and replays.
  QA: run the tests listed in 7.11, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws7: Tensor and pipeline parallelism with fixed order`.
- [ ] 60. [WS6 6.5] Metrics, tracing and rate limits
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 6.5, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws6: Metrics, tracing and rate limits`.
- [ ] 61. [WS6 6.6] Live multi-region run
  Recommended task executor category: unspecified-high
  Depends on: 55, 56, 57, 58, 60, 32, 28. Rental: 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3); 2 x Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py). Cost: USD 60.
  Acceptance: Per-round finalised roots agreed by >= 2 regions; join/leave events in journal; auditor quorum verdicts; Vast + CPU charges reconciled.
  QA: run the tests listed in 6.6, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws6: Live multi-region run`.
- [ ] 62. [WS9 9.1] Fix doc inconsistencies
  Recommended task executor category: writing
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 9.1, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws9: Fix doc inconsistencies`.
- [ ] 63. [WS9 9.2] Property/fuzz tests T1, T2, T9 + hypothesis dev dep
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: pytest log.
  QA: run the tests listed in 9.2, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws9: Property/fuzz tests T1, T2, T9 + hypothesis dev dep`.
- [ ] 64. [WS9 9.3] Supply chain T11 and gitleaks in CI
  Recommended task executor category: unspecified-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: CI logs.
  QA: run the tests listed in 9.3, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws9: Supply chain T11 and gitleaks in CI`.
- [ ] 65. [WS9 9.4] Cross-hardware determinism matrix T4
  Recommended task executor category: deep-low
  Depends on: 20. Rental: Vast.ai 1 GPU each: RTX 5090 x2 drivers, H100, L40S. Cost: USD 20.
  Acceptance: matrix JSON.
  QA: run the tests listed in 9.4, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws9: Cross-hardware determinism matrix T4`.
- [ ] 66. [WS9 9.5] Adversarial e2e T6 on GPU
  Recommended task executor category: deep-low
  Depends on: 61. Rental: Vast.ai 2x RTX 5090 single host (same offer filter as scripts/vast_e2e.py). Cost: USD 10.
  Acceptance: catch-rate JSON.
  QA: run the tests listed in 9.5, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws9: Adversarial e2e T6 on GPU`.
- [ ] 67. [WS9 9.6] Mutation testing T10 and direct unit tests T13
  Recommended task executor category: deep-low
  Depends on: none. Rental: none (local CPU). Cost: USD 0.
  Acceptance: mutmut report.
  QA: run the tests listed in 9.6, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws9: Mutation testing T10 and direct unit tests T13`.
- [ ] 68. [WS9 9.7] Parity grid >= 8 seeds (T14) on CPU nodes
  Recommended task executor category: deep-low
  Depends on: none. Rental: 3 CPU nodes, one per region EU/US/APAC: 16 vCPU, 64 GB RAM, 500 GB NVMe, >=1 Gbit/s (Vast.ai CPU offer if one exists in region, else equivalent hourly cloud VM; owner decision D3). Cost: USD 40.
  Acceptance: parity results + TOST verdict.
  QA: run the tests listed in 9.7, then `scripts/check.sh`; failure case: the listed rejection tests must fail before the change where they lock a bug.
  Commit: one atomic commit `ws9: Parity grid >= 8 seeds (T14) on CPU nodes`.

## Final verification wave

- [ ] F1. Recompute every MFU claim from raw evidence
  Recommended task executor category: unspecified-high
  How: scripts/check_gpu_evidence.py --kind mfu over experiments/results/mfu/final-*.json; require min > 40% and replay_match true on all.
- [ ] F2. Compression claims
  Recommended task executor category: unspecified-high
  How: rerun experiments/compression/entropy_bound.py on the captured corpus; require v2 within 5% of bound and ef_out v1==v2 test green.
- [ ] F3. Network live run audit
  Recommended task executor category: unspecified-high
  How: verify network_live JSON: 20 finalised rounds, >=2 regional agreement each round, join/leave/failover events, quorum verdicts.
- [ ] F4. Full quality gate + spend reconciliation
  Recommended task executor category: unspecified-high
  How: scripts/check.sh + slow suite exit 0; Vast/CPU charges <= ceilings via provider GET.

## Must NOT have

- No reduced-scope substitute for any target (no 'MFU on a toy shape', no lossy change sold as lossless).
- No nondeterministic kernel, NCCL reduction op on verified tensors, or autotuning in qualified runs.
- No credentials in logs, evidence or git; no merge, deploy or push to main.

## Review amendments (review.json)

- A1 1.8 cryptographic chunk digests
- A2 4.2 SIMD-independent pairwise tree
- A3 2.2 no GPU dependency; v2 = lossless transform of v1 bytes
- A4 1.2 precision policy clarified
- A5 MFU denominator n_gpus x peak
