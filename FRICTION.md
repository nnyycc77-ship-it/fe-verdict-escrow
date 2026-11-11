# Friction log

What this file is: the points where Fe 26.6.0, the exact toolchain this repo builds with (`fe 26.6.0 (5b3f96a)`), surprised us while we built Verdict Escrow, each with the smallest reproduction we ran. The reproductions were first run in scratch ingots during the build and were run again when this file was written. Nothing here blocks the contract. Two of the three findings shaped its design, and the README limitations carry the consequences a user of the contract feels.

## 1. A contract compiled with this toolchain cannot receive a plain transfer

What we saw. A claim that pays a deployed Fe contract reverts with the message "Payout failed", even with 100,000 gas forwarded to the send. The same flow pays a codeless address without trouble, and the rest of the suite, with claims to fresh addresses throughout, proves the send itself is sound.

Why it happens. Every runtime compiled in our sessions opens with a calldata size guard that routes empty calldata into the selector dispatch, where the zero word matches no selector and the call reverts. A plain transfer carries empty calldata, so it never reaches any code that could accept it. The compiled output has no entry point that answers a call carrying no selector.

Smallest reproduction. Two tiny contracts in a scratch ingot: a sink that does nothing but answer one message, and a sender that forwards what it receives with `send_value` at 100,000 gas, asserting on success exactly the way `Claim` does.

```fe
msg SinkMsg {
    #[selector = sol("ping()")]
    Ping {} -> u256,
}

pub contract Sink uses (ctx: Ctx) {
    recv SinkMsg {
        Ping {} -> u256 uses (ctx) {
            1
        }
    }
}

msg SenderMsg {
    #[selector = sol("pay(address)")]
    Pay { to: Address } -> Wei,
}

pub contract Sender uses (ctx: Ctx, call: mut Call) {
    recv SenderMsg {
        #[payable]
        Pay { to } -> Wei uses (ctx, mut call) {
            let amount = ctx.value()
            assert!(
                call.send_value(addr: to, gas: 100000, value: amount).success(),
                "Payout failed",
            )
            amount
        }
    }
}
```

```fe
use ingot::{Sender, SenderMsg, Sink}
use std::evm::ether

#[test(should_revert, balance = 10_000_000_000_000_000_000)]
fn plain_transfer_to_fe_contract_reverts() uses (evm: mut Evm) {
    let sink = evm.create2<Sink>(value: Wei::zero(), args: (), salt: 0)
    let sender = evm.create2<Sender>(value: Wei::zero(), args: (), salt: 0)

    let _: Wei = evm
        .call(
            addr: sender,
            gas: 300000,
            value: ether(1),
            message: SenderMsg::Pay { to: sink },
        )
}
```

The test carries the `should_revert` marker and passes: the transfer fails, the assert fires, and the sender call reverts whole.

The shipped suite carries the same fact as a documentation test, `claim_to_fe_contract_reverts` in `src/tests.fe`, where the entitled worker names the `PosterProxy` contract as the claim target and the claim reverts:

```fe
#[test(should_revert, balance = 10_000_000_000_000_000_000)]
fn claim_to_fe_contract_reverts() uses (evm: mut Evm) {
    let me = evm.address()
    let escrow = evm.create2<VerdictEscrow>(value: Wei::zero(), args: (days(7),), salt: 0)
    let proxy = evm.create2<PosterProxy>(value: Wei::zero(), args: (), salt: 0)

    let job: u256 = evm
        .call(
            addr: escrow,
            gas: 200000,
            value: ether(1),
            message: VerdictEscrowMsg::CreateJob { worker: me, evaluator: me },
        )
    let _: u256 = evm
        .call(
            addr: escrow,
            gas: 200000,
            value: Wei::zero(),
            message: VerdictEscrowMsg::Deliver { job_id: job, delivery_hash: 111 },
        )
    let _: u256 = evm
        .call(
            addr: escrow,
            gas: 300000,
            value: Wei::zero(),
            message: VerdictEscrowMsg::Verdict { job_id: job, kind: 1, delivery_hash: 111 },
        )
    let _: Wei = evm
        .call(
            addr: escrow,
            gas: 300000,
            value: Wei::zero(),
            message: VerdictEscrowMsg::Claim { job_id: job, to: proxy },
        )
}
```

What it means for this contract. Claims pay plain addresses. A payee contract in the refusing class cannot be paid this way: the claim reverts whole, the paid flag is never set, nothing is lost, and the payee can name another address and claim again. Contract wallets written in the Solidity style, with a payable receive that answers empty calldata, are a different class and remain payable. That is why the README limitation is worded for receivers that refuse empty calldata transfers, Fe contracts included, rather than for contract payees in general.

