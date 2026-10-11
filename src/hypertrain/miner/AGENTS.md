# src/hypertrain/miner

## OVERVIEW
Miner client: coordinator API, torchrun island launch + capacity runs, dispute follow/watch. Score 16; `hypertrain-miner` (`cli.py:main`).

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Miner core | `core.py` (1065) | `Miner`, `NetworkMiner`, `Api`, `MinerConfig`, `load_keyfile`, `MinerError` |
| Island launch | `island_launch.py` (705) | `launch_island`, `validate_artifacts` (18+ importers), `confined`, `IslandArtifacts`, `run_capacity_argv` |
| torchrun worker | `island_worker.py` | `execute`; uses `trainer.determinism.apply_reference_env` |
| Join signing | `admission.py` | `sign_join`; own `main` |
| Disputes | `dispute_follow.py`, `dispute_watch.py` | `run_follow` |
| CLI | `cli.py` | |

## CONVENTIONS
- Capacity runs launch as native systemd units `ht-cap-<32hex>(-deadline).service|timer` (CI cleans by that regex).
- `ponytail:` host RAM not probed at admission; CLI admission via operator `PUT /roster`; dispute_follow on CPU cannot reproduce CUDA bits.
- Signs with `challenge.store.assignment_hash` / `rerun_message`; keep in sync with store.

## ANTI-PATTERNS
- Pickle or implicit CPU<->CUDA oracle in island_worker.
- Unbounded fallback when manager permission missing (`test_missing_manager_permission_has_no_unbounded_fallback`).
- Calling `Popen.communicate` in island launch paths (tests forbid it).
- Publishing artifacts before bounded failure check (`test_capacity_branch_uses_real_bounded_failure_before_publication`).

## COMMANDS
- `hypertrain-miner` (console script); sample config `tests/fixtures/miner.toml`.

## TESTS
`tests/miner/` (`miner_harness.py` `World`/`Operator`, `world_factory` fixture). Real-runner tests in `test_capacity_runtime_v2.py`, `test_island_launch_v2.py` need root + systemd: they fail as normal user locally; CI runs them in the privileged native job.
