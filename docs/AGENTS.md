# docs

## OVERVIEW
Operator/miner/protocol docs, frozen v1 JSON schemas, executable doctests, evidence-tagged numbers. Score 9, kept as distinct domain: several files are machine-checked.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Entry | `index.md` | |
| HTTP route + JSON contract | `challenge-routes.md` | mirror of `challenge/app.py` routes |
| Network v2 design | `network-v2.md` (437) | |
| Operator runbook | `operator.md` | contains `bash doctest` blocks |
| Miner guide | `miner.md` | doctests; determinism pin list |
| Protocol / mechanism | `protocol.md`, `mechanism.md` | mechanism tables from `experiments/results/mechanism_tables.json` |
| Results | `RESULTS.md` | includes French summary |
| Other | `opendecision.md`, `parity.md`, `r2.md`, `teacher.md`, `audit/`, `assets/` | |
| v1 wire schemas | `schemas/*.json` (17) | generated |

## CONVENTIONS
- `schemas/` is generated output: `python -m hypertrain.protocol.schema docs/schemas`; `tests/protocol/test_messages.py::test_schema_files_match_and_regenerate_byte_identically` requires byte equality with `render()`. Never hand-edit.
- Numeric claims tagged `0.5{ev:experiments/results/x.json#a.b}` (optional `*factor`); `scripts/check_doc_numbers.py` checks value + relative links.
- Runnable shell in operator/miner docs: fence as `bash doctest` (run by `scripts/doctest_docs.sh`, one bash per doc) or `doctest-skip:prod` (listed, never run).

## ANTI-PATTERNS
- Writing untagged numbers that come from evidence files.
- Doctest blocks that touch production / spend money without `doctest-skip:prod`.
- Changing a route in `challenge/app.py` without updating `challenge-routes.md`.

## NOTES
- `check_doc_numbers.py` is not in CI; `parity.md` currently fails it (missing `experiments/results/summary_smoke.json`).
