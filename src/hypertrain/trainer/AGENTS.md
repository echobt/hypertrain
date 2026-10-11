# src/hypertrain/trainer

## OVERVIEW
Deterministic island trainer (leaf scheme `ht-leaf-v1`); importing the package pins determinism. Score 19: config/model/compress imported 90/79/70 times.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Determinism pin | `determinism.py`, `__init__.py` | `DETERMINISM = setup_determinism()` at import; `apply_reference_env`, `require_threads` |
| Configs | `config.py` | `ModelConfig`, `InnerConfig`, `CompressConfig`, `TrainConfig` |
| Model params | `model.py` | `init_params`, `param_shapes`, `stage_of`, `forward` (OD model mirrors it) |
| Delta codecs | `compress.py` | `ht-dense-int8-v1` ("dense-int8", default) + `ht-sparse-v1` ("sparseloco"); `compress`, `decompress`, `validate_payload`, `state_hash`, `canonical_topk` |
| Inner loop / replay | `loop.py` | `train_round`, `replay`, `Assignment`, `RoundResult`, `batch_hash` |
| Multi-GPU island | `island.py` (759) | `Comm`/`DistComm`/`ThreadComm`, `emulate`, `forbid_reductions`, `check_launch` |
| Optimizer | `optim.py` | |
| RNG | `rng.py` | own Philox4x32 (`philox4x32`, `rng_ctr`, bounded by `RNG_CTR_MAX`); never torch RNG |

## CONVENTIONS
- Pinned: `CUBLAS_WORKSPACE_CONFIG=:4096:8`, deterministic algorithms (no warn_only), TF32 off, no reduced-precision bf16/fp16 reductions, matmul precision highest, CPU threads pinned (auditor must match manifest `reference_spec.env.cpu_threads`).
- Codecs (`config.py:105` default `dense-int8`): dense-int8 = bits 8, blocks of 256, f32 scale amax/127; sparseloco = bits 2, `0 < topk_frac <= 1`, indices u32 ascending. Both lossy with error feedback; `validate_payload` defaults to a 64 KiB cap.
- Torch pinned `torch==2.14.0` via `trainer` extra.

## ANTI-PATTERNS
- Banned in source (enforced by `tests/trainer/test_trainer_core.py::test_forbidden_constructs_absent`): `topk(`, `index_add(_)`, `scatter_add(_)`, `compile(`, `allclose(`, `isclose(`. Use `canonical_topk` (|x| desc, index asc, stable).
- `all_reduce`/`reduce`/`reduce_scatter` on verified tensors; `forbid_reductions()` raises `ForbiddenCollective`.
- Decoding or allocating before metadata validation (`tests/trainer/test_codec_metadata.py` monkeypatches `forbidden_*`).
- Importing torch here before `determinism` runs; `setup_determinism` raises if torch preloaded without the cuBLAS env.

## TESTS
`tests/trainer/` (helpers `trainer_fixtures.py`, `fake_device.py`, `_worker.py`), `tests/layout/`.
