# ADAPTATION

AskSwarmRegistry ships in an `evm_contracts` launch on Ethereum (chain id 1). This file lists every
change made to the project for this launch, the file it touches, and the rule or audit finding that
required it. The earlier launch of this registry (launch #2088) and its adaptation notes are in
[contracts/ADAPTATION.md](contracts/ADAPTATION.md); the code here starts from the version that
launch accepted.

## What the launch deploys

One contract, `AskSwarmRegistry` (`src/AskSwarmRegistry.sol`), constructor
`(address intake, address imd, address signer)` =
(`0xa43e6f75ee006411f79ac1c84120606c2330de82`, `0xd34a99bc0f67ae1bbd63c660e6d0b0dd03e263b7`,
`0x5598aa9146215bc13eb26f2c692ad1461fd32982`), all from the brief. The constructor is nonpayable,
takes three static arguments, makes no external call and needs no contract to exist, so it deploys on
the launch's empty rehearsal chain. There is no owner, admin, initializer, proxy, upgrade path, ETH path
or loop, so `$owner` is not used. No DELEGATECALL, CALLCODE or SELFDESTRUCT; runtime about 8.4 KB.
The Foundry project is at the repository root (`foundry.toml`, `src/`, `test/`, `script/`, vendored
forge-std under `contracts/lib/`), as the project arrived; `bytecode_hash = "none"` was already set.
No build configuration or dependency was changed.

## Changes

### 1. Forge build output is no longer tracked (repository, no source change)

- Files: `contracts/out/**`, `contracts/cache/**`, `contracts/broadcast/**` removed from the index
  (46 files). They were already listed in `.gitignore`; the tree tracked them anyway.
- Rule: the verifier rejected the previous attempt with `path_violation`: contracts are compiled from
  source, and a tree may not carry what forge would read in their place. Nothing else about the
  repository layout changed.

### 2. Audit finding 1b549341 (low), reproduced and fixed: a settled ask could be delivered again

- File: `src/AskSwarmRegistry.sol`, `onOracleResult` and `onFailure`.
- Reproduced with the audit's proof: after ask A was answered "CHF", the writer completed a second
  open ask B with arguments naming A and "EUR". The live Intake v2 binds a completion's arguments to
  its request id only for status 1 and 2, so the registry's `onOracleResult(A, …)` ran again,
  rewrote A's answer, and with an empty signature flipped A to `Unverified` while `latestFor` still
  pointed at it.
- Change: both hooks now revert `AlreadySettled(requestId)` unless the ask is `Pending`. The status
  shares the slot already loaded for the `UnknownAsk` check, so the callback's gas is unchanged
  within measurement noise. The header invariant now states what the contract enforces rather
  than what it assumed of Intake.
- Tests: `test_settledAskIsNotRewrittenByAnotherRequestsCompletion` (the audit's scenario and its
  zero-signature variant, through an Intake model that forwards status-0 arguments unchecked),
  `test_bothHooksRefuseEverySettledStatus` (Answered, Unverified, over-cap Unverified, Refused,
  NoResult, for both hooks, with the record byte-identical afterwards),
  `test_settledAskRefusesRedeliveryThroughAnotherOpenAsk` (through the mock etched at the real Intake
  address), and the invariant handler `replaySettledThroughOpenRequest`, which replays random settled
  ids through random open asks during the campaign and tracks the asks that Intake then leaves
  completed-but-undelivered.

### 3. Audit finding d8368ddd (low), reproduced and fixed: attestations issued in the future verified

- File: `src/AskSwarmRegistry.sol`, `onOracleResult` and the new constant `ISSUED_AT_TOLERANCE`.
- Reproduced: an attestation with `issuedAt` a year ahead and `expiresAt` a year and a day ahead was
  stored `verified` and indexed, locking the question for far longer than the plane's 30-day cap.
  The protocol's `OracleAttestationConsumer` refuses `issuedAt > block.timestamp + 5 minutes`.
- Change: `verified` additionally requires `a.issuedAt <= block.timestamp + ISSUED_AT_TOLERANCE`
  (5 minutes, the protocol consumer's value). A pre-dated delivery is recorded `Unverified` and never
  indexed, like an expired one.
- Tests: `test_issuedAtToleranceBoundary` (exactly 5 minutes ahead is verified, one second more is
  not, the audit's one-year case), `testFuzz_verifiedRequiresIssuedAtWithinTolerance`, and
  `testFuzz_indexedAnswersAreMajorities` now also fuzzes `issuedAt` ahead of the clock.

### 4. Audit finding 77599b52 (info), reproduced and fixed: unanswered records hashed as an empty answer

- File: `src/AskSwarmRegistry.sol`, `Stored.stored` and `_read`.
- Reproduced: `get(id)` for a Pending, Refused, NoResult or over-cap ask, and for an id never asked,
  returned `answerHash == keccak256("")`, indistinguishable from an answer that was empty bytes.
- Change: `Stored` gains a `stored` flag in the free bytes of its third slot (written in the same
  SSTORE, so no gas change; the three-slot layout is kept), and `_read` returns a zero `answerHash`
  and empty `answer` when nothing was stored. A stored empty answer still hashes as `keccak256("")`.
- Tests: `test_recordsWithoutAStoredAnswerReportZeroAnswerHash`, `test_unknownReadsDoNotLookAnswered`
  (now asserts the hash), and the invariant campaign asserts a zero hash for every Pending ask.

### 5. Audit finding f2768f2b (info): no code change

Each point was checked; none is a defect in the registry's own logic.

- Intake's owner can retire the action (`ask` reverts `NotSold`), lower the callback gas, or rotate
  the writer or signer. These are the trust assumptions the contract header states; the registry
  has no owner to react, by the requester's design, and a successor is a new deployment.
- `SIGNER` is compared by `ecrecover`, so a rotation to an ERC-1271 signer stops new answers
  verifying. Documented in the header; unchanged.
- `DOMAIN_SEPARATOR` is cached at construction with `block.chainid`. After a chain-id-changing fork
  answers fail to verify rather than being accepted. Unchanged.
- The one-argument `answerOf` returns the held answer unfiltered; readers that need freshness or a
  minimum panel use the three-argument overload or `isCurrent`. Documented; unchanged.
- The brief names `contracts/src/AskSwarmRegistry.sol`; the tree has `src/AskSwarmRegistry.sol` with
  `foundry.toml` at the root (`src = "src"`), as the previous launch accepted. Kept.
- The brief's "~105k gas" for the callback predates the review changes. Measured from the test frame
  in this tree: 146,342 gas for a bytes32 answer and 133,966 for a warm 352-byte answer, both under
  the 150k the test asserts and under Intake's 200k stipend. Delivery through the Intake model
  reports `delivered == true`, including the cold 4096-byte worst case.
- Launch-forbidden features: none found. IMD is an OpenZeppelin ERC-20 without transfer hooks, so
  `ask` is not reentrant and leaves zero allowance and balance.

## Test and documentation changes

- `test/AskSwarmRegistryCallbacks.t.sol`: the five tests listed above; the retry test's comment no
  longer says the registry has no Pending guard.
- `test/AskSwarmRegistry.t.sol`: one new test and the extended fuzz test listed above.
- `test/AskSwarmRegistryInvariant.t.sol`: the `replaySettledThroughOpenRequest` handler, a `stranded`
  flag on expected asks, and matching checks in `assertRecordsAndIndexes`.
- `test/TESTING.md`, `README.md`: describe the new behaviour and this launch.
- `contracts/launch.json` is the manifest the previous launch accepted and is left as it is; the
  manifest step writes this launch's own.

All 70 tests pass offline with the verifier's fuzz seed; the three audit proofs fail against the
contract as it arrived and pass against this one. The `test/AskSwarmRegistry.t.sol` suite was also
run on a mainnet fork (`forge test --match-path test/AskSwarmRegistry.t.sol --fork-url <rpc>`, public
RPC, 17 Oct 2026): all 20 passed against the live Intake v2 and IMD. There the real `Intake.complete`
reports `delivered == true` for a verified answer under its stipend, and `delivered == false` when a
second open ask carries a settled id, which confirms the finding's premise against the live contract.
Callback gas measured on the fork: 146,366 (bytes32 answer), 133,966 (warm 352-byte answer).
