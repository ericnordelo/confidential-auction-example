# Confidential auction example

A Daml proof of concept for a confidential, uniform-price token auction. The
contracts demonstrate allocation requests initiated by sellers and bidders,
registry custodian approval, prefunding, batch admission, atomic settlement and
recovery. The Daml Scripts explain each flow and assert its results.

Start with [the flow report](FLOW_REPORT.md) for transaction steps and authority.
For a complete auction from opening to settlement, read `happyPath` in [Clearing](scripts/daml/Flows/Clearing.daml).

## Explore the flows

| Script | Demonstrates |
|---|---|
| [Opening](scripts/daml/Flows/Opening.daml) | Immediate and custody round funding, mixed readiness, opening and preparation pause |
| [Funding](scripts/daml/Flows/Funding.daml) | Seller authority, insufficient funds and atomic funding or bid-creation rollback |
| [Bidding](scripts/daml/Flows/Bidding.daml) | Immediate bidder creation, authorization and batch acceptance |
| [Clearing](scripts/daml/Flows/Clearing.daml) | Payments, delivery, zero-fill refunds and rounds with no winners |
| [Recovery](scripts/daml/Flows/Recovery.daml) | Abandoned requests, registry withdrawal, missed deadlines and cancelled rounds |
| [Controls](scripts/daml/Flows/Controls.daml) | Eligibility, cancelled allocations, settlement approval, rollback and pause |
| [Iterated](scripts/daml/Flows/Iterated.daml) | Splice funding and receipt allocations, custody approval, mixed readiness, refunds, rollover and admission guards |
| [Authority](scripts/daml/Flows/Authority.daml) | Operator restrictions and the trust placed in `av` |
| [Audit](scripts/daml/Flows/Audit.daml) | Reconciliation of bids, outcomes and proceeds |
| [Privacy](scripts/daml/Flows/Privacy.daml) | Provider visibility during approval, settlement and recovery |
| [Math](scripts/daml/Flows/Math.daml) | Clearing rules and payment rounding |

`auction/` contains the contracts. The scripts share setup helpers in
[Support](scripts/daml/Flows/Support.daml) and test-fund minting in
[Fixtures](scripts/daml/Flows/Fixtures.daml). The unchanged Splice V2 test-token registry and pinned Token
Standard dependencies are described in [vendor/README.md](vendor/README.md).

## Run

Use [DPM](https://docs.digitalasset.com/build/3.4/dpm/index.html) and Java 21.
The workspace pins SDK 3.4.11 and Daml-LF 2.1. From this directory:

```sh
dpm install
dpm build --all
cd scripts
dpm test
```

The multi-package build orders the Splice token, auction and scripts by their
dependencies. DPM supplies `daml-script` from the selected SDK. To run only the
complete auction example, use `dpm test -p happyPath` from `scripts/`.

## Design boundaries

The approved operator workflow settles both assets atomically. Unrestricted
submission as `av` can bypass that workflow, as the authority script demonstrates.
Production governance must restrict that authority. Batching still serializes
admission, and settlement needs the required participants to remain available.

These scripts demonstrate contract behavior. Wallet integration, distributed
hosting, production governance and a continuous audit service need separate
implementation and validation. The [flow report](FLOW_REPORT.md#7-audit-and-governance-boundaries) explains the
audit and governance assumptions.
