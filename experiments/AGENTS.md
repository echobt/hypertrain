# experiments

## OVERVIEW
Research harnesses: CPU DiLoCo simulations, GPU phase A/B benchmarks, GPU network v2 orchestration, recorded results. Score 12, distinct domain; namespace package imported by tests/scripts.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Simulation library | `sim/diloco.py` | `simulate`, `simulate_streaming`, `data_parallel`, `SimSpec`, `Arm`; used by `tests/sim` |
| Parity grid | `sim/parity_grid.py` (798), `sim/parity_prereg.json` | preregistered designs/hypotheses |
| State policy | `sim/state_policy.py` | |
| GPU network v2 | `gpu_network_v2/orchestrate.py` (3028), `run.py`, `profile.json` | mypy-checked; `run.py JOB_DIR [cpu|cuda]` |
| Phase A | `gpu_phase_a/run_phase_a.py`, `select_hosts.py`, `phase_a.json` | GPU host only |
| Phase B | `gpu_phase_b/orchestrate.py`, `run_phase_b.py`, `select_hosts.py`, `nccl_bench.py`, `live-config.example.json` | |
| Recorded outputs | `results/*.json|csv|jsonl` | data only; cited by docs `{ev:...}` tags |
| Data / trainer QA | `fetch_data.py`, `trainer_qa.py` | |

## CONVENTIONS
- `profile.json` floats hex-encoded (`topk_frac`, `ef_beta`), sparseloco outer optimizer.
- `ponytail:` diloco has no robust aggregation, no Kale eager proxy, mean (m,v); parity_grid per-tensor top-k not chunked.
- Phase A: image pinned by digest (`docker/phase-a/IMAGE.txt`), `pip install --require-hashes -r docker/phase-a/requirements.txt`; refuses unless `torch==2.14.0+cu130` and SM count 170.
- Results status: parity runs mostly `CENSORED` (user CPU/energy stop 2026-10-08, 0/150 finished); summary cells null.

## COMMANDS
- `uv run --frozen python experiments/gpu_network_v2/run.py JOB_DIR [cpu|cuda]`
- `uv run --frozen pytest -q tests/sim` (slow CI job)

## ANTI-PATTERNS
- Editing preregistration after seeing results.
- Hand-editing `results/mechanism_tables.json`: regenerate with `scripts/gen_mechanism_tables.py`.
- Provider actions from `gpu_network_v2/run.py` ("no provider actions"; content-authenticated staged jobs only).
- Torch before `hypertrain.trainer` in any script here.
