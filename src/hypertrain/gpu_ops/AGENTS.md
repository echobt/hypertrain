# src/hypertrain/gpu_ops

## OVERVIEW
Rented-GPU operations (Vast.ai): provider client, SSH, launch orchestration, budget, journal, qualification, work screening. Score 14, distinct domain (external spend + secrets).

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Launch orchestrator | `launcher.py` (926) | `Orchestrator`, `Reject`; `main` |
| Provider HTTP | `provider.py` | `Provider`, `Response`, `redact`, no-redirect handler, `RefusedWrite` |
| SSH | `remote.py` | `Ssh`, `ensure_identity` |
| Durable journal | `journal.py` | `Journal`, `durable_write`, `fsha`, `sha` |
| Spend caps | `budget.py` | imported as module |
| Supervisor | `supervisor.py` | `main` |
| Network qualification | `network_qualification.py` | `sources` (aliased `engine` in tests); `scripts/network_gpu_qualification.py` wraps it |
| Work screening | `work_screen.py` | `screen_work`, `IslandLaunch` |
| Verdicts | `verdict.py` | |

## CONVENTIONS
- Every provider call journaled before acting; journal is the recovery source.
- `ponytail:` network_qualification checks custody only.
- Pydantic `extra="forbid", frozen=True` for qualification models.

## ANTI-PATTERNS
- Leaking API key into logs/journal; key file must be mode 0600.
- Non-loopback writes without the explicit live flag.
- Treating unknown / 429 / malformed inventory as absence; recreating an instance to "recover".
- Hardware hints as admission substitute (`test_work_screen_v2`).
- Deprecated or trailing-slash endpoints (mock returns 410 / 308).

## TESTS
`tests/gpu_ops/`: `mock_vast.py` (fake Vast HTTP API + real spawned sshd), `conftest.py` fixtures `mock_factory`, `run_cfg`, `launch`, `journal`, `wait_pid_exit`. Biggest: `test_network_admit_cli.py` (2662).
