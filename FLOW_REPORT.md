# Confidential auction flows and authority

This report follows one sealed-bid auction from the issuer's proposal through
funding, batch acceptance, close, clearing and recovery. The accompanying Daml
contracts implement that workflow using actual Token Standard V2 allocations
and two instances of the pinned OpenZeppelin experimental token registry.

The worked example sells 100 tokens in lots of 10. Alice requests 50 at a maximum
price of 12, Bob requests 80 at 10, and Carol requests 40 at 10. The result is
50, 40 and 10 tokens respectively, all at price 10.

The operator's restricted grant enforces the complete clear. Governance must
preserve those approved entry points and restrict direct submissions as `av`.
If someone can submit directly as `av`, the contracts permit other valid Daml
transactions, including settling only the payment side. The authority
demonstration executes that counterexample explicitly.

## Parties and authority

| Party | Responsibility |
|---|---|
| Issuer | Approves the round terms and use of its inventory and proceeds accounts |
| Bidder | Approves its quantity, maximum price and payment and delivery accounts |
| Account provider | Supplies any additional consent required for its accounts |
| `ao` | Submits the operator's fixed preparation, admission, close, clear and recovery workflows |
| `av` | Signs auction state and delegation grants and is the allocation executor |
| Cash and token admins | Operate the two asset registries |
| Eligibility provider | Signs and updates bidder credentials |
| Settlement attester | Approves settlement when required by the chosen policy |
| Auditor | Reads the disclosed history and checks membership, results and recovery |

In the notation below, `S` is a contract's signatory set and `C` is a choice's
controller set. Exercising a choice requires its controllers' authority. Inside
that choice, the available authority is its controllers plus its contract's
signatories. A further nested choice gets its own controllers and signatories.
It does not inherit every other party from earlier ancestors.

The token interfaces call their selected authorizers `actors`. Naming a party
in that list does not manufacture consent. The exercise must already have that
party's authority, and the registry also checks that the actors satisfy its rule.

The scripts submit operator commands with `actAs ao` and `readAs av`. This models
the operator's authorized access to its co-hosted validation party's projection.
`readAs` supplies visibility, not the right to exercise an `av`-controlled
choice. The authority tests verify that distinction. Factory and input-holding
disclosures model information supplied by registries and account services.

