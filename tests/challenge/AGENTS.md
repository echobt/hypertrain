# tests/challenge

## OVERVIEW
Coordinator tests: store rounds, admission, disputes, trust, capacity/outer v2, full v2 service via TestClient. Score 18: 22 files, ~21k LOC, heaviest `ChallengeStore` consumer.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Shared world | `conftest.py` | `SLUG`, `COORD_SEED=bytes(range(32))`, `AUDITOR_SEED`, `OWNER=Keypair(b"\x77"*32)`, `PARAMS`, `FixtureBeacon`, `Master`, `make_client`, `clock`, `Net` fixture `net` |
| v2 network service | `test_service_network_v2.py` (4071) | largest test file in repo |
| Capacity outer v2 | `test_capacity_outer_v2.py` (2558), `test_service_capacity_v2.py` | include real-kernel tests split in CI |
| Fixtures | `authority_fixture_v2.py`, `protected_opening_fixture.py` (1945), `protected_opening_fixture_static.py` | |
| Rounds v1 | `test_rounds.py`, `test_fix1.py`, `test_multirun.py`, `test_contract.py` | |

## CONVENTIONS
- Package (`__init__.py`): import helpers relatively, `from .conftest import ...`.
- Time driven by `clock()` list + monkeypatched `store.time.time`; rounds via `beacon_payload(rnd)`.
- Some v2 tests load scripts via `importlib.util` and `ast`-check source.
- Service tests use `fastapi.testclient` + `httpx`.

## ANTI-PATTERNS
- Letting opening/fixture paths gain kernel approval ("signed hermetic metadata/state oracle; opening never gains kernel approval").
- Removing `forbidden` guards on `store._capacity_outer_v2`, `island_launch.run_capacity_argv`, `init_params`, `decompress`, `OuterState.from_bytes`, `store.verified_inputs_v2`.

## CI SPLIT
Native root/systemd job runs, main job `--deselect`s:
- `test_capacity_outer_v2.py::test_actual_bounded_outer_child_matches_original_signed_replay`
- `test_capacity_outer_v2.py::test_actual_bounded_genesis_preparation_and_duplicate_debit`
- `test_service_capacity_v2.py::test_real_capacity_launch_failure_keeps_debit_and_prevents_restart_rerun`
These fail locally as non-root; that is expected, not a regression.
