# src/hypertrain/challenge

## OVERVIEW
Coordinator: FastAPI app + SQLite (WAL) state machine for runs, roster, rounds, intake, audits, disputes. Score 20: largest module, central to most tests.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| HTTP routes, `Config`, `create_app` | `app.py` | Cortex contract v1; `_Strict` base model; route contract in `docs/challenge-routes.md` |
| All persistent state | `store.py` | 9662 LOC god-class `ChallengeStore`; `ChallengeError` |
| Round open/finalize, forfeits, receipts | `store.py` | heaviest areas; relay network manifest ~L8078; capacity/outer v2 flows |
| Join/admission | `admission.py`, `admission_store.py` | `AdmissionError` |
| Disputes v2 | `disputes_v2.py` | consumed by miner/auditor tests |
| Finality v2 | `finality_v2.py` | small |
| Trust, audit floor, q | `trust_v2.py` | `nominal_q`, `audit_floor`, `select_audit`; ~10 importers |
| Module helpers | `store.py` | `assignment_hash`, `rerun_message` (used by `miner/core.py`), `audit_selected` |

## CONVENTIONS
- Clock = latest operator-pushed, verified drand round D. Ledger events carry drand chain time of D. `LEASE_ROUNDS = 600` (30 min at 3 s).
- Coordinator-sealed non-expiring records: `seal(self._coord(), "<Type>", run_id, body, NEVER)`, `NEVER = 2**53 - 1`.
- Sentinels: domain-separated sha256 (`b"ht-zero-outer-state"`, `b"ht-no-honeypot"`).
- Closed wire vocabularies: `FAULTS`, `MINER_TYPES`, `COORD_TYPES` - extend these, not ad-hoc strings.
- JSON via `_dumps`; exact math via `Fraction`.
- store.py lazily imports aggregator (`network_inputs`, `TapeV2`, `replay_tape`, `th`, `get_object`, `capacity_worker`); TYPE_CHECKING-only imports from `experiments.gpu_network_v2.orchestrate`, `scripts.network_gpu_operation`.
- Tests monkeypatch `hypertrain.challenge.store.time.time`: keep module-level `import time`; do not `from time import time`.

## ANTI-PATTERNS
- Wall-clock deadlines.
- Editing `_SCHEMA` DDL without updating `tests/public_api/test_public_api.py` (pins it).
- Splitting store.py casually: many tests import private paths (`store._capacity_outer_v2`, `store.verified_inputs_v2`) for `forbidden` monkeypatches.
- Paying unresolved disputes: unresolved at d_final = excluded, unpaid, no burn (ponytail ~L1748).

## NOTES
- `ponytail:` equal emission share per run, no cross-run 65,536-hotkey cap (~L309); sub-run objects always local, S3 prefix per run is upgrade path (~L478); app.py books dropped units as paid, burned to UID0.
- Tests: `tests/challenge`, `tests/e2e`, `tests/public_api`.