The asset helper uses `AllocationFactory_PublicFetch` to read the registry under
the account parties' authority, then calls `AllocationFactory_Allocate`. See
[Auction.Assets](auction/daml/Auction/Assets.daml) and the
[Daml authorization model](https://docs.canton.network/appdev/modules/m3-authorization).

## 1. Prepare and open the round

The issuer supplies the intended terms. The operator creates a preparation
contract, and the issuer and its account provider approve those exact terms in
separate transactions. Each approval consumes its predecessor and adds the
approver as a signatory of the successor.

```text
T1  ao -> Preparation Grant.StartRound(terms)
      Grant S = av, C = ao
      create RoundPreparation S = av

T2  issuer -> RoundPreparation.ApproveRound(issuer)
      consume preparation
      create successor S = av + issuer

T3  issuer provider -> ApproveRound(provider)
      consume preparation
      create successor S = av + issuer + provider

T4  ao -> RoundPreparation.FundRound(inventory holdings)
      C = ao, S = av + all required issuer account parties
      check complete consent and bidding deadline
      registry.PublicFetch(actors = inventory account parties)
      registry.Allocate(actors = inventory account parties)
      create committed supply allocation, executor = [av]
      create SaleAuthority S = av + issuer account parties
      create Proposal S = issuer + av
      consume preparation

T5  ao -> Opening Grant.OpenRound(proposal, current supply allocation)
      -> Proposal.Proposal_Open, C = av, S = issuer + av
         validate funded, unmarked supply and deadline
         -> SaleAuthority.BindSale, C = av
         create Round(Open, accepted = [], nextNumber = 0)
         create PublishedTerms
         consume proposal
```

The initial supply allocation authorizes the sender side of a leg from the
inventory account back to itself. It reserves inventory without choosing winners
or supplying the receiver approval needed for settlement. Its settlement reference
contains the preparation identity, and its executor list is exactly `[av]`.
It is committed through the settlement deadline.

Funding and creation of the proposal and sale authority are one transaction.
Opening is a later transaction. A failed funding transaction leaves the signed
preparation available. Consuming the proposal prevents a second opening through
that proposal. Its contract ID becomes the stable round identity, copied into
every round successor and the published terms.

The prototype checks the bidding and settlement cutoffs. It does not enforce
separate minimum preparation or clearing margins when opening a round.

Read [Opening](scripts/daml/Flows/Opening.daml) for the individual submissions.

## 2. Prepare and fund each bid

Preparation uses a separate contract per bid. It does not update the round.
Alice can prepare while Bob prepares, provided they do not compete for the same
holding or another shared account resource. The scripts demonstrate independent
states through sequential submissions, rather than benchmarking parallel clients.

```text
ao -> Preparation Grant.StartBid(published terms, Alice's bid)
  create BidPreparation S = av

Alice -> ApproveBid(Alice)
  successor S = av + Alice

Alice's account provider -> ApproveBid(provider)
  successor S = av + Alice + provider

ao -> BidPreparation.FundBid(payment holdings)
  require all payment and delivery account approvals
  maximum payment = roundUp(quantity * maximum price)
  registry.Allocate(actors = payment account parties)
  create committed payment allocation, executor = [av]
  create PreparedBid S = av + Alice + account parties
  consume BidPreparation
```

Alice's example reserves 600. The registry consumes her input holdings and
creates locked holdings for that amount. The prepared bid binds the allocation
root, complete fixed terms, both accounts and round identity. It has no acceptance
number and no right to a fill.

The account approvals supply authority later. A recorded account-party identifier
alone would not do that. The [Funding](scripts/daml/Flows/Funding.daml) script
separately rejects missing provider consent and a 599-unit payment input.

## 3. Accept bids in batches

The round's accepted list is the authoritative auction membership. A prepared
bid becomes a member only when a committed batch consumes it and appends the
resulting accepted-bid contract ID to that list.

```text
ao -> Admission Grant.AcceptBatch(current round, [Alice, Bob], evidence)
  -> Round.Round_AcceptBatch, C = av, S = issuer + av
     require Open and ledger time < bidding deadline
     require distinct prepared IDs and batch/round capacity
     validate the current supply lock
     for each prepared bid in the supplied order:
       validate round, accounts, terms, exact funded lock and eligibility
       -> PreparedBid.Prepared_Accept
          C = issuer + av
          S = av + bidder + account parties
          create AcceptedBid with the next acceptance number
          consume PreparedBid
     create one round successor with appended IDs and updated nextNumber
     consume the previous round
```

The operator selects batch order. Alice and Bob get numbers 0 and 1. A later
batch containing Carol gets number 2. A rejected batch assigns no numbers and
accepts no bids, even if its first bid passed before a later bid failed.

There is still one serialization point: the current round contract. Batching
reduces round replacements from one per bid to one per batch. It does not remove
contention, guarantee throughput, or prove fairness of off-ledger arrival order.
An operator should bound batch size and waiting time and reconcile the current
round ID before retrying.

Each bidder can see its own acceptance number. Multiple bids can therefore
reveal intervening acceptance counts. The private full list remains visible to
the issuer, operator and validation party. The closed list fixes membership,
not merely a count or an operator-supplied claim that the book is complete.

Read [Bidding](scripts/daml/Flows/Bidding.daml) for the rollback and stale-round
examples.

## 4. Close and compute the result

At or after the bidding deadline, the close grant exercises the current round
and creates its `Closed` successor. It preserves every accepted ID and order
number. Admission then fails because the round is closed or the deadline passed.

The operator computes a proposed result off ledger from the closed book. The
clear recomputes the result on ledger and rejects a mismatch. Both use
`computeResult`. The off-ledger proposal does not remove the on-ledger calculation.

Higher price bands fill first. An oversubscribed marginal band receives
proportional whole-lot floors, with remaining lots assigned by acceptance order.
The uniform price is the reserve when eligible demand is at most supply, and
otherwise the lowest winning maximum price. The same upward quantum rounding
applies to the initial maximum payment and the final payment.

```text
Supply: 10 lots of 10 tokens, reserve price 8
Alice: 5 lots at 12       -> 5 lots
Bob:   8 lots at 10       -> 4 lots
Carol: 4 lots at 10       -> 1 lot

Remaining marginal supply: 5 lots
Proportional floors: Bob 3, Carol 1
Remaining lot: Bob, because he was accepted earlier
Uniform price: 10
Payments: 500 + 400 + 100 = 1,000
```

A current revoked or expired credential, or a live marked payment allocation,
gives that bid zero fill. Missing current evidence blocks validation. A marked
supply allocation blocks the clear. A consumed allocation without live funded
continuation cannot be silently treated as an excluded bid.

## 5. Clear and settle in one transaction

The clear receives the exact accepted IDs in their recorded order, current lock
and credential references, the proposed result and any required approval. The
operator cannot remove, add, duplicate, reorder or substitute a bid through this
choice.

```text
ao -> Clearing Grant.ClearRound(closed round, evidence, proposed result)
  -> Round.Round_Clear, C = av, S = issuer + av
     require ledger time < settlement deadline
     validate the complete book, current funding and current eligibility
     recompute and check the proposed result

     for every accepted bid:
       -> AcceptedBid.FinalizeBid(own fill, price, current payment allocation)
          C = av, S = issuer + av + bidder + account parties
          validate its local quantity and maximum-payment bounds
          if it wins:
            cancel original payment allocation, actors = [av]
            receive unlocked holdings back into the same account
            allocate exact payment sender side using those holdings
            allocate exact token receiver side with no funding input
          else:
            retain original payment lock and create a recovery ticket
          create the bidder's private outcome
          consume AcceptedBid

     -> SaleAuthority.FinalizeSale(exact payment and delivery legs)
        C = av, S = av + issuer account parties
        cancel original supply allocation
        allocate exact token sender sides using returned holdings
        allocate exact payment receiver sides
        consume SaleAuthority

     verify full-reference auction approval when required
     cash registry.SettleBatch(all exact cash legs and their allocations)
     token registry.SettleBatch(all exact token legs and their allocations)
     create ClearedRound and consume the closed Round
```

**Every indented operation is part of the same transaction.** The cancellation,
replacement allocations and both registry settlements are not separate operator
submissions. The clear either commits all of them or none of them.

Cancellation returns funds to the original authorizing account. It does not
transfer them to `av`. Creating the replacement payment and delivery allocations
requires the account authority carried by the bid or sale contract. The parent
round's issuer signature would not by itself supply that authority inside an
unrelated nested choice.

Each bid child receives its own fill. The issuer child receives the exact sale
legs, without the winners' original maximum prices. The two factory calls sit
outside individual bid branches. Registry-specific disclosure and shared account
providers still affect visibility and need deployment projection tests.

The [pinned D1 experiment](https://github.com/OpenZeppelin/canton-contracts/blob/7696749737885e25cd88422847105f890f03b00d/experiments/token/tokenCIP112-v1/daml/OpenZeppelin/TokenCIP112V1/D1.daml)
binds the textual settlement ID, executors and exact legs. The prototype adds
`BatchApproval` to bind the complete `SettlementInfo`, including its round
contract ID and metadata. It still invokes the registry's real D1 check.

`atomicRollbackAndRetry` deliberately omits the token registry's D1 attestation.
The token factory rejects after the cash factory has run within the transaction.
The script checks that the original closed round, accepted bid, supply and payment
locks, and application approval remain active, with no committed proceeds or
outcome. Supplying the attestation allows the same closed round to clear.

An empty round returns the supply and records a zero-sale result without calling
either factory with an empty settlement batch.

## 6. Failure and recovery

| Situation | Result and next step |
|---|---|
| A preparation lacks account consent | Funding fails. Obtain the outstanding approval or abandon preparation |
| Input holdings are insufficient | Funding rolls back. Existing holdings and preparation remain usable |
| One bid invalidates a batch | No bid in that batch is accepted. Correct or split the batch before cutoff |
| A prepared bid misses cutoff | It remains unaccepted. Its bidder can abandon it and obtain a recovery ticket |
| A clear definitively fails before expiry | Original round and commitments remain. Retry the complete accepted set with valid current evidence |
| A client times out without a final result | Reconcile command completion and ledger state before treating the transaction as rejected |
| Settlement deadline is reached | Clear fails. The termination grant consumes the round and records an end without sale |
| Operator cancels an uncleared round | The entire round ends with a reason. This grants no selective early removal from an active round |
| A bid has zero fill | Its outcome permits separate cancellation of its original lock. An unmarked lock can be released immediately |
| A lock is marked | Registry cancellation and withdrawal fail until an authorized release of the mark or another registry resolution |
| Operator recovery is unavailable after expiry | The account parties can invoke the registry's withdrawal path when its deadline and policies allow |

Ending the round and recovering assets are separate operations:

```text
ao -> Termination Grant.EndRound
  consume the open or closed round
  create EndedRound retaining the accepted list and sale reference

For each available accepted bid:
  ao -> Recovery Grant.CloseUnsettledBid
    EndedRound checks membership
    AcceptedBid creates a private outcome and recovery ticket, then is consumed
  ao -> Recovery Grant.Release(ticket, current allocation)
    validate allocation lineage and cancel under registry rules

Supply:
  ao -> Recovery Grant.CloseUnsettledSupply
    SaleAuthority creates a recovery ticket and is consumed
  ao -> Recovery Grant.Release(ticket, current supply allocation)
```

This separates recovery across accounts. An unavailable bidder does not need to
block another bidder's recovery transaction. A bidder whose account parties
withdraw directly may leave an application recovery record requiring later
reconciliation. That bookkeeping cleanup is outside this prototype.

The direct-withdrawal script submits jointly as the account owner and provider,
which are hosted together in the fixture. Separate hosts would need an account
consent workflow that brings both authorities into the withdrawal choice.

The scripts simulate ledger deadlines with the SDK's time controls. A closed
wallet is different from an unavailable participant: the user's earlier consent
remains valid, but the required participant confirmations must still be available
for a transaction involving its contracts.

After a definitive rejection for missing confirmations, the round and accepted
bids remain active. The operator can retry before the settlement deadline once
the required participants are available. The scripts in this package exercise
contract behavior in the test engine. Participant outages require a distributed
network test.

## 7. Audit and governance boundaries

An auditor should retain the accepted records before they are consumed and check:

1. Every accepted bid occurs exactly once in the closed list and has the expected
   round identity and acceptance number.
2. Current allocation successors preserve their original account, instrument,
   amount, executor and settlement bindings and remain funded.
3. Each exclusion has current authenticated evidence.
4. Recomputed fills, price, rounded payments and totals match the aggregate result.
5. Every accepted bid has one private outcome, including zero fills, and asset
   movements and recovery reconcile with those outcomes.

The audit script retains a closed-book snapshot and checks the worked example's
outcomes and proceeds against independently specified figures. It is a
demonstration of those checks, not a persistent historical indexer. Signed request
receipts, failed-command evidence, admission-policy compliance and historical
exports require additional operator and auditor infrastructure.

The intended hosting topology places `av` on the operator's participant and two
independent validation participants, with two of three required to confirm. An
auditor can host `av` with Observation permission, receiving its private auction
data without submission authority or a confirmation vote. These hosting settings
are outside the included scripts.

Confirmation, control over the party's namespace, and authority to submit as `av`
are separate controls. A two-of-three confirmation threshold does not by itself
restrict direct `av` submissions. Grant administration and topology changes need
their own governed authorization process.

`validationPartyTrustBoundary` uses a submission-capable `av`.
It calls the bid and sale finalization choices in separate transactions,
then settles only cash. The issuer receives 400 and Alice has no delivered tokens.
This exposes why governance must restrict direct submissions as `av`. The
operator cannot do the same thing with its grants.
The contracts therefore rely on governance preserving the restricted operator
entry points. They do not guarantee atomic exchange against unrestricted `av`
authority.

## Scope and sources

The executable model uses two distinct asset admins, fixed account-owner consent,
explicit provider approval, bounded whole-lot bids and at most 64 accepted bids.
It uses synchronous, non-iterated allocations and the specific registry's D2
successor rules. It covers D1 approval, credential revocation, marking/unmarking,
normal recovery and committed-allocation withdrawal. Seizure sweeps, registry
upgrades, fees, external-party signatures, historical replay and production
load testing require further work.

The registry is copied without edits from
[OpenZeppelin/canton-contracts at `7696749`](https://github.com/OpenZeppelin/canton-contracts/tree/7696749737885e25cd88422847105f890f03b00d/experiments/token/tokenCIP112-v1).
Its canonical Token Standard DARs are included under `vendor/dars/vendor/`.
The interface specification is
[CIP-0112 at `6f37c89`](https://github.com/canton-foundation/cips/blob/6f37c896a5a76ec3bc1aa67bc045623ae5df41e5/cip-0112/cip-0112.md).
Dependency versions, checksums and licenses are recorded in
[vendor/README.md](vendor/README.md). [The README](README.md#run) explains how to
run the scripts.
