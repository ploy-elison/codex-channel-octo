# V50 / V52 Round 2 & XIN-2858 native startup endpoint continuation — A-B4/A-B5/A-B6/A-B7 blocking repairs, native startup U/E recovery compatibility, and Server `53b0dcc6` & CLI `424942e8` ABI alignment

This continuation implements the `ACTIVE v50` (`Owner2120905444695117824`), `ACTIVE v52` (`Owner2120917153384140800`), `ACTIVE 12:51 alignment — CURRENT ROUND 2 ONLY` (`Owner2120982929117974528` / `Owner2120953081876090880`, independent review `XIN-2834` verdict `7827f70f-94e7-4efc-b5b7-da7b7496f2db`, trigger comment `a25ee2e2-6960-4da7-b561-c38541da62ee`), `ACTIVE — XIN-2858 native startup repair only` (`Owner 2122964206335463424`, Charter receipt `65ac8fad-1356-4f0b-a2d2-fe55bab0111e`, preflight `b7a0d0d2-322c-41a0-a042-7c92a76f57d3`, trigger comment `776c4f93-6325-4cc5-9871-9d436cae7d67`), and `ACTIVE — Agent native U/E recovery compatibility only` (`Owner 2123015364148105216`, Charter receipt `a6e4d87e-ae08-43e5-a97b-53ddfa0513ea`, `XIN-2858` outcome `2079f6eb-02f8-4f14-911e-07141ea0cecf`, trigger comment `c672a597-0985-4ed8-97c3-259ec55a0907`) slices on `XIN-2795` (`2f4a34e9-2be3-48ac-99f0-38a30f077acd`) under parent charter `XIN-2323` (`17e81670-5b49-461d-9509-7c22ca127939`).

