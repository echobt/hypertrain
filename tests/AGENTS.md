# tests

## OVERVIEW
pytest suite mirroring `src/hypertrain/<pkg>` as `tests/<pkg>/`, plus repo-level script/publish tests. Score 20: 25 subdirs, ~60k LOC.

## STRUCTURE
```
tests/
├── conftest.py        # session autouse hermetic corpus -> HYPERTRAIN_DATA_DIR
├── challenge/         # own AGENTS.md; coordinator + v2 service
├── e2e/               # own AGENTS.md; multi-process, slow CI job
├── sim/               # experiments.sim smoke; slow CI job
├── miner/ auditor/ aggregator/ ledger/ trainer/ layout/ gpu_ops/ relay/ protocol/ ...
├── fixtures/miner.toml
└── test_*.py          # scripts + publish exclusions
```

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Corpus fixture | `conftest.py` | synthetic u32 shards; set `HYPERTRAIN_DATA_DIR` for real data |
| Publish exclusions | `test_publish_exclusions.py` (600) | `.publishignore`, secret scans |
| Script tests | `test_vast_e2e.py`, `test_check_doc_numbers.py`, `test_preservation_check.py` | load scripts by path |
| Shared helpers | `trainer/trainer_fixtures.py`, `trainer/fake_device.py`, `layout/layout_fixtures.py`, `auditor/aud_fixtures.py`, `miner/miner_harness.py`, `aggregator/agg_helpers.py`, `gpu_ops/mock_vast.py` | |
| Relay kind tests | `relay/test_transport_v2.py` (2960) | loads `scripts/relay_kind_smoke.py` by path |
| Manual QA | `aggregator/qa_scenarios.py`, `ledger/qa_scenarios.py`, `auditor/qa_scenarios.py` | run via `uv run python`, not collected |

## CONVENTIONS
- Torch-using subdirs have a 1-line `conftest.py`: `import hypertrain.trainer  # noqa: F401`. Add it to any new torch test dir; standalone files need `# noqa: E402` after it.
- `pythonpath=["."]`: import `scripts.*` / `experiments.*` as namespace packages, or `importlib.util.spec_from_file_location` / `runpy.run_path`.
- Prove a path is NOT taken: monkeypatch the target to a `forbidden(*args, **kwargs)` raiser.
- Names: `test_<outcome>_when_<condition>`; faults via `pytest.mark.parametrize("fault", [...])`.
- Hand-written fakes (`storage/fakes.py`, `mock_vast.py`), no mocking framework.
- Fixed seeds / fixed key bytes; statistical asserts use 99.9% normal-approx bands.
- Multi-process worker scripts prefixed `_` (`_worker.py`, `_island_worker.py`, `_phase_b_proxy.py`).
- `--strict-markers`: register any new marker in pyproject.

## ANTI-PATTERNS
- Reading `<repo>/data` directly (unpublished; fresh clone lacks it).
- Sleep-based waits; use events/fixtures like `wait_pid_exit`.
- Real provider/network calls; use local mocks.
- Adding a real-root/systemd test without wiring it into BOTH the native CI job list and the `--deselect` list in `.github/workflows/ci.yml`.
