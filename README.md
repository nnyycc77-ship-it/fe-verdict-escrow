# Verdict Escrow for Agent Work

A small Fe contract suite where a poster escrows payment for a job, the worker records a delivery hash, and a named evaluator's verdict, bound to that hash, decides whether the escrow is released, refunded, or sent back for another round.

## What it does

Agent work is usually paid on trust: the poster hopes the delivery is good, the worker hopes the poster pays. This contract narrows both hopes to one named evaluator and a rule every party can read on chain.

1. A poster creates a job by funding the escrow in the same call, naming the worker and the evaluator up front. The worker accepts the terms by delivering and is bound to nothing before that.
2. The worker records the delivery hash of the finished work.
3. The evaluator records a verdict against that exact hash: pass (1), revise (2), or fail (3).
4. A pass entitles the worker to the escrow. A fail entitles the poster to a refund. A revise holds the funds and reopens the job for a new delivery and a new verdict.
5. Every move restarts the job's clock: creation, each delivery, and each revise verdict. The verdict window is the earliest moment the poster may reclaim the escrow on an unsettled job, and after it the first call to land decides: a verdict can still arrive any time after the window, and it stands unless the poster has already reclaimed. The reclaim covers a worker who never delivers, a worker who abandons the job after a revise, and an evaluator who stays silent after a delivery. Its other half is harder: a worker who delivers to an evaluator that stays silent is unpaid and has no call of their own, so the reclaim is a poster side exit only.
6. Whoever a settlement entitles collects the funds with a claim, to an address of their choice. The verdict fixes the entitlement on record; collecting is a separate step, so a settlement can never be blocked by a receiver that rejects transfers.

Every verdict is counted in a per job sequence that only increases, and each verdict only counts when it names the delivery hash currently on record, so an old verdict can never be replayed against a newer delivery. Many jobs coexist in one deployment, each with its own parties, amount, hash, and sequence. Once a job is settled, its terms cannot change; the only remaining action is the entitled party collecting.

## Why Fe

The point of this entry is how the release logic reads in Fe, and three language properties carry it:

- **Explicit effects.** Every handler declares exactly what it may touch. Reading the signatures is most of the audit: `Verdict` declares storage writes and the log, but no `Call` effect, so a verdict can change who is entitled and can never itself move money. In `VerdictEscrow`, only `Claim` declares the `Call` effect, and it is the only handler that sends value.
- **Least privilege, stated exactly.** Storage is split into three maps by privilege, and the compiler enforces the split through the effect lists. The entitlement map is written only by the `Verdict` and `ReclaimTimeout` handlers, the only two handlers that declare it mutable, so a settlement is written only where a settlement is decided. The paid flag lives in its own map, written only by `Claim`, and `Claim` declares the core record and the entitlement map read only, so the claim cannot change a verdict: it can read who is entitled, check the amount, and collect, and nothing more. Only `Claim` can send value, and the read only views cannot write at all. What the compiler does not check is role separation and the order of states. Role separation (the worker cannot judge, the evaluator cannot deliver) stays with caller checks inside the handlers. The order of states stays with the state asserts: `Create` and `Deliver` hold write access to the core record, which carries the state field, so a settled job staying settled is held by those asserts, not by the compiler. Both are ordinary asserts that the tests exercise.
- **Solidity compatible ABI.** Message selectors are computed at compile time with the `sol(...)` helper, and `fe build` emits deploy bytecode, runtime bytecode, and a Solidity compatible ABI JSON, so the contract drops into existing EVM tooling, while the source stays in Fe's effect discipline.

## Negative proofs you can run

The strongest claim above is that a verdict can never move money, because the `Verdict` handler declares no `Call` effect. You do not have to take this README at its word. In a scratch copy of this repo, add these four lines to the settling branch of the `Verdict` handler in `src/lib.fe`, right after the entitlement is written:

```fe
assert!(
    call.send_value(addr: to, gas: 100000, value: amount).success(),
    "Payout failed",
)
```

Then run `fe build`. The compiler refuses, with exit code 1:

```
error[8-0012]: undefined variable
    ┌─ src/lib.fe:238:21
    │
238 │                     call.send_value(addr: to, gas: 100000, value: amount).success(),
    │                     ^^^^ undefined variable `call`
```

