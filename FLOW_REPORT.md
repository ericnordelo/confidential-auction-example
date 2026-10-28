# Confidential auction flows and authority

This report follows the executable auction from preparation through batch
acceptance, clearing and recovery. It uses two instances of Splice's
[TestTokenV2](https://github.com/canton-network/splice/tree/69b43eb761e38695052c983715aa855c8cb207fc/token-standard/examples/splice-test-token-v2),
one for payment and one for inventory. The upstream Daml source is unchanged.

The seller and bidders authorize committed, iterated allocations before opening
or admission. Clearing computes the exact transfers, adds their sides to those
existing allocations and settles both assets in one transaction. Finalization
uses `nextIterationFunding = None`, so the registries return unused funding to
the original accounts during settlement. Bids with zero fill settle both
allocations with no transfer sides and recover their full payment in that same
transaction.

The worked auction sells 100 tokens in lots of 10. Alice requests 50 at a maximum
price of 12, Bob requests 80 at 10, and Carol requests 40 at 10. They receive
50, 40 and 10 tokens respectively, all at price 10. The separate custody example
has one bidder, so Alice receives 50 at reserve 8 and recovers 200 of her 600
reserved payment.

## Parties and authority

| Party | Responsibility |
|---|---|
| Seller | Approves round terms, inventory delivery and payment receipt. |
| Bidder | Submits signed bid terms and requests payment and receipt allocations. |
| Account provider | Approves the account policy and the allocations required by that policy. |
| `ao` | Coordinates round preparation, batch admission, close, clear and recovery. |
| `av` | Signs auction state and grants and is the sole allocation executor. |
| Cash and token admins | Operate the two upstream asset registries. |
| Eligibility provider | Signs and updates bidder credentials. |
| Settlement attester | Signs the application's optional approval of the exact settlement. |
| Auditor | Checks disclosed membership, results, asset movements and recovery. |

In the pseudocode, each `TX` is a separate submission. Indented calls happen
inside that submission. `S` names a contract's signatories and `C` names a
choice's controllers. A caller must supply the controllers' authority. A choice
body uses its own controllers and contract signatories as authorizers for its
nested actions. This rule is applied again at each nested exercise. Creating a
contract requires the authority of all its signatories.

Token Standard choices name their controllers through `actors`. Naming a party
there requires its authority and must satisfy the registry's authorization
rules. Operator submissions use `actAs ao` and `readAs av`. The latter provides
read access to `av`'s projection. Its authority comes from exercising the
`av`-signed grant, rather than from `readAs`.

[Workflow.daml](auction/daml/Auction/Workflow.daml) defines the application
choices. [Assets.daml](auction/daml/Auction/Assets.daml) calls the token interfaces.
The [Daml authorization model](https://docs.canton.network/appdev/modules/m3-authorization)
explains the underlying rule.

## Allocation approvals and registry context

Before creating allocations, the setup scripts create each registry's
`TokenRules`, signed by its admin. For accounts with a provider, they also agree
on an upstream `AccountConfig`:

```text
TX  provider -> create AccountProposal(config), S = provider
TX  owner -> AccountProposal_Accept, C = owner, S = provider
      create AccountConfig, S = owner + provider, observer = admin

TX  av -> create separate Grant contracts for each operation category
      S = av, observer = ao
```

The example configures both owner and provider to approve account actions.
Basic accounts have no provider and use the owner's authority. Account policy
is distinct from consent to an individual bid or allocation. Test funding uses
Splice's mint-offer and transfer-acceptance flow through the script-only
`MintConsent` helper.

All auction allocations use the same full settlement reference:

```text
SettlementInfo
  executors = [av]
  id        = "auction-clear"
  cid       = Some roundId
  meta      = emptyMetadata
```

`roundId` is the `RoundPreparation` contract ID, fixed before funding. It
persists in the funding request or proposal, published terms and every round
state. The current `Round` contract ID changes whenever a batch is accepted or
bidding closes.

Each initial allocation has `transferLegSides = []`, `committed = True` and the
round's settlement deadline. Its funding depends on the account's role:

| Authorization | Initial `nextIterationFunding` | Reserved funds |
|---|---|---:|
| Seller inventory send | `Some {ISSUE: 100}` | 100 tokens |
| Seller payment receipt | `Some {}` | None |
| Alice payment send | `Some {USD: 600}` | 600 payment units |
| Alice token receipt | `Some {}` | None |

`Some {}` enables iterated settlement without reserving funds. It allows the
executor to supply a receiving side once the fill is known. Setting this field
to `None` at creation would permit only the original fixed sides. Setting it
to `None` in a later `FinalizedAllocation` ends the allocation after that
settlement and releases the remainder. These are different uses of the field.
See [CIP-0112 section 4.3.6](https://github.com/canton-foundation/cips/blob/6f37c896a5a76ec3bc1aa67bc045623ae5df41e5/cip-0112/cip-0112.md#436-committed-allocations-and-iterated-settlement).

Splice's `ExtraArgs` supplies the event-log registry ID and any required
account-configuration contract IDs. Allocation creation also needs input-holding
disclosures. Settlement and cancellation need fresh disclosures of the locked
holdings. The scripts simulate the registry's context service through
`withRegistryDisclosures`, querying the admin's records. The operator still
submits as `ao`, with no admin submission authority.

`Assets.requestAllocation` checks the registry admin through
`AllocationFactory_PublicFetch`, then calls `AllocationFactory_Allocate`. Bid
creation supplies only the bidder as an actor, and round funding supplies only
the seller. Registry instructions collect any additional account approvals.

The auction validates the pinned `TokenAllocationV2` template, allocation root,
exact specification, settlement reference and recorded funding keys. It requires
`numIterations = 0` before clear. Funding amounts are established by the upstream
registry implementation. The round does not fetch each private holding to
recalculate those amounts. The scripts inspect resulting balances separately.

## 1. Prepare and open the round

The seller approves the round terms and requests inventory-send and
proceeds-receipt allocations in one transaction. Basic accounts complete
immediately. Custodied accounts use the registry's allocation instructions to
collect provider approval.

### Request funding

`ao` creates the preparation through its grant. The seller then exercises
`FundRound`, supplying the live preparation grant, inventory holdings and
registry context.

```text
TX  ao -> Preparation Grant.StartRound(terms)
      Grant S = av, C = ao
      create RoundPreparation, S = av

TX  seller -> RoundPreparation.FundRound(preparation grant, inventory inputs, context)
      S = av, C = seller
      check the live preparation grant and bidding deadline
      roundId = this preparation's contract ID
      request inventory-send allocation, actors = [seller]
        nextIterationFunding = Some {ISSUE: 100}
      request proceeds-receipt allocation, actors = [seller]
        nextIterationFunding = Some {}

      if both results are Completed:
        create SaleAuthority and Proposal, S = seller + av
        bind both allocation roots
        return FundedRound(proposal)
      otherwise:
        create RoundFundingRequest, S = seller + av
        retain fixed terms, roundId and both initial registry results
        return RegistryPreparedRound(request)

      consume RoundPreparation
```

Each factory has its asset admin as signatory. It combines that authority with
the seller's `actors` to create either a completed allocation or a pending
instruction. The completed allocation carries the registry's required account
consent. Providers approve through registry contracts. Round preparation,
funding requests and sale authority are visible to the seller, `av` and `ao`.

A failure in either registry rolls back the whole funding submission and
preserves the preparation and input holdings. A pending result commits the
instruction and auction request. In this registry, pending instructions leave
inventory available until approval, so the seller must keep the inputs usable.

### Complete custody approval

```text
TX  seller provider -> inventory instruction.AllocationInstruction_Accept
      actors = [provider], existing instruction S = token admin + seller
      reserve 100 tokens and consume the instruction
      create inventory allocation, S = token admin + seller + provider
        observer = av

TX  seller provider -> proceeds instruction.AllocationInstruction_Accept
      actors = [provider], existing instruction S = cash admin + seller
      consume the instruction
      create unfunded proceeds allocation, S = cash admin + seller + provider
        observer = av
```

These are registry transactions. They leave `RoundFundingRequest` unchanged.
One side may have completed in the seller's initial submission while the other
needs approval. The operator waits for both completed allocation IDs.

### Open the round

```text
TX  ao -> Opening Grant.OpenRound(source, current inventory, current proceeds receipt)
      Grant S = av, C = ao
      check the opening grant against the source's fixed terms

      if source is RegistryPreparedRound:
        -> RoundFundingRequest.CompleteFunding
           S = seller + av, C = av
           validate both completed allocations against the fixed terms and roundId
           preserve the root of any initially completed side
           consume the request and create SaleAuthority + Proposal

      -> Proposal.Proposal_Open, S = seller + av, C = av
         validate both allocation bindings and the bidding deadline
         -> SaleAuthority.BindSale(roundId), S = seller + av, C = av
         create Round(Open, accepted = [], nextNumber = 0)
           S = seller + av, observer = ao
         create PublishedTerms, S = seller + av, observer = ao
         consume Proposal
```

The opening grant supplies `av` authority to the nested choices. The seller's
signed proposal or funding request supplies its terms approval. Provider
authority is already carried by the allocations and is needed only inside
registry operations.

For a custody request, binding and opening commit in one transaction. An
incomplete or invalid allocation preserves the request, and consuming it on
success prevents another opening. Immediate funding uses the existing proposal.
The same `roundId` persists through both paths. Opening must precede the bidding
cutoff. Minimum preparation and clearing margins remain outside this prototype.

The seller can abandon an unfunded preparation without asset cleanup. Abandoning
an immediate funded proposal creates a recovery ticket for its bound allocation
pair. Abandoning a `RoundFundingRequest` consumes the request, while registry
withdrawal handles pending instructions and completed committed allocations.
The latter remain committed until registry withdrawal is permitted.

Read [Opening](scripts/daml/Flows/Opening.daml) for immediate, custody and mixed
funding, binding checks, preparation pause and proposal abandonment.
[Recovery](scripts/daml/Flows/Recovery.daml) shows withdrawal after abandoning
pending or partly funded round requests.

## 2. Submit each bid

The bidder creates its bid and requests both allocations in one transaction.
The registry decides whether the account policy permits immediate completion or
requires an allocation instruction. Account-provider approvals use registry
choices. They are separate from the bidder's consent to its auction terms.

`SubmitBid` is a non-consuming choice on `PublishedTerms`. It leaves the round
and published terms active, so bids with distinct funding inputs and account
resources can prepare independently. The scripts submit sequentially and do not
benchmark concurrent clients.

```text
TX  bidder -> PublishedTerms.SubmitBid(bid, payment inputs, registry context)
      S = seller + av, C = bidder
      fetch and check the live Preparation Grant
      check bid terms and time < bidding deadline
      maximum payment = roundUp(quantity * maximum price)

      -> cash factory.AllocationFactory_Allocate
         factory S = cash admin, actors = [bidder]
         nextIterationFunding = Some {USD: maximum payment}
         result = Completed(payment allocation) or Pending(payment instruction)

      -> token factory.AllocationFactory_Allocate
         factory S = token admin, actors = [bidder]
         nextIterationFunding = Some {}
         result = Completed(receipt allocation) or Pending(receipt instruction)

      if both results are Completed:
        create PreparedBid, S = av + bidder
        retain both allocation roots and the exact bid terms
        return FundedBid(PreparedBid)
      otherwise:
        create BidRequest, S = av + bidder
        retain the exact bid terms and both initial registry results
        return RegistryPreparedBid(BidRequest)
```

The bidder supplies the choice's controller authority. The published terms supply
`av` authority to create the signed application record. Each registry factory
uses its own admin authority and the bidder's authorization to process the
allocation request. The bidder, seller, `av` and `ao` see the bid record.
Providers see the registry instructions and allocations for their accounts.

A failed factory call rolls back the whole submission, including any allocation
created by the first registry. A `Pending` result commits the instruction and
bid request. Payment and receipt can have different readiness states.

### 2.1 Immediate allocations

Bob uses accounts without a provider. His one `SubmitBid` transaction reserves
600 for a bid of 50 tokens at maximum price 12, creates the unfunded receipt
allocation and creates `PreparedBid`. The operator can then include it in a
batch. Both allocations and the signed bid exist before admission.

`PreparedBid` has no acceptance number. The bidder can abandon it and obtain a
recovery ticket for the two allocations created by its submission.
[Bidding](scripts/daml/Flows/Bidding.daml) demonstrates immediate creation,
authorization checks and batch admission. [Funding](scripts/daml/Flows/Funding.daml)
checks insufficient inputs and rollback when the second registry fails.

### 2.2 Custodian approval

Alice's account policies require her provider's approval. Her single `SubmitBid`
transaction creates two upstream `AllocationInstruction` contracts and a
`BidRequest`, signed by Alice and `av`. The request stays active while each
registry collects its account approvals.

```text
TX  Alice -> PublishedTerms.SubmitBid
      payment = Pending(cash instruction)
      receipt = Pending(token instruction)
      create BidRequest, S = av + Alice
      each instruction S = its registry admin + Alice

TX  Alice provider -> cash instruction.AllocationInstruction_Accept
      actors = [provider], S = cash admin + Alice
      reserve 600, consume the instruction
      result = Completed(payment allocation)

TX  Alice provider -> token instruction.AllocationInstruction_Accept
      actors = [provider], S = token admin + Alice
      create unfunded receipt allocation, consume the instruction
      result = Completed(receipt allocation)

TX  ao -> Admission Grant.AcceptBatch
      include RegistryPreparedBid(BidRequest)
      supply both completed allocation IDs and current credential
      bind and accept inside the batch transaction described in section 3
```

A pending payment instruction in this registry leaves its inputs unlocked. Those
holdings must still be available when the provider completes it. Payment funding
alone is insufficient for admission while receipt approval is pending.

The initial results let the wallet follow the registry workflows. Acceptance
checks the actual completed allocations against the bid's accounts, funding and
full round reference. It binds them and consumes `BidRequest` in the same
transaction. Before acceptance, abandoning this request creates no application
cancellation ticket. Registry withdrawal handles its instructions and allocations,
so another unaccepted request cannot gain recovery authority over accepted funds.

Read `custodianApproval` and `mixedReadiness` in
[Iterated](scripts/daml/Flows/Iterated.daml) for two pending approvals and a
completed receipt with pending payment. Both examples clear after registry
approval without another bidder or provider submission at settlement.

## 3. Accept bids in batches

A committed batch appends accepted-bid contract IDs to the round's private list.
That list determines membership. Both bid sources can use the same batch:

- `FundedBid` names a `PreparedBid` created with two immediate allocations.
- `RegistryPreparedBid` names a signed `BidRequest` whose allocations have
  completed their registry workflows.

```text
TX  ao -> Admission Grant.AcceptBatch(round, ordered bids, seller allocations)
      Grant S = av, C = ao
      -> Round.Round_AcceptBatch, S = seller + av, C = av
         require Open, time < bidding deadline and batch/round capacity
         read each source's authenticated terms and allocation bindings
         reject duplicate sources and allocation roots reused within the
           batch or by an already accepted bid
         validate seller inventory and proceeds allocations

         for each source in the supplied order:
           validate round, terms, payment and receipt allocations
           validate current eligibility and retain its credential identity
           if FundedBid:
             -> PreparedBid.Prepared_Accept
                C = seller + av, S = bidder + av
                consume PreparedBid
           if RegistryPreparedBid:
             -> BidRequest.AcceptRegistryPrepared
                C = seller + av, S = bidder + av
                validate and bind the completed allocations
                consume BidRequest
           create AcceptedBid with the next acceptance number
             S = seller + av + bidder
             ao observes it

         create one Round successor with appended IDs and updated nextNumber
         consume the prior Round
```

The round supplies seller and `av` authority to the acceptance choice. The
bid source supplies bidder authority, allowing the accepted record to carry
all three signatures. Providers' asset consent remains in their allocations.

All bids share the round's `SettlementInfo`. Separate bids, including identical
ones, need separate allocation pairs so their funding is counted once and each
recovery ticket releases only its own allocations.

The operator chooses order within each batch. A batch starting at number `n`
assigns consecutive numbers and replaces the round once. One invalid or
unconfirmable input rejects the entire transaction, preserving the sources,
allocations and previous round. Competing batches still contend on that round.

The seller, operator and validation party see the accepted list. Each bidder
sees its own accepted record and number. Comparing several numbers can reveal
acceptance activity. The stored order does not prove fair ordering of off-ledger
requests, and the operator can refuse or delay admission before it commits.

[Bidding](scripts/daml/Flows/Bidding.daml) exercises batch rollback and stale round
IDs. `allocationReuseGuard` and `roundReferenceGuard` in
[Iterated](scripts/daml/Flows/Iterated.daml) check duplicate funding and use of
allocations from another round.

## 4. Close bidding and compute the result

```text
TX  ao -> Closing Grant.CloseRound(current round)
      Grant S = av, C = ao
      -> Round.Round_Close, S = seller + av, C = av
         require Open and time >= bidding deadline
         consume the open round
         create Closed successor preserving terms, accepted list and order
```

The operator reads the closed book and proposes a result off ledger. The clear
independently calls `computeResult` on authenticated inputs and rejects a
mismatch. Closing fixes membership, while current credential status can still
affect eligibility at clear.

Higher maximum-price bands fill first. The oversubscribed marginal band gets
proportional whole-lot floors. Leftover lots go by acceptance order within that
band. Winners pay the reserve when eligible demand is at most supply, otherwise
the lowest winning maximum price.

```text
Supply: 10 lots of 10 tokens, reserve 8
Alice: 5 lots at 12       -> 5 lots, payment 500
Bob:   8 lots at 10       -> 4 lots, payment 400
Carol: 4 lots at 10       -> 1 lot, payment 100

Marginal supply: 5 lots
Proportional floors: Bob 3, Carol 1
Leftover lot: Bob, because he was accepted earlier
Uniform price: 10
```

`Credential` is provider-signed and observed by the bidder and `av`. Its update
consumes the prior status and preserves the original credential identity.
The provider creates the credential before acceptance. Current revocation or
expiry gives an accepted bid zero fill. Missing evidence
blocks validation. A cancelled funding or receipt allocation also blocks the
clear rather than removing the bid from the accepted set.

## 5. Clear and settle atomically

The clear receives the exact accepted IDs in their recorded order, current
sending and receiving allocation IDs, current credentials, the proposed result,
registry context and optional application approval.

```text
TX  ao -> Clearing Grant.ClearRound(closed round, ClearInput)
      Grant S = av, C = ao
      -> Round.Round_Clear, S = seller + av, C = av
         require Closed and time < settlement deadline
         validate the exact accepted list and both seller allocations
         validate both allocations and current credential for every bid
         recompute price, fills and rounded payments
         require the proposed result to match
         construct the exact payment and delivery legs

         for every accepted bid:
           -> AcceptedBid.FinalizeBid(current payment, current receipt,
                                      own fill, clearing price)
              S = seller + av + bidder, C = av
              check local quantity, price and payment bounds
              winner: return FinalizedAllocation values for existing IDs
                payment allocation: extra sides = [payment sender side]
                delivery allocation: extra sides = [delivery receiver side]
              zero fill: return both existing allocations with extra sides = []
              both cases: nextIterationFunding = None for both allocations
              create private Outcome with no recovery ticket, S = av
                observers = seller + ao + bidder
              consume AcceptedBid

         -> SaleAuthority.FinalizeSale(current supply, current proceeds,
                                       price, payment legs, delivery legs)
            S = av + seller, C = av
            check inventory, proceeds and pairing of sale legs
            return FinalizedAllocation values for existing IDs
              inventory allocation: extra sides = all delivery sender sides
              proceeds allocation: extra sides = all payment receiver sides
              both: nextIterationFunding = None
            consume SaleAuthority

         when an application attester is configured:
           -> BatchApproval.VerifyBatch
              S = attester, C = [av]
              verify full settlement reference, exact legs and expiry
              consume approval

         cash registry.SettlementFactory_SettleBatch
           transferLegs = all winner payment legs
           allocations = all accepted payment allocations + seller receipt allocation
           actors = [av]

         token registry.SettlementFactory_SettleBatch
           transferLegs = all winner delivery legs
           allocations = all accepted receipt allocations + seller inventory allocation
           actors = [av]

         require every allocation result to be Settled with no successor
         create ClearedRound, S = seller + av, observer = ao
         consume the closed Round
```

Everything indented under this `TX` commits atomically. `FinalizedAllocation`
is a value passed to the registry, not a new contract. Settlement consumes both
allocations for every accepted bid and the seller. Unused funds return to their
authorizing accounts, including the full payment reserved for zero-fill bids.

### Settlement authority

The factory uses existing allocation consent to perform asset operations:

```text
Grant.ClearRound
  S = av, C = ao
  -> Round_Clear
     S = seller + av, C = av
     -> cash SettlementFactory_SettleBatch
        S = cash admin, C = av
        -> each Allocation_Settle, actors = [cash admin, av]
           allocation S = cash admin + account owner + any provider
           body has those account parties' authority
           consume locked holdings, execute the net debit or credit,
             and create unlocked payout holdings

The token factory follows the same authority chain with token admin.
```

The enclosing factory supplies its admin and `av` authority to the allocation
choice. The allocation itself supplies the account parties' authority inside
that choice. Bid and sale finalization use this existing consent. Bidders and
providers make no new submission, although the participants required to confirm
the transaction must be available.

The application enforces the accepted set, pricing and bid bounds. The registry
checks matching sending and receiving sides for each transfer and performs net
account movements. All allocations keep the same full round settlement reference
through preparation and clear.

### Transfers and refunds

For each winner, payment moves from its payment account to seller proceeds,
and inventory moves from seller inventory to its delivery account. Each leg
has a deterministic ID based on the acceptance number. The two registries
receive the corresponding leg sets and authorizations.

| Account | Initially reserved | Movement | Returned or received unlocked |
|---|---:|---:|---:|
| Alice payment | 600 | Pays 500 | 100 returned |
| Bob payment | 800 | Pays 400 | 400 returned |
| Carol payment | 400 | Pays 100 | 300 returned |
| Seller inventory | 100 tokens | Delivers 100 | No remaining inventory |
| Seller proceeds | No funding | Receives 1,000 | 1,000 payment units |

The one-bid custody example pays 400 for 50 tokens. It returns Alice's remaining
200 and the seller's unsold 50 tokens. `preparedAllocationsAndRefunds` also checks
that all four original allocations are consumed with no successors.

For a zero-fill bid, both allocations have empty transfer sides at settlement.
The registry returns all reserved payment as unlocked holdings and closes the
unfunded receipt allocation. In `zeroFill`, Bob's 400 becomes available when
clear commits, with no payment to the seller and no delivery to Bob. The script
also checks this flow with Alice's custodied accounts.

An empty or entirely ineligible round records zero sales at reserve. Both
factories receive empty movement lists and finalize the seller's allocations
alongside any accepted bid allocations. All reserved inventory and payments
return to their original accounts, and outcomes have no recovery tickets.
Configured application approval still covers the full reference and empty leg
sets. Empty sides add no transfer authorization to the batch. They are not
zero-amount legs, which the registry rejects.

### Rollback and retry

`atomicRollbackAndRetry` supplies a valid application approval and cash context,
but omits the token account context. The cash factory runs, then the token factory
fails. The ledger restores the closed round, accepted bid, all four allocations
and the approval, with no committed proceeds or outcome. Supplying the token
context allows the same closed round to clear.

`noWinners` checks the same rollback when every accepted bid is excluded by
revocation or expiry. A token-registry failure restores the payments refunded
by the cash batch, the allocations, and the approval. The successful retry
returns all payments and inventory without a separate recovery transaction.

`BatchApproval` is an application-level check. The upstream Splice test token
has no OpenZeppelin D1/D2 extensions. See
[Controls](scripts/daml/Flows/Controls.daml) and
[Clearing](scripts/daml/Flows/Clearing.daml).

## 6. Failure and recovery

| Situation | Result and next step |
|---|---|
| Immediate bid creation lacks holdings or valid registry context | The whole submission rolls back. Supply valid inputs and context. |
| Custody approval is pending | Opening or bid admission waits for both completed allocations. Inputs must remain available until the registry reserves them. |
| Round funding lacks holdings or valid registry context | The whole seller submission rolls back, preserving preparation and inputs. |
| A round funding request is abandoned | Consume the request and withdraw its registry instructions or allocations under registry policy. |
| One batch input is invalid or reuses an accepted allocation | The entire batch fails, preserving its sources and the prior round. |
| A funded `PreparedBid` misses admission | The bidder can withdraw it and obtain a ticket for both allocations. |
| A registry-prepared bid misses admission | Abandon its request. Withdraw pending instructions under account policy, or completed committed allocations after the settlement deadline. |
| An allocation ID changes | Resolve the active successor and verify its root and unchanged specification. |
| A required allocation was cancelled | The complete clear fails. Cancellation is not an eligibility exclusion. |
| A clear definitively fails before the deadline | Retry the complete accepted set with valid current context and evidence. |
| A submission times out with an uncertain result | Reconcile command completion and ledger state before retrying. |
| Settlement time expires or the operator cancels the round | Record `EndedRound` and recover each bid and the seller allocations separately. |

### Recover through the operator

```text
TX  ao -> Termination Grant.EndRound(round, expired, reason)
      Grant S = av, C = ao
      -> Round.Round_End, S = seller + av, C = av
         require a reason and, if expired, time >= settlement deadline
         consume Round and create EndedRound, S = seller + av, observer = ao
         retain accepted membership and the sale reference

For each accepted bid, independently:
TX  ao -> Recovery Grant.CloseUnsettledBid(ended, bid)
      -> EndedRound.Ended_CloseBid, C = av
         check membership
         -> AcceptedBid.CloseBidWithoutSale, C = av
            consume bid and create Outcome + RecoveryTicket

TX  ao -> Recovery Grant.Release(ticket, current payment, current receipt)
      Grant S = av, C = ao
      -> RecoveryTicket.Recover, S = av, C = av
         validate both allocation bindings and active successors
         cancel payment allocation, actors = [av]
         cancel receipt allocation, actors = [av]
         return the released payment holdings and consume the ticket

Seller, independently:
TX  ao -> Recovery Grant.CloseUnsettledSupply(ended)
      -> SaleAuthority.CloseSaleWithoutSale, C = av
         consume sale authority and create a ticket for both seller allocations
TX  ao -> Recovery Grant.Release(ticket, current supply, current proceeds)
```

Tickets are signed by `av` and observed by `ao` and the affected bidder or
seller. Providers follow the release through their registry records. Abandoning
a funded proposal or immediate prepared bid also creates a ticket covering its
sending and receiving allocations.
Cancelling an unfunded receipt approval invalidates permission without moving
assets. Ticket release cancels the pair atomically. Different tickets can recover
in separate transactions, so another bidder's unavailable account need not
block that recovery.

### Withdraw through the registry

After the settlement deadline, the account parties can complete
`Allocation_Withdraw` without the operator. The account-withdrawal script submits
jointly as owner and provider for the payment allocation. The receiving approval
has its own lifecycle and can also be withdrawn under the registry's rules.

Before the deadline, an owner-only withdrawal on a custodied account can return
`Pending` and create an allocation successor without releasing committed funds.
Provider completion is blocked until the deadline. The owner cannot repeat its
approval, so this withdrawal has at most one pending successor in the pinned
registry. Its `originalAllocationCid` identifies the original allocation. The
`committedWithdrawalSuccessor` script rejects the stale allocation ID and then
clears using its active successor with `numIterations = 0`. The
`allocationReuseGuard` script changes both payment and receipt IDs and checks
that neither can be assigned to another bid. Other registry lifecycles need a
verified stable root or additional authenticated lineage evidence.

`custodyRecoverySuccessor` checks these ID changes after round cancellation.
It rejects stale IDs, releases the active allocations through their ticket, and
rejects a second release.

Abandoned `BidRequest` and `RoundFundingRequest` allocations are released
through registry withdrawal. Pending instructions can be withdrawn by their
existing approvers. Completed committed allocations follow registry deadlines
and account policy. The mixed round-withdrawal script demonstrates inventory
remaining locked until that deadline. Direct registry withdrawal can also leave
application records requiring reconciliation. Administrative cleanup of those
records is outside this implementation.

### Funding rollover demonstration

`fundingRollover` is a registry-only demonstration of retaining funding between
iterations:

```text
Initial allocation: Some {USD: 600}
First settlement: pay 200, nextIterationFunding = Some {USD: 300}
  return 100, create successor reserving 300, numIterations = 1
Second settlement: pay 150, nextIterationFunding = None
  return remaining 150, create no successor
Totals: sender recovers 250, receiver gets 350
```

The script rejects reserving 500 after paying 200 from 600. The auction itself
uses one final settlement with `None`. It rejects allocations that have already
executed a settlement iteration before clear.

## 7. Audit and governance boundaries

Account providers see their registry instructions, allocations, holdings and
asset movements. Auction records are shared with the seller, operator and `av`,
and each bidder sees its own bid and outcome. Providers receive no observer
access to auction records or recovery tickets, whether allocation approval is
immediate or staged.

Asset records reveal reserved payments and settlement amounts, and providers
serving several accounts can combine that information. An organization acting
as both provider and seller, operator or validation host also receives that
role's disclosures.

An auditor should retain accepted records before consumption and check:

1. Every accepted bid appears exactly once in the closed list with its round
   identity and acceptance number. Its sending and receiving roots are not reused.
2. Allocations match the expected accounts, instruments, funding, executor,
   deadline and settlement reference. Successors preserve their bindings.
3. Every eligibility exclusion has current provider-authenticated evidence.
4. Recomputed fills, price, rounded payments and totals match the aggregate result.
5. Each accepted bid has one outcome, and committed asset movements, refunds and
   retained recovery obligations reconcile with those outcomes.

[Audit](scripts/daml/Flows/Audit.daml) checks a retained closed-book snapshot,
private outcomes and proceeds against independently specified figures.
It is a script demonstration. Continuous history retention, request receipts,
failed-command evidence and admission-policy audits require additional services.

The intended deployment hosts `av` on the operator participant and two independent
validation participants, with two confirmations required. An auditor can observe
`av` without submission authority or a confirmation vote. The included scripts
run in the Daml test engine and do not configure or validate that topology,
participant outages or full production transaction projections.

[Privacy](scripts/daml/Flows/Privacy.daml) checks contract lookup and disclosure
access using each provider's identity during funding, bidding, acceptance,
settlement and recovery. Approval and settlement use the same registry workflows.
Production privacy also depends on transaction projections and the registry's
disclosure service.

Direct submission as `av` remains a trust boundary. In
`validationPartyTrustBoundary`, privileged `av` finalizes the bid and sale in
separate submissions, then settles only cash. The seller receives 400 while Alice
receives no tokens. Restricting direct `av` submissions and approved code is
therefore necessary to preserve the operator's complete-clear workflow.
[Authority](scripts/daml/Flows/Authority.daml) also rejects the operator's direct
attempts to cancel allocations or finalize a bid without its grant.

Grant instances govern separate operation categories. `StartRound`,
`FundRound` and `SubmitBid` require a live preparation grant. Revocation stops
new preparations and funding or bid requests. Existing registry instructions
can still complete. The independently governed opening and admission grants
allow those completed requests to open or be accepted before the cutoff. Close,
clear and recovery use their own grants. The pause scripts exercise preparation
and admission revocation separately.

## Scope and executable checks

The implementation uses two distinct asset admins, bounded whole-lot bids,
fixed account owners and at most 64 accepted bids. Its completed allocations
carry asset consent, while its application contracts authenticate terms and
control admission, clearing and recovery. Minimum timing margins, fees,
registry upgrades, external-party signing, administrative reconciliation and
production load testing remain implementation decisions outside this prototype.

| Script | Main checks |
|---|---|
| [Opening](scripts/daml/Flows/Opening.daml) | Immediate and custody round funding, both mixed states, fixed roots, one opening and preparation pause. |
| [Funding](scripts/daml/Flows/Funding.daml) | Seller authority, insufficient holdings, second-registry rollback, consumed inputs and funding cutoff. |
| [Bidding](scripts/daml/Flows/Bidding.daml) | Immediate bidder creation, authorization, preparation pause, batch admission and numbering. |
| [Clearing](scripts/daml/Flows/Clearing.daml) | Complete result, payments, deliveries, zero-fill refunds, empty rounds, and rollback and retry with no winners. |
| [Iterated](scripts/daml/Flows/Iterated.daml) | Custody instructions, receipt approval, allocation reuse, round references, refunds, rollover and active successors. |
| [Controls](scripts/daml/Flows/Controls.daml) | Credential revocation, missing allocations, application approval, rollback and admission pause. |
| [Recovery](scripts/daml/Flows/Recovery.daml) | Abandoned requests, instruction withdrawal, late admission, missed settlement, and successor checks after cancellation. |
| [Authority](scripts/daml/Flows/Authority.daml) | Operator restrictions and the direct-`av` counterexample. |
| [Audit](scripts/daml/Flows/Audit.daml) | Accepted membership, outcomes and proceeds. |
| [Privacy](scripts/daml/Flows/Privacy.daml) | Provider access to registry records, hidden auction records and recovery tickets, immediate and staged approval, settlement and recovery. |
| [Math](scripts/daml/Flows/Math.daml) | Price bands, proportional lots, exclusions and rounding. |

The scripts verify immediate allocation, custodian approval and mixed readiness
for both round funding and bids, alongside atomic rollback, registry withdrawal
and the complete auction flows. Source hashes and build adjustments are recorded in
[vendor/splice-test-token-v2/SOURCE.json](vendor/splice-test-token-v2/SOURCE.json).
Interface DAR versions, checksums and licenses are recorded in
[vendor/dars/manifest.yaml](vendor/dars/manifest.yaml). See
[README.md](README.md#run) for the run commands.
