# src/hypertrain/protocol

## OVERVIEW
Wire contract: pydantic messages, signed envelopes, JCS, hashing, keys, JSON-schema export. Score 21: most-imported package (`jcs.canonicalize` ~80, `envelope_v2` ~55, `Keypair` ~53).

## WHERE TO LOOK
| Task | Location | Notes |
|------|----------|-------|
| Canonical JSON | `jcs.py` | RFC 8785; `MAX_SAFE_INT = 2**53-1`; `canonicalize(value, allow_float=...)` |
| v1 messages (frozen) | `messages.py` | `Receipt`, `Finalize`, `RunManifest`, `LeafPreimage`, `Rollback`, `QUICKNET_GENESIS`, `f32hex`/`f32val` |
| v2 messages | `messages_v2.py` | `RunManifestV2`, `RoundOpenV2`, `CommitV2`, `IslandJobV1`, `AuditJobV2`, `MESSAGE_TYPES_V2` |
| v1 envelope | `envelope.py` | `body_digest`; `run_id` appended after spec fields (ponytail) |
| v2 envelope | `envelope_v2.py` | `seal`, `load_json`, `tape_signing_message`, `replay_key`, `Intake`, `SIGNER_BOUND_TYPES`, `MAX_DEPTH`/`MAX_ARRAY` |
| Relay wire | `relay_envelope.py`, `relay_messages.py` | `RelayRegistryV1`, `RelayNetworkManifest`, `UploadGrant`, `RetrievalRequest` |
| Hashes, Merkle | `hashing.py` | `sha256_hex`, `MerkleTree`, `tensor_hash` |
| Ed25519 keys | `keys.py` | `Keypair`, `decode_hotkey` |
| Example manifest | `example.py` | `example_manifest` (~31 importers, test fixture base) |
| Schema export | `schema.py` (v1), `schema_v2.py` | v1 output pinned in `docs/schemas/` |

## CONVENTIONS
- Envelopes domain-separated (`DOMAIN`, `VERSION`); body hotkey must equal signer.
- Relay types carry `V1` suffix even inside v2 protocol.
- drand quicknet constants pinned in both `messages.py` and `beacon/drand.py`: change together.

## ANTI-PATTERNS
- Raw floats in v2 bodies ("raw floats forbidden; use f32 hex"); floats in any signed preimage.
- bool aliasing int (`test_bool_cannot_alias_int_round`); unknown manifest fields.
- Changing a v1 model without regenerating: `docs/schemas/*.json` must equal `schema.render()` byte-for-byte (`python -m hypertrain.protocol.schema docs/schemas`). Schema v2 never writes into the frozen v1 dir.
- Trusting `example.py` outer defaults (`topk_frac=1.0, bits=8, ef_beta=0.0`): marked UNSOURCED/provisional.
- Accepting replayed, expired, wrong-`run_id`, or signer-swapped envelopes; >1 dispute decision per verdict.

## TESTS
`tests/protocol/` (JCS RFC 8785 vectors, schema regen, `test_network_v2.py` 1078 LOC).
