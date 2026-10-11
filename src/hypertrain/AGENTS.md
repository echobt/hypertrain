# src/hypertrain

## OVERVIEW
Installed package (src layout, `py.typed`); 18 subpackages. Score 22: package boundary, highest centrality.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Coordinator HTTP + state | `challenge/` | own AGENTS.md |
| Wire messages, signing, JCS | `protocol/` | own AGENTS.md |
| Deterministic training core | `trainer/` | own AGENTS.md |
| Outer step, tapes, checkpoints, rollback | `aggregator/` | own AGENTS.md; `hypertrain` console script |
| Replay / bisect / honeypots | `auditor/` | own AGENTS.md |
| Miner client, torchrun islands | `miner/` | own AGENTS.md; `hypertrain-miner` script |
| Vast.ai GPU ops, journal, qualification | `gpu_ops/` | own AGENTS.md |
| Emission ledger, escrow v2 | `ledger/` | integer units, hash-chained journal (`GENESIS_HASH`, flock, `durable_write`) |
| Relay transport (regions) | `relay/` | `core.Relay` 967 LOC; `app.create_app`; `client.RelayClient` |
| drand randomness | `beacon/` | quicknet only; `FixtureBeacon` in tests; `python -m hypertrain.beacon` |
| Object store, shards, assignment | `data/` | `LocalFSStore` (most-used test dep); hand-written SigV4 S3 |
| Dataset registry/build | `datasets/` | `registry.json`; `python -m hypertrain.datasets`; empty `__init__` |
| R2 publish | `storage/` | immutable puts; `python -m hypertrain.storage` |
| Read-only public views | `public_api/` | `create_public_app`; sqlite snapshot views |
| Labelling teacher | `teacher/` | `python -m hypertrain.teacher` |
| OpenDecision model | `models/opendecision/` | `od` extra, pinned git commit; dense only (`moe` must be None) |
| Parity stats | `parity/` | TOST, bootstrap CI90 |

## CONVENTIONS
- `from __future__ import annotations` in every module.
- Wire/config models: pydantic `ConfigDict(extra="forbid", frozen=True)` (often `strict=True`, `StrictInt`).
- Floats never raw on the wire: f32 hex via `protocol.messages.f32hex` / `f32val`; exact math via `Fraction`.
- Time = drand round numbers, never wall clock, for every deadline.
- `_v2` suffix = network protocol v2 living beside frozen v1 (`messages`/`messages_v2`, `envelope`/`envelope_v2`).
- Lazy function-local imports break cycles (aggregator <-> challenge.store, checkpoint <-> rollback_v2 <-> tape_v2). Keep them local.
- Subprocess: fixed argv lists only; each `# noqa: S603/S607/S310` carries a reason. Broad except only as `# noqa: BLE001 - <reason>`.
- `ponytail:` comments mark deliberate simplifications with ceiling + upgrade path; keep format.
- Package `__init__` files mostly empty: import from submodules.
- Content-addressed objects: SHA-256 hex keys, verified on read (`CorruptObjectError`).

## ANTI-PATTERNS
- `import torch` (directly or transitively) before `import hypertrain.trainer` - `setup_determinism` raises.
- `shell=True`, printing/journaling API keys, writes to non-loopback hosts without the live flag.
- Overwriting R2 objects; treating unknown/429/malformed provider inventory as absence.
- MS MARCO in any dataset source (`test_no_ms_marco_anywhere`).
