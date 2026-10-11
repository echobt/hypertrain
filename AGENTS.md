# PROJECT KNOWLEDGE BASE

**Generated:** 2026-10-11T21:33:38.961Z
**Commit:** 8baf7c8
**Branch:** boat/224a523e5f92

## OVERVIEW
hypertrain: decentralized deterministic training network (coordinator, miners, aggregator, auditor, relay). Python 3.12 + uv, FastAPI, pydantic, sr25519 signing, torch 2.14 via optional `trainer` extra.

## STRUCTURE
```
repo/
├── src/hypertrain/   # 18 subpackages; see src/hypertrain/AGENTS.md
├── tests/            # mirrors src/hypertrain/<pkg>; sim/ + e2e/ run in CI job test-slow
├── scripts/          # check.sh gate, publish, canary, proof drivers
├── experiments/      # sim + GPU phase A/B + network v2; results/ partly generated
├── docs/             # schemas/ generated; audit/ = audit.md + audit.json report
├── deploy/k8s/       # kustomize base + overlays local|apac|eu|us; versions.json pins
├── docker/           # Dockerfile.server, Dockerfile.gpu, phase-a
└── teacher.example.toml
```

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Wire model / envelope | src/hypertrain/protocol/ | regenerate docs/schemas after change |
| Coordinator state | src/hypertrain/challenge/store.py | `ChallengeStore`, huge SQLite state machine |
| Determinism rules | src/hypertrain/trainer/determinism.py | import-order guard |
| `hypertrain` CLI | src/hypertrain/aggregator/cli.py | console script |
| `hypertrain-miner` CLI | src/hypertrain/miner/cli.py | console script |
| Module CLIs | `python -m hypertrain.{auditor,beacon,datasets,storage,teacher,models.opendecision}` | |
| CI | .github/workflows/ci.yml | jobs test, test-slow, canary |
| Audit findings | docs/audit/audit.md, docs/audit/audit.json | |
| Infra pins | deploy/k8s/versions.json | kind/k8s/Calico by digest |

## CODE MAP
Refs = grep hits in .py under src, tests, scripts, experiments.

| Symbol | Type | Location | Refs | Role |
|--------|------|----------|------|------|
| `canonicalize` | fn | src/hypertrain/protocol/jcs.py | 773 | RFC 8785 JCS bytes for hashing/signing |
| `envelope_v2` | module | src/hypertrain/protocol/envelope_v2.py | 535 | domain-separated signed envelopes |
| `seal` | fn | src/hypertrain/protocol/envelope_v2.py | 384 | sign envelope; signer = body hotkey |
| `Keypair` | class | src/hypertrain/protocol/keys.py | 266 | sr25519 keys, SS58 hotkeys |
| `f32hex` | fn | src/hypertrain/protocol/messages.py | 124 | float -> f32 hex; no raw floats on wire |
| `ChallengeStore` | class | src/hypertrain/challenge/store.py | 54 | coordinator persistence/state machine |
| `NEVER` | const | src/hypertrain/challenge/store.py | 9 | `2**53-1` drand round for non-expiring seals |
| `setup_determinism` | fn | src/hypertrain/trainer/determinism.py | 7 | pins cuBLAS/TF32/threads; raises if torch preloaded |
| `canonical_topk` | fn | src/hypertrain/trainer/compress.py | 7 | deterministic topk; use instead of `torch.topk` |

## CONVENTIONS
- `import hypertrain.trainer` BEFORE any torch import - src, tests, experiments, scripts. Else `RuntimeError("torch imported before hypertrain determinism setup")` (determinism.py:45-47). Torch test dirs carry a 1-line conftest doing this import.
- Wire: pydantic extra fields forbidden; floats as f32 hex; time = drand quicknet rounds only, never wall clock.
- Versioned replacements use `_v2` suffix (e.g. `envelope_v2`); old module kept.
- pytest `pythonpath=["."]`: `scripts/` and `experiments/` import as namespace packages.
- Everything pinned by digest/sha256 (images, k8s, gpu requirements); never floating tags.
- Deliberate simplifications marked `ponytail:` comments.

## ANTI-PATTERNS (THIS PROJECT)
- NEVER `torch.topk`, collective reductions on verified tensors, or torch's global RNG (own Philox RNG).
- NEVER hand-edit `docs/schemas/*.json` or `experiments/results/mechanism_tables.json`; regenerate.
- NEVER sign non-JCS bytes or raw floats.
- NEVER use the repo `data/` dir in tests; `tests/conftest.py` builds a hermetic corpus into `HYPERTRAIN_DATA_DIR`.
- NEVER publish outside `scripts/publish.sh` (`.publishignore`, `.gitleaks.toml`, `scripts/publish-allowlist.txt`; guarded by tests/test_publish_exclusions.py).

## UNIQUE STYLES
- Test names `test_<outcome>_when_<condition>`; `forbidden` monkeypatch guard pattern.
- `docs/schemas/*.json` guarded by a byte-equality test against the generator.

## COMMANDS
```bash
uv sync --locked --all-extras --group dev
bash scripts/check.sh                    # gate: ruff format --check, ruff check, mypy, pytest
bash scripts/check.sh tests/protocol     # extra args go to pytest
uv run pytest tests/sim tests/e2e        # = CI test-slow
uv run python -m hypertrain.protocol.schema docs/schemas
uv run python scripts/gen_mechanism_tables.py
docker build -f docker/Dockerfile.server -t hypertrain:local . && ENGINE=docker bash scripts/canary.sh
```

## NOTES
- 10 real-systemd capacity/launch tests need root; fail locally as non-root - expected. CI runs them in privileged job `test`, then deselects them.
- CI `test` ignores tests/sim + tests/e2e; those only run in `test-slow`.
- mypy runs with `MYPYPATH=src:scripts:.` plus explicit modules from experiments/ and scripts/ (see scripts/check.sh).
- `scripts/check_doc_numbers.py` is not in CI; docs/parity.md currently fails it (missing experiments/results/summary_smoke.json).
- uv environments restricted to linux x86_64.
- Other workflows: images.yml (CUDA image), release.yml, gpu-vast.yml (scripts/vast_e2e.py).