The `call` capability is simply not in scope inside a handler that does not declare it, so the send cannot be smuggled in. The only way to make that line compile is to add `mut call` to the `Verdict` handler's own `uses` list, in the signature, where any reader can see it. The shipped tree contains no such line and no such declaration; the scratch copy is the whole experiment, and deleting it restores the build.

The storage split carries the matching proof, for the round's other central claim: the claim cannot change a verdict. In a scratch copy of this repo, add this one line to the `Claim` handler in `src/lib.fe`, right after the paid flag is written:

```fe
entitlements.set(key: job_id, value: to)
```

Then run `fe build`. The compiler refuses, with exit code 1:

```
error[8-0066]: mutable borrow requires a mutable place
    ┌─ src/lib.fe:304:13
    │
291 │             entitlements,
    │             ------------ `entitlements` does not provide mutable access
    ·
304 │             entitlements.set(key: job_id, value: to)
    │             ^^^^^^^^^^^^ cannot borrow as `mut`
```

`Claim` declares the entitlement map without `mut`, so the write cannot borrow it as mutable. The only way to make that line compile is to declare the map mutable in the `Claim` signature, in the open, where the effect list states it. The shipped tree declares it read only there; again the scratch copy is the whole experiment, and deleting it restores the build.

## Architecture

One contract, `VerdictEscrow` in `src/lib.fe`, one ingot, no dependencies. The same file also carries a small test helper contract, `PosterProxy`, used only by the test suite.

- **State.** Three storage maps, split by who may write them. The core record lives in a `StorageMap<u256, Job>`: poster, worker, evaluator, amount, current delivery hash, state, verdict sequence, and the time of the last move. The entitled payee lives in a `StorageMap<u256, Address>`, written only by the verdict and reclaim handlers. The paid flag lives in a `StorageMap<u256, u256>`, written only by the claim handler. Job ids are handed out from a counter starting at 1; id 0 reads as an empty job, with no entitlement recorded and nothing paid.
- **State machine.** A job is open (1) while it waits for a delivery, delivered (2) while it waits for a verdict, and settled (3) forever after. Deliveries are accepted only in state 1, verdicts only in state 2, reclaims in either unsettled state once the window has passed since the last move, and claims only in state 3 before payment.
- **Messages.** `createJob(address,address)` (payable), `deliver(uint256,uint256)`, `verdict(uint256,uint256,uint256)`, `reclaimTimeout(uint256)`, `claim(uint256,address)`, and read only views `stateOf`, `amountOf`, `deliveryHashOf`, `verdictSeqOf`, each taking the job id.
- **Events.** `JobCreated`, `Delivered`, `VerdictRecorded` (with kind, hash, and sequence), `Settled` (with outcome and entitled party), and `Claimed` give an off chain observer the full life of a job without reading storage.
- **Timeout.** The verdict window is a deployment parameter, set in the constructor as a `Duration`, and it runs from the job's last move: creation, each delivery, and each revise verdict each restart it, so every round carries its own deadline.

## Build and test

Exact toolchain: **Fe 26.6.0** (`fe 26.6.0 (5b3f96a)`). Get the exact release from the Fe project's GitHub releases page (github.com/argotorg/fe, release 26.6.0, published 2026-11-10): download the Linux x86_64 binary `fe_linux_amd64` (55,932,656 bytes, SHA 256 697081cb9265bd1be7a5d33e6bb0c8559ee5ff542abab9a046540fc151b4edaa), verify the checksum, and put the binary on your path as `fe`.

```sh
fe build   # writes out/VerdictEscrow.bin, out/VerdictEscrow.runtime.bin, out/VerdictEscrow.abi.json, and the same three outputs for PosterProxy
fe test    # runs the suite in src/tests.fe against the integrated EVM simulation
fe fmt     # the sources are formatted with fe fmt
```

The test suite is also the demo: it walks the complete life of several jobs end to end and asserts balances at each step. Twenty tests, all passing:

