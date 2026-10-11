# src/hypertrain/aggregator

## OVERVIEW
Outer optimizer step, robust aggregation, v2 tapes, checkpoints, rollback/repair, capacity worker. Score 17; `hypertrain` console script (`cli.py:main`).

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Outer step + aggregation | `core.py` (865) | `aggregate`, `centered_clip`, `preclip`, `regional_mean`, `outer_step`, `OuterState`, `OuterParams`, `th`, `get_object` |
| Error types | `core.py` | `AggregatorError`, `HashMismatch`, `MissingObject`, `MalformedDelta`, `ReplayMismatch`, `FinalityError`, `TapeError`, `SequenceError`, `JournalError` |
| Signed aggregation tape | `tape_v2.py` | `TapeV2`, `make_tape`, `replay_tape`, `VerifiedInput`, `Exclusion` |
| Weighted allocation | `weighted_v2.py` | `allocate_weights`, `aggregate_weighted`, `InsufficientEligibleWeight` |
| Checkpoints | `checkpoint.py` (730) | `write/verify_network_checkpoint`, `network_inputs`, `require_network_roster` |
| Rollback / repair | `rollback_v2.py` (1131) | `RepairTape`, `execute_repair_context`, `replay_history_context`, `context_anchors` |
| Capacity child process | `capacity_worker.py` | `collect_genesis`, `collect_result`, `execute`, `bounded_bytes`; private `_directory_fd`/`_read_at` used by scripts |
| CLI | `cli.py` | |

## CONVENTIONS
- Cycle checkpoint <-> rollback_v2 <-> tape_v2 broken with function-local imports; `challenge/store.py` imports this package lazily too.
- `core` is the hub: every sibling and challenge.store import it; keep its API stable.
- `ponytail:` `core.py:47` overlap-screen top-k fraction hardcoded 1% (not a manifest field yet).

## ANTI-PATTERNS
- Rollback after finality or once next round applied (tests assert refusal).
- Accepting forged duplicate journal records: rebuild must break.
- Slashing on copy suspicion: it raises q, never slashes.
- Replay paths calling `trainer.model.init_params`, `compress.decompress`, `OuterState.from_bytes` where tests install `forbidden` guards.

## TESTS
`tests/aggregator/` (`agg_helpers.py`, manual `qa_scenarios.py` not collected), `tests/e2e/test_rollback_*`.
