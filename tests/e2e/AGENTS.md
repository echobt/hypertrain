# tests/e2e

## OVERVIEW
Multi-process end-to-end: real uvicorn coordinator + actor subprocesses (miner, auditor, aggregator, relay). Score 15: distinct harness, own CI job (`test-slow`).

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| World / process mgmt | `harness.py` | `World` (central fixture object), `Server`, `Proc`, `ActorError`, `key(i)=bytes([i])*32`, `NO_REPLAY` |
| Actor subprocess | `actor.py` (598) | `MinerActor`, `AuditorActor`, `AggregatorActor`, `E2EMiner(core.Miner)`, `Relay`, `install_no_replay` |
| Server subprocess | `server.py` | `ReadyServer(uvicorn.Server)` |
| Shared constants | `common.py` | `INTERNAL, ADMIN, WORKER, COORD, AUDITOR, REFEREE, OWNER, TIMEOUTS, HTTP_LIMITS, DELTA, EPOCH_SECONDS` |
| Fixtures | `conftest.py` | `evidence_root`, `scenario_dir`, `record`, `world` |
| Scenarios | `test_service_network_v2_e2e.py` (2175), `test_network_service_proof.py` (1638), `test_rollback_*_v2.py`, `test_e2e.py` | |

## CONVENTIONS
- `common`, `harness`, `trainer_fixtures` imported as bare modules: tests prepend `tests/e2e` and `tests/trainer` to `sys.path` (`# noqa: E402`).
- Import `common` / `hypertrain.trainer` before torch.
- Timeouts in drand rounds (`TIMEOUTS`); protocol enforces `audit_after_commit >= 40`. All events in one ledger epoch (`EPOCH_SECONDS=86400`).
- `HT_E2E_EVIDENCE=<dir>` keeps per-scenario `result.json` / `processes.json`; default tmp.
- Scripts (`scripts/network_gpu_seed.py`, `network_service_proof.py`, `network_gpu_prepare.py`) loaded by path.

## ANTI-PATTERNS
- HTTP keepalive reuse in actors: keep `max_keepalive_connections=0` (races server idle close).
- Private material in prepared public output (`private`, `.key`, `.seed`, `.hex` asserted absent).
- Spawning real proofs where tests patch `driver.proof` / `subprocess.Popen` to `forbidden`.

## COMMANDS
- `HT_E2E_EVIDENCE=<dir> uv run --frozen pytest tests/e2e` to keep evidence.
`uv run --frozen pytest -q --durations=25 tests/sim tests/e2e` (excluded from `scripts/check.sh` in CI via `--ignore`).
