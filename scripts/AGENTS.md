# scripts

## OVERVIEW
Quality gate, publish/secret controls, image checks, GPU/network proof drivers, kind relay smoke. Score 16: entry points for CI + operators; namespace package importable by tests.

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Full quality gate | `check.sh` | ruff format/check, mypy, pytest; extra args -> pytest |
| Publish to public repo | `publish.sh` | embedded Python secret scan; `--dry-run`, `--tag`, `--expect-token-file F`, env `HYPERTRAIN_PUBLISH_APPROVED` |
| Reviewed scan false positives | `publish-allowlist.txt` | exact `path:sha256(line)` entries |
| Image conformance | `verify_image.sh`, `canary.sh` | Cortex contract v1; uid 65532, read-only root, secrets `/run/secrets` |
| Docs doctests | `doctest_docs.sh` | `bash doctest` blocks in docs/operator.md, docs/miner.md; `doctest-skip:prod` never run; `DOCTEST_TIMEOUT` 1800 |
| Doc number claims | `check_doc_numbers.py` | `0.5{ev:path.json#key}` tags; not in CI |
| Freeze hashes | `preservation_check.py` | `--out` / `--compare` |
| Mechanism tables | `gen_mechanism_tables.py` | `> experiments/results/mechanism_tables.json` |
| Network v2 proof | `network_service_proof.py` (2488) | mypy-checked module |
| GPU network ops | `network_gpu_{operation,prepare,seed,qualification}.py`, `network_authority_snapshot.py`, `check_gpu_evidence.py` | `operation` mypy-checked |
| kind relay smoke | `relay_kind_smoke.py` (3936) | PEP 723 header; tests load by path |
| Vast.ai e2e | `vast_e2e.py` (stdlib only), `vast_e2e_remote.sh` | 2x RTX 5090 one-host harness |

## CONVENTIONS
- Only `network_service_proof` and `network_gpu_operation` are type-checked (`MYPYPATH=src:scripts:.`, `--explicit-package-bases`); add new script modules to `check.sh` if they must be.
- Script pydantic models: `ConfigDict(extra="forbid", frozen=True)`.
- `vast_e2e_remote.sh`: `set -uo pipefail` WITHOUT `-e` on purpose (later steps run after failure; rc in `steps.tsv`); determinism env `CUBLAS_WORKSPACE_CONFIG=:4096:8 CUDA_DISABLE_PTX_JIT=1 OMP_NUM_THREADS=1 PYTHONHASHSEED=0`.
- `vast_e2e.py` flow: credit check, worst-case cost admission, ONE create, watchdog, rescue, DELETE, parsed absence; `--dry-run` stops after cost.
- relay smoke documented run: `env OMP_NUM_THREADS=1 nice -n 19 taskset -c 0-3 flock /tmp/hypertrain-network-cpu.lock uv run --frozen python scripts/relay_kind_smoke.py --create --cluster hypertrain-v2-proof --cpus 0-3 --cpu-quota 4 --cleanup`.

## ANTI-PATTERNS
- Publishing with any secret-scan hit, or without out-of-band token approval + manual file-list review.
- Allowlisting broadly: entries are exact line hashes, strict rules only.
- Agents/lane implementations executing `relay_kind_smoke.py` ("Never executed by lane implementation"); kind provider fallback (must be Docker).
- Printing/journaling the Vast API key (`--key-file` only).
- Treating podman as canary-equivalent (rejects `uid=` tmpfs option).