It delivers the bounded native startup & `U`/`E` recovery binding (`XIN-2858`) on top of the Round 2 blocking repairs (`[A-B4]`, `[A-B5]`, `[A-B6]`, `[A-B7]`) and Round 1 repairs (`[A-B1]`, `[A-B2]`, `[A-B3]`, shutdown drain order):
- **Endpoint (`XIN-2795` = `2f4a34e9-2be3-48ac-99f0-38a30f077acd`, Draft #102):**
  builds on `b0dfa1057b806ed49ac5096cd5dfade3af65a5cf` (`ploy-elison:xin-2795-admission-context-replay`).
- **Server (`XIN-2793` = `6a92a547-cb82-4c3d-ac6c-8f5529028ba4`, Draft #951):**
  `53b0dcc6d176a56746d9da3ab64a65c21c434348` (`handoff-v10.json`, `contract-manifest-v10.json`, and `remaining-producers-v8.json`).
- **CLI (`XIN-2794` = `229ad215-8526-4a04-8f5b-36f073f81a16`, Draft #234):**
  tested against pinned head `424942e82934a105ed011830d5a482cb0682bd49` (`cli-component-contract-v7.json`, `contract-manifest-v7.json`, `isolated-test-catalog-fixture-v1.json`).

Draft #102 remains on the `ploy-elison` fork targeting `main` (historical base `5dfde6cfd56621b0fbcf9ec7f310c7caa5135f34`; verified clean merge-tree against live upstream `main` `2f4d0521bcae9e15c008c29e290427de5813e8d2`). No sibling repository or task is modified.

## Round 2 blocking repairs (`[A-B4]`, `[A-B5]`, `[A-B6]`, `[A-B7]`) and XIN-2858 native startup & U/E recovery compatibility

1. **`[XIN-2858]` Authenticated native startup recovery binding, wire `U`/`E` stage & snapshot validation, and durable replay lineage verification (`src/permissions/endpoint-factory.ts:295-720, 1510-1555`, `src/permissions/replay-journal.ts:32-42, 130-265`).**
   - Removed the synthetic-only `syntheticRecoveryFence` / `PERMISSION_NATIVE_RECOVERY_FENCE_UNAVAILABLE` dependency from `createNativeEndpointDriver` and implemented authenticated native recovery against `GET /v1/bot/permission-bundle/v1?space_id=<space_id>` (`Authorization: Bearer bf_...`, `X-Space-ID: <space_id>`, `Cache-Control: no-store`, `Pragma: no-cache`, `redirect: 'error'`, `cache: 'no-store'`).
   - Validates authenticated Server `53b0dcc6` wire stages (`"U"` / `"E"`) and snapshot semantics independently across both planes (`invocation` and `capability`):
     - `state === "U"` requires `rules.length === 0`.
     - `state === "E"` allows `0..128` rules (`InvocationRule` with exact keys `rule_id`, `effect`, `caller`, `source`; `CapabilityRule` with exact keys `rule_id`, `effect`, `caller`, `source`, `capabilities`), enforcing strictly sorted/unique `rule_id`s, `caller.uids`, `source.conversation_types`, canonical `source.refs` (`dm`, `group` with boolean `include_descendants`, `thread`), and `capabilities`, plus snapshot-wide bounds (`selectors <= 4096`, `capabilities <= 1024`).
     - Normalizes wire `"U"` / `"E"` to `'unconfigured'` / `'configured'` only at the Agent boundary via `recoveredStages()` while verifying `bundle_digest` over the canonical wire JSON.
   - Rejects non-200 HTTP responses, compressed responses, responses missing `no-store` in `Cache-Control` or carrying `Age` / `X-Cache: HIT`, BOM/non-UTF8/oversize payloads (`> 262144` bytes), non-canonical JSON framing, mismatched `key` (`origin`, `bot_uid`, `space_id`), invalid `scope_incarnation`/`revision`/`context_epoch`/`catalog_version`/`catalog_digest`, mismatched `expectedCatalog` pin, unknown/malformed/inconsistent `invocation`/`capability` stages or rules, or mismatched cryptographic `bundle_digest` (`SHA-256` over canonical JSON without `bundle_digest`).
   - Added `ReplayJournal.bindLineage(lineage, options)` storing a keyed HMAC-SHA256 `lineage_mac` and metadata `checksum` in `permission_replay_lineage` (no raw identifiers in SQLite). On non-provisioning restarts (`provision: false`) or existing journals, validates journal metadata, HMAC key, and `scope_incarnation` lineage before coordinator/socket/model readiness and refuses missing, corrupt, stale, or wrong-incarnation state without auto-provisioning or evicting replay entries (`recoverInterrupted()` converts interrupted `'reserved'` rows to `'uncertain'`).
   - Enforces `binding.scopeIncarnation === recoveredLineage.scopeIncarnation` during `prepareInput` and `publish` (`PERMISSION_NATIVE_SCOPE_INCARNATION_MISMATCH`).
2. **`[A-B4]` `stdio` relay ticket `max_response_bytes` acceptance and preservation (`src/permissions/endpoint-factory.ts`, `src/permissions/worker-launch.ts`).**
   - `checkedTurnRelay` includes `'max_response_bytes'` in `allowedTicketKeys` in Go `json.Marshal` lexicographic key order between `'endpoint'` and `'peer_sha256'`, validated by `validRelayMaxResponseBytes` (`1..67108864`, accepting both positive integer `number` and positive integer decimal `string` `/^[1-9][0-9]{0,18}$/`).
   - `preparePrivateWorker` and `encodeGoRelayTicket` preserve `max_response_bytes` across staged-CA (`ca_file`) and digest (`ca_sha256`) ticket re-encoding so `octo --turn-relay` retains the per-turn response cap.
3. **`[A-B5]` HTTP `202 Accepted` zero-byte notification acknowledgements (`src/permissions/worker-launch.ts`).**
   - `postHttpTurnRelay` resolves `undefined` when `res.statusCode === 202 && totalBytes === 0` (such as Streamable HTTP `notifications/initialized`).
   - `startTurnRelayBridge` responds with `HTTP 202 Accepted` and `Content-Length: 0` when `postHttpTurnRelay` resolves `undefined`.
4. **`[A-B6]` `history_disclosure` permit validation aligned with `octo-server` `#/$defs/Token` schema (`src/permissions/endpoint-factory.ts`).**
   - `createNativeEndpointDriver.disclose` validates `primary.permit` against `octo-server`'s real `#/$defs/Token` pattern (`!/^[A-Za-z0-9_-]{1,512}$/.test(primary.permit)` in `pkg/permissionbundle/contracts/permission-native-facts-v1.schema.json`), accepting `octo-server`'s 43-character base64url HMAC-SHA256 permit without requiring an invented `pmt1-` prefix.
5. **`[A-B7]` Linux `bwrap --unshare-pid` isolation and configured HTTP relay pin propagation (`src/permissions/worker-launch.ts`, `src/permissions/endpoint-factory.ts`).**
   - `bwrapBaseArgs` in `createIsolatingWorkerLauncher` includes `'--unshare-pid'`.
   - `checkedTurnRelay` propagates and enforces configured `boot.hostConfig` `relay_ca`/`peer_ca`, `relay_ca_sha256`, `relay_cert_sha256`/`relay_peer_sha256`, and `relay_server_name` on HTTP `TurnRelay` (`tls`, `caSha256`, `peerSha256`) even when optional `open_relay` reply fields (`tls`, `ca_sha256`, `peer_sha256`) are omitted, and fails closed if any reply pin conflicts with configured host pins.

## Consumed sibling revisions and exact byte/hash inventory

### Server (`XIN-2793` = `6a92a547-cb82-4c3d-ac6c-8f5529028ba4`, Draft #951)
Head: `53b0dcc6d176a56746d9da3ab64a65c21c434348` (`XIN-2858` native single-writer lineage authority delivered)

| File | Bytes | SHA-256 |
| --- | ---: | --- |
| `pkg/permissionbundle/contracts/handoff-v10.json` | 10,103 | `6fa4c5aad2e4e1d30d3ca451b832c1f65897c5920e909962c305d2b817857c82` |
| `pkg/permissionbundle/contracts/contract-manifest-v10.json` | 4,216 | `2baf35e17437ee3aeaf1467e762bb6839a775bd1daac2273b905b96265ef7b05` |
| `pkg/permissionbundle/contracts/remaining-producers-v8.json` | 4,069 | `1c926424a547142640e117e62b3151f6702478a7fdd52952c25542e35d05dd91` |
| `pkg/permissionbundle/contracts/permission-native-facts-v1.schema.json` | 30,549 | `bf08d33e87ad1a4726116c97171b008394e6896bba4babf03762914b5ad0e643` |
| `pkg/permissionbundle/authority_fence.go` | 41,228 | `e7b4d3cb824b340f26c4f6c5cc679c4dc0e64393f16e5357c9feeae41001430b` |
| `pkg/permissionbundle/install.go` | 14,414 | `46c422683b4e238c9a35fc398288f7c8b908ad55c08596b065e1525e932cc513` |
| `docs/permission-server-convergence-v52-r3-startup.md` | 8,931 | `65e18171f10132abb50c267b97e2b3cd58c2cffb899f4c1994609f4af0a0d0f0` |

### CLI (`XIN-2794` = `229ad215-8526-4a04-8f5b-36f073f81a16`, Draft #234)
Tested pinned head: `424942e82934a105ed011830d5a482cb0682bd49` (`XIN-2858` isolated-test catalog startup delivered)

| File | Bytes | SHA-256 |
| --- | ---: | --- |
| `docs/permission/endpoint-abi-v1.json` | 3,923 | `d0b055d6070545c749a08dfb49cabbd7d6fa15758328c111c20d1d881d20c835` |
| `docs/permission/endpoint-abi-v2.json` | 11,539 | `49b2ed31a4c78a7a7cee3957ea60aa7c6370ac8103d2896d297025c24b17fcb2` |
| `docs/permission/contract-manifest-v7.json` | 3,737 | `5264830ae91aa10ca1d337b35884da850abc69595dc8dbac9900257f081b49ef` |
| `docs/permission/cli-component-contract-v7.json` | 4,755 | `d697552aa34c89d32fec13f8ec00a3948e0f5498e6fd2c371c054225afe30f97` |
| `docs/permission/isolated-test-catalog-fixture-v1.json` | 10,154 | `c1b4cacb40a58394fdb9c172c48f06a6f98c555fb29ad2d3f0c1b6ef113c90cc` |
| `internal/mcp/permission_host.go` | 40,356 | `15908cc939730ff2d4723b7ab921d27e8a8648b10242c50d5a1fe689cd84d91e` |
| `internal/mcp/permission_relay.go` | 13,482 | `825c2160c90486a7be3da3ff85890e7ab8785097b93812bff9310eb6b5955174` |
| `docs/permission/catalog-package-v2/manifest.json` | 4,934 | `ecc56c2ab0a2c520ab7e5203d2abab06671daf90c28d69f88fe1de8d154109fc` |
| `docs/permission/source-slot-witness-v5.json.gz` | 12,562 | `0c5da173bd0f53c64562a70a5c5feebc955ad111bbe84ed11f5588c08b4b9c0b` |

## Residual classification (`DELIVER`)

1. **`code defect`:** None open on `XIN-2795` (`[A-B1]`–`[A-B7]` and `XIN-2858` native startup & `U`/`E` recovery compatibility closed).
2. **`sibling synchronization`:** Pinned `octo-server` (`53b0dcc6d176a56746d9da3ab64a65c21c434348`) and `octo-cli` (`424942e82934a105ed011830d5a482cb0682bd49`) remain unchanged; endpoint `U`/`E` recovery compatibility is verified against both exact pins.
3. **`independent review`:** Under Charter receipt `a6e4d87e-ae08-43e5-a97b-53ddfa0513ea`, after verified Agent delivery and containment feasibility on `XIN-2858`, PjM proceeds directly to isolated deployment and full independent black-box QA.
4. **`product decision`:**
   - Owner decision under `XIN-2323` (`17e81670-5b49-461d-9509-7c22ca127939`) / `XIN-2830` (`2fd1cf39-ab4d-4da5-aaa0-75161eb9ae52`) on the already-delivered unapproved memo `4831fa26-b5de-4dc0-a3dc-79477c891f91` (plus report `edd9ad84-f10b-4e1e-9161-19a598bbb153` and overriding addendum `28585cc5-4cc3-47e0-b3a5-17f81ac55b75`).
   - Owner decision under `XIN-2323` (`17e81670-5b49-461d-9509-7c22ca127939`) / `XIN-2794` (`229ad215-8526-4a04-8f5b-36f073f81a16`) on A1 production `ResponseBudgets` default values, A2/A3 status, and pinning candidate semantic digest `15c7ca2bf6df703979df529afbca6828f116b2463b1a818456edfad80bcad799` into `supported_catalog_digest`.
5. **`later deployment or tester gate`:**
   - `octo-server` (`XIN-2793` = `6a92a547-cb82-4c3d-ac6c-8f5529028ba4`) production `ServedCatalogs` startup activation (`RegisterBotPermissionRoutesFromEnv` / `LoadVerifiedCatalogRegistryFromEnv` remain fail-closed `503` until an Owner-approved `supported_catalog_digest` is pinned and configured).
   - Post-approval multi-service staging/deployment and tester gates (`V-A`–`V-E` / `V-D2`).