- `pass_releases_to_worker`, `fail_refunds_poster`: the two settlement paths, with the entitled party collecting.
- `revise_then_second_delivery_pass_releases`: a revise holds the funds, a second delivery lands, the sequence reaches 2, and the pass pays out.
- Timeout paths: `silence_timeout_lets_poster_reclaim` and `reclaim_before_timeout_reverts` cover the evaluator silence case after a delivery; `reclaim_undelivered_job_succeeds` and `reclaim_undelivered_before_window_reverts` cover a job that was never delivered; `reclaim_abandoned_after_revise_succeeds` covers a worker who never returns after a revise. Each is exercised with a zero length window (always expired) or a long window (never expired).
- `jobs_are_independent`: two jobs in one deployment settle independently.
- Proxy tests, where a forwarding contract (`PosterProxy` in `src/lib.fe`) creates and funds the job, so the poster is a different address from the test account that delivers and judges: `proxy_pass_pays_worker_not_poster` and `proxy_fail_refunds_poster_not_worker` check each settlement names the right payee, and `proxy_poster_claim_on_pass_reverts` with `proxy_worker_claim_on_fail_reverts` check that a claim by anyone but the entitled payee reverts.
- Guard tests, all expected reverts: `verdict_from_non_evaluator_reverts`, `delivery_from_non_worker_reverts`, `second_verdict_after_settlement_reverts`, `second_claim_reverts`, `verdict_with_wrong_hash_reverts`, `verdict_before_delivery_reverts`.
- `claim_to_fe_contract_reverts`: the documentation test for the receiver finding in `FRICTION.md`. A claim that names a deployed Fe contract as its target reverts whole, because a runtime compiled with this toolchain refuses a plain transfer with empty calldata.

## Limitations

- **Claim based payout.** Settlement fixes the entitlement on record; the entitled party collects in a second call. This is deliberate (a settlement cannot be blocked by a refusing receiver, and no party is ever forced a transfer), but it does mean payout is two steps, not one.
- **Claims pay plain addresses.** The claim sends value with a plain transfer that forwards 100,000 gas, well beyond the bare transfer stipend, so ordinary receivers are fine. A payee contract whose code refuses plain transfers with empty calldata cannot be paid this way: the claim reverts whole, nothing is lost, and the payee can name another address and claim again. Fe contracts compiled with this toolchain are in that refusing class today, because their dispatchers only answer calls that carry a message selector.
- **One evaluator per job, chosen by the poster.** There is no evaluator staking, appeal, or panel logic. The verdict is only as honest as the evaluator the poster named, which matches small job escrows and does not try to be a dispute court.
- **Revise rounds are uncapped.** A revise restarts the clock each round, so an active evaluator and worker can loop indefinitely; the funds only leave when a verdict settles the job or the poster reclaims after a silent window. The poster is never locked in, but a cooperative pair can keep a job open.
- **The delivery hash is opaque.** The contract binds verdicts to a 256 bit hash and stores nothing else about the work. What the hash covers, and how the evaluator inspects the delivery, live off chain by design.
- **Native value only.** Escrows are funded in the chain's native asset. There is no token escrow in this version.
- **Timeout tested at the boundary.** The Fe test harness offers no clock control in the API available to us, so the timeout is tested with a zero length window (always expired) and with a long window (never expired), rather than by advancing time mid test.

## Declarations

- **Preexisting work.** The escrow and verdict design comes from our own earlier design work in other stacks: a NEAR testnet escrow gate demo and a written verdict layer design for agent work (the Trust Toll research line), both from October and November 2026. Those designs are ideas and research, in other languages; no code from them is reused here. The starter project's time locked vault, published by the contest organisers as a template, was used as a syntax reference for Fe idioms.
- **Written inside the window.** All Fe code, tests, and documentation in this repository were written during the contest build window, 2026-11-03 to 2026-11-24.
- **AI assistance.** This project was built with AI assistance, as the contest expressly welcomes: an AI assistant drafted the contract, the tests, and this documentation under the direction of the human contact. A second, independent AI reviewer read every line, and its findings were applied in the fix rounds recorded in the commit history. The build and the full test suite run on each push in the repository's own workflow. Every claim in this README was rechecked against the final code, and the full suite was run green (`fe build` exit 0, `fe test` 20 passed and 0 failed) before the entry was submitted. The assistance is disclosed here and in the entry registration; a human contact is responsible for the entry and can explain the code.
- **License.** MIT, see `LICENSE`.
