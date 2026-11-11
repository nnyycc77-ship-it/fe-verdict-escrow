# Verdict Escrow for Agent Work

A small Fe contract suite where a poster escrows payment for a job, the worker records a delivery hash, and a named evaluator's verdict, bound to that hash, decides whether the escrow is released, refunded, or sent back for another round.

## What it does

Agent work is usually paid on trust: the poster hopes the delivery is good, the worker hopes the poster pays. This contract replaces both hopes with a rule that all three parties can read before any money moves.

1. A poster creates a job by funding the escrow in the same call, naming the worker and the evaluator up front.
2. The worker records the delivery hash of the finished work.
3. The evaluator records a verdict against that exact hash: pass (1), revise (2), or fail (3).
4. A pass entitles the worker to the escrow. A fail entitles the poster to a refund. A revise holds the funds and reopens the job for a new delivery and a new verdict.
5. If the evaluator never acts, a silence timeout protects the poster: once the verdict window after a delivery has passed, the poster can reclaim the entitlement.
6. Whoever a settlement entitles collects the funds with a claim, to an address of their choice. The verdict fixes the entitlement on record; collecting is a separate step, so a settlement can never be blocked by a receiver that rejects transfers.

Every verdict is counted in a per job sequence that only increases, and each verdict only counts when it names the delivery hash currently on record, so an old verdict can never be replayed against a newer delivery. Many jobs coexist in one deployment, each with its own parties, amount, hash, and sequence. Once a job is settled, its terms cannot change; the only remaining action is the entitled party collecting.

## Why Fe

The point of this entry is how the release logic reads in Fe, and three language properties carry it:

- **Explicit effects.** Every handler declares exactly what it may touch. Reading the signatures is most of the audit: `Verdict` declares storage writes and the log, but no `Call` effect, so a verdict can change who is entitled and can never itself move money. Only `Claim` declares the `Call` effect, and it is the only handler that sends value.
- **Least privilege by construction.** Role checks sit inside handlers whose effect lists are already minimal. The evaluator's handler cannot pay anyone, the worker's handler cannot judge, and the claim handler cannot change a verdict. The compiler enforces the effect lists, so these boundaries are checked, not just documented.
- **Solidity compatible ABI.** Message selectors are computed at compile time with the `sol(...)` helper, and `fe build` emits deploy bytecode, runtime bytecode, and a Solidity compatible ABI JSON, so the contract drops into existing EVM tooling, while the source stays a fraction of the size of the equivalent hand checked Solidity.

## Architecture

One contract, `VerdictEscrow` in `src/lib.fe`, one ingot, no dependencies.

- **State.** Jobs live in a `StorageMap<u256, Job>`. A `Job` records poster, worker, evaluator, amount, current delivery hash, state, verdict sequence, delivery time, entitled payee, and a paid flag. Job ids are handed out from a counter starting at 1; id 0 reads as an empty job.
- **State machine.** A job is open (1) while it waits for a delivery, delivered (2) while it waits for a verdict, and settled (3) forever after. Deliveries are accepted only in state 1, verdicts and timeout reclaims only in state 2, claims only in state 3 before payment.
- **Messages.** `createJob(address,address)` (payable), `deliver(uint256,uint256)`, `verdict(uint256,uint256,uint256)`, `reclaimTimeout(uint256)`, `claim(uint256,address)`, and read only views `stateOf`, `amountOf`, `deliveryHashOf`, `verdictSeqOf`, each taking the job id.
- **Events.** `JobCreated`, `Delivered`, `VerdictRecorded` (with kind, hash, and sequence), `Settled` (with outcome and entitled party), and `Claimed` give an off chain observer the full life of a job without reading storage.
- **Timeout.** The verdict window is a deployment parameter, set in the constructor as a `Duration`. The deadline for a verdict is the delivery time plus this window.

## Build and test

Exact toolchain: **Fe 26.6.0** (`fe 26.6.0 (5b3f96a)`).

```sh
fe build   # writes out/VerdictEscrow.bin, out/VerdictEscrow.runtime.bin, out/VerdictEscrow.abi.json
fe test    # runs the suite in src/tests.fe against the integrated EVM simulation
fe fmt     # the sources are formatted with fe fmt
```

The test suite is also the demo: it walks the complete life of several jobs end to end and asserts balances at each step. Twelve tests, all passing:

- `pass_releases_to_worker`, `fail_refunds_poster`: the two settlement paths, with the entitled party collecting.
- `revise_then_second_delivery_pass_releases`: a revise holds the funds, a second delivery lands, the sequence reaches 2, and the pass pays out.
- `silence_timeout_lets_poster_reclaim` and `reclaim_before_timeout_reverts`: the timeout path, exercised with a zero length window, and its guard.
- `jobs_are_independent`: two jobs in one deployment settle independently.
- Guard tests, all expected reverts: `verdict_from_non_evaluator_reverts`, `delivery_from_non_worker_reverts`, `second_verdict_after_settlement_reverts`, `second_claim_reverts`, `verdict_with_wrong_hash_reverts`, `verdict_before_delivery_reverts`.

## Limitations

- **Claim based payout.** Settlement fixes the entitlement on record; the entitled party collects in a second call. This is deliberate (a settlement cannot be blocked by a refusing receiver, and no party is ever forced a transfer), but it does mean payout is two steps, not one.
- **One evaluator per job, chosen by the poster.** There is no evaluator staking, appeal, or panel logic. The verdict is only as honest as the evaluator the poster named, which matches small job escrows and does not try to be a dispute court.
- **The delivery hash is opaque.** The contract binds verdicts to a 256 bit hash and stores nothing else about the work. What the hash covers, and how the evaluator inspects the delivery, live off chain by design.
- **Native value only.** Escrows are funded in the chain's native asset. There is no token escrow in this version.
- **Timeout tested at the boundary.** The Fe test harness offers no clock control in the API available to us, so the timeout is tested with a zero length window (always expired) and with a long window (never expired), rather than by advancing time mid test.

## Declarations

- **Preexisting work.** The escrow and verdict design comes from our own earlier design work in other stacks: a NEAR testnet escrow gate demo and a written verdict layer design for agent work (the Trust Toll research line), both from October and November 2026. Those designs are ideas and research, in other languages; no code from them is reused here. The starter project's time locked vault, published by the contest organisers as a template, was used as a syntax reference for Fe idioms.
- **Written inside the window.** All Fe code, tests, and documentation in this repository were written during the contest build window, 2026-11-03 to 2026-11-24.
- **AI assistance.** This project was built with AI assistance, as the contest expressly welcomes. The assistance is disclosed here and in the entry registration; a human contact is responsible for the entry.
- **License.** MIT, see `LICENSE`.