## 2. A typed call made with gas 0 never executes

What we saw. A typed contract to contract call (`call.call` with a message) made with `gas: 0` and no value does not run the callee at all: with no value there is no transfer stipend, so the callee is granted no gas. The caller then panics on the missing return data; in the build session the failure surfaced as the standard library panic payload `Panic(1)`. The starter vault's inner calls use `gas: 0` as well, but only for calls whose returns are defaulted, which masks the same behavior.

Smallest reproduction. One caller contract with two handlers that differ only in the gas argument, calling the sink from the first entry, and one test per handler:

```fe
msg CallerMsg {
    #[selector = sol("poke(address)")]
    Poke { sink: Address } -> u256,
    #[selector = sol("pokeNoGas(address)")]
    PokeNoGas { sink: Address } -> u256,
}

pub contract Caller uses (call: mut Call) {
    recv CallerMsg {
        Poke { sink } -> u256 uses (mut call) {
            call.call(
                addr: sink,
                gas: 500000,
                value: Wei::zero(),
                message: SinkMsg::Ping {},
            )
        }

        PokeNoGas { sink } -> u256 uses (mut call) {
            call.call(
                addr: sink,
                gas: 0,
                value: Wei::zero(),
                message: SinkMsg::Ping {},
            )
        }
    }
}
```

```fe
use ingot::{Caller, CallerMsg, Sink}

#[test(balance = 10_000_000_000_000_000_000)]
fn typed_call_with_gas_runs() uses (evm: mut Evm) {
    let sink = evm.create2<Sink>(value: Wei::zero(), args: (), salt: 0)
    let caller = evm.create2<Caller>(value: Wei::zero(), args: (), salt: 0)

    let pong: u256 = evm
        .call(
            addr: caller,
            gas: 900000,
            value: Wei::zero(),
            message: CallerMsg::Poke { sink },
        )
    assert!(pong == 1)
}

#[test(should_revert, balance = 10_000_000_000_000_000_000)]
fn typed_call_with_zero_gas_never_runs() uses (evm: mut Evm) {
    let sink = evm.create2<Sink>(value: Wei::zero(), args: (), salt: 0)
    let caller = evm.create2<Caller>(value: Wei::zero(), args: (), salt: 0)

    let _: u256 = evm
        .call(
            addr: caller,
            gas: 900000,
            value: Wei::zero(),
            message: CallerMsg::PokeNoGas { sink },
        )
}
```

The first test passes and returns the sink's answer. The second carries the `should_revert` marker and passes.

What it means for this repo. The `PosterProxy` helper in `src/lib.fe` makes its inner calls with explicit gas (500,000). Inner gas is also capped by the gas the outer call arrives with: the proxy flow in the harness needs the outer create call to arrive with roughly 900,000 gas before the inner 500,000 is actually available, which is why the proxy tests call with 900,000.

## 3. send_value passes its gas argument straight to the CALL

What we saw. The standard library source embedded in the toolchain binary shows `send_value` is a raw call with empty calldata, and its gas argument passes straight through as the gas of that CALL, with no special case at 0. A CALL whose gas argument is 0 still grants the callee the 2,300 gas value transfer stipend when value moves, so a gas argument of 0 does not mean no execution; it means the bare stipend only.

How we checked. We built the scratch sender from the first entry twice, once with `gas: 100000` and once with `gas: 2300`, and read the constants out of the compiled runtime. The first build carries the constant `0x186a0` (100,000) and no 2,300 constant; the second carries `0x08fc` (2,300) and no 100,000 constant. The argument in the source is the argument in the bytecode, unchanged.

What it means for this contract. The first version of `Claim` sent with a gas argument of 0 and so paid with the bare stipend only, which a receiver doing any work at all would exceed. `Claim` now forwards 100,000 gas explicitly. That is safe here because the paid flag is set before the send and `Claim` is the only handler that moves value.

## Smaller notes

Storage values and Copy. `StorageMap.get` requires the value type to implement `Copy`. A derive attribute on the `Job` struct parsed but did not satisfy the bound; the explicit block `impl Copy for Job {}` did.

Ingot names. An ingot name carrying a hyphen character (`verdict-escrow`) was rejected; `verdict_escrow` was accepted.

Mutable bindings. A mutable local is spelled `var`, not `let mut`, and the compiler error for the wrong spelling says so in plain language.

The test harness. The harness sends every call from one account, so distinct parties cannot each act for themselves in a happy path; this suite proves role separation from the other side, with jobs that name a different worker or evaluator and with the `PosterProxy` split. The harness API available to us offers no clock control, so the verdict timeout is tested at its two boundaries, a zero length window and a long one.
