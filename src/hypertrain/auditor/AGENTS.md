# src/hypertrain/auditor

## OVERVIEW
Independent replay of miner work, bisection, honeypots; `python -m hypertrain.auditor`. Score 15, distinct domain; `replay.py` most-imported module in repo tests.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| State pack/replay | `replay.py` (907) | `unpack_state` (29+ importers), `pack_state`, `AnchorCache`, `tensor_root`, `optimizer_hash`, `AuditInputError` |
| Audit worker loop | `worker.py` | `Auditor`, `HttpApi`, `LeaseGuard` |
| Single-device bisect | `bisect.py` | |
| Island (multi-GPU) bisect | `island_bisect.py` | `IslandParty`, `RefereeEvidence` |
| Honeypots | `honeypot.py` | used by challenge tests |
| Entry | `__main__.py` | |

## CONVENTIONS
- `__init__.py` itself imports `hypertrain.trainer` first: any `hypertrain.auditor` import pins determinism.
- Fault attribution: undecodable blob = miner fault (`# noqa: BLE001` justified); infra errors retried in worker.
- Non-deterministic miner stack = FAULT.

## ANTI-PATTERNS
- Admitting anything into `AnchorCache` except deterministic genesis or successful independent full replay.
- Ignoring genesis theta mismatch: must raise `AuditInputError`.
- Replay that trusts miner assertions instead of recomputing.

## COMMANDS
- `python -m hypertrain.auditor` (worker entry).
- Manual QA: `uv run python tests/auditor/qa_scenarios.py` (imports conftest first on purpose; ponytail note).

## TESTS
`tests/auditor/` (`aud_fixtures.py`, manual `qa_scenarios.py`), `tests/miner/` (bisect/dispute), `tests/challenge/test_auditor_e2e.py`.
