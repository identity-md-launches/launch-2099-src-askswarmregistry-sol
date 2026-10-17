# AskSwarmRegistry test coverage

The tests run offline with the repository's existing forge-std dependency. Foundry configuration is
unchanged. Run `forge build` and `forge test` from the repository root. Any fuzz seed passes; the
second launch's verifier used:

```sh
FOUNDRY_FUZZ_SEED=0xd094d872360bfdb7dab18e96fc214e6614b324111bad9f9dd12ad8e4b5d75c41 forge test
```

- `AskSwarmRegistryPayments.t.sol` covers exact fee forwarding, changing prices, body limits,
  insufficient funds/approval, malformed or unsuccessful ERC20 calls, no-return tokens, Intake
  rejection before and after payment, donation isolation, and rejected ETH payments. Reverts must
  leave balances, approvals, request records, and counters unchanged.
- `AskSwarmRegistryCallbacks.t.sol` covers tampering with every signed field, wrong signers and
  domains, invalid signature lengths/recovery values, a valid false answer, distinct oracle/intake
  identifiers, byte-exact short answers, nonzero ABI padding, long-answer hashes and events, the
  4,096/4,097-byte boundary, both failure statuses, access checks, filtered reads, expiry transitions,
  and both dimensions of panel strength. Delivery uses a 200,000-gas call. From the second launch's
  audit: a settled ask refuses re-delivery on both hooks whatever Intake relays (including a status-0
  completion of another open request that names the settled id), `issuedAt` more than 5 minutes
  ahead of the chain clock is recorded `Unverified`, and records with no stored answer read back
  with a zero `answerHash`.
- `AskSwarmRegistryInvariant.t.sol` runs 256 sequences of 96 calls with three funded callers.
  Nine handlers exercise successful and reverted asks, donations, signed/invalid/expired/oversized
  answers, refusals, time changes, unauthorized callbacks, transport retries, and replays of a
  settled id through an open request. Ghost accounting tracks each caller's payments and donations.
  Checks cover conservation, exact fee forwarding, zero remaining Intake approval, request counts,
  immutable settled records under any writer behaviour, pending asks reading as unanswered, and
  coherent majority-only question indexes. Seeded setup reaches meaningful states even before
  random calls begin. Unexpected handler reverts fail the campaign.
- Existing `OracleConsumerConformance.t.sol` retains the independent protocol digest/signature
  vector. New signature tests complement that vector; signing the registry's own digest alone
  would not prove protocol compatibility.

The new `helpers/RegistryTestSupport.sol` models the external payment token and Intake. Intake's
body-size checks, forwarding, request-id construction, failure-argument binding, and one-shot
callback behavior were checked against its
[verified Ethereum source](https://eth.blockscout.com/address/0xa43e6F75ee006411F79Ac1C84120606C2330DE82?tab=contract).
The model intentionally exposes test controls for price changes and payment failures. It does not
model offchain JSON screening, panel formation, or the operational signer. Opaque test bodies are
transport fixtures, not production oracle questions.

The existing `AskSwarmRegistryTest` also passed against the live Intake and IMD on an Ethereum
fork at block 26,213,316. It remains offline by default; a separately supplied RPC can enable its
fork path. No endpoint or credential is embedded in the new tests.

The findings that campaign reported (settled-record overwrite through another request's completion,
future issuance beyond the canonical clock tolerance) are fixed in the contract and covered by the
tests above; see `ADAPTATION.md` at the repository root. The cached domain after a chain-id-changing
fork is unchanged: answers on the forked chain would fail to verify rather than be accepted.
