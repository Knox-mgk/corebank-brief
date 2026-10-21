# corebank — the idea, and what is left to build

A read-only brief. **No code lives here**, by design: this repo exists so the
idea and the remaining work can be read from any device without opening the
implementation. The code is at
[Knox-mgk/corebank](https://github.com/Knox-mgk/corebank) — **deliberately
private**, and it will 404 for you unless you are the author. This brief is
public so the design can be read and discussed; the implementation is not.

**Nothing described here is deployed, and it holds no money.** The status board
in §5 lists unfinished and known-broken work openly because that is what makes
a status board useful — none of it is reachable by anyone, there is no running
instance, and §3 sets out why no real money can move until a licensed partner
signs.

**Last synced with the code:** 2026-10-21, commit `653c540`.
**Suite at that commit:** 291 unit tests + 451 live assertions, 0 failures,
0 resource warnings under `-W always::ResourceWarning`.

---

## 1. The idea in one paragraph

corebank is **Banking-as-a-Service for Nigerian rails**: a multi-tenant,
double-entry core that holds one pooled balance at a licensed partner and issues
sub-ledger accounts against it to organisations who integrate the API. Those
organisations create their own end customers, each customer gets a real account
with a virtual NUBAN that can receive money, and money can be paid out to any
Nigerian bank account. On top of that sits the feature it was forked from:
**AI agents that can spend money under limits they cannot reach.**

It is forked from **aibank** — code at `Knox-mgk/aibank`, now **private**; the
public brief for it is [aibank-brief](https://github.com/Knox-mgk/aibank-brief).
The fork inverts aibank's defining property:

| | aibank | corebank |
|---|---|---|
| who runs it | the customer | us |
| who holds the money | the customer's own provider account | a pooled account corebank operates |
| tenancy root | a user | an organisation |
| whose balance is it | the deployer's | the organisation's end customers' |
| inbound money | admin-gated stand-in | virtual accounts + collections webhooks |
| webhooks | received only | received **and sent** |
| regulatory surface | none | real (see §3) |
| blast radius of a bug | one deployment | every tenant at once |

The ledger primitives carry over unchanged, and that is why the fork is cheap:
integer minor units, balanced entries, immutable postings, holds, idempotency by
UNIQUE constraint, trial balance zero. What they lacked was a tenancy boundary
above them and any story for money that is not already ours.

---

## 2. The two invariants

**The authority invariant (inherited, still true):**

> The human sets the limits. The agent operates under them and can never reach
> them.

Enforced structurally — the route that changes an agent's policy is reachable
only with a human session, so no sequence of agent API calls can raise the
agent's own ceiling.

**The custody invariant (new, and the one the business rests on):**

> The sum of every sub-ledger balance equals the pooled balance held at the
> partner, minus money in flight — continuously, and provably.

aibank could not violate this because aibank held nothing. corebank can violate
it a dozen ways: a collection credited twice, a payout debited but never
submitted, a provider fee taken out of the pool that the ledger never sees, a
refund that lands without a webhook. Every one shows the same symptom — the
database disagrees with the bank — and every one is, legally, somebody else's
money.

What that forces:

1. The pooled account is a **ledger account**, not a config value. No
   "provider balance" field exists anywhere.
2. Reconciliation compares ledger to partner and posts the difference to
   `reconciliation:unexplained` **rather than adjusting anything**. A non-zero
   balance there is an alarm, not a correction.
3. Reconciliation **exits non-zero** when unexplained ≠ 0, so a timer can alert
   without parsing output.
4. Provider fees get their own account — otherwise the pool shrinks daily, the
   invariant fails daily, and the first instinct is to widen the tolerance,
   which turns the alarm into decoration.

---

## 3. The regulatory shape — DECIDED: sponsored

Holding customer balances in Nigeria needs an e-money/wallet licence. An MMO
licence means capital in the billions of naira plus CBN escrow; a PSSP licence
is **explicitly barred from holding customer funds**. aibank's whole shape was
chosen to step around this. corebank walks back into it, so the shape had to be
chosen deliberately:

- **B1 — Sponsored. ← this is what is being built.** A licensed partner (micro-
  finance bank, commercial bank or MMO) is the licence holder and the pooled
  account is in **their** name, on their balance sheet. corebank is the
  technology: it keeps the sub-ledger, issues virtual accounts under the
  partner's NUBAN range, and instructs payments over the partner's rails. This
  is what Anchor, Bloc and Fincra do, and it is the only version that can exist
  before a licence or a partnership exists on paper.
- **B2 — Own licence.** corebank holds the licence and the float. Billions in
  capital plus escrow. Not reachable; revisit only if B1 earns revenue.
- **B3 — Pure sub-ledger, no custody.** Books for money held elsewhere, closer
  to Modern Treasury than to Unit. Legally safest and a real product, but the
  organisation still needs someone else for the money, so it is not the thing
  asked for.

**What B1 means for the code, and it is not cosmetic:** the partner — not
corebank — is the authority on what a balance is. The ledger therefore treats
the partner as an **external system of record it reconciles against**, never as
a mirror it assumes agrees. That is the whole reason the unexplained-difference
account exists and refuses to self-correct.

**Hard line until a partner signs: not one naira of real money.** The build
target is a complete system running against a partner sandbox plus a fake
provider. Writing the code is legal; holding the float is not. The licensing
position needs a Nigerian fintech lawyer before anything goes live — the
engineering is deliberately shaped so the legal answer changes a credential and
a partner adapter, not the ledger.

---

## 4. Shape of the system

```
platform (us)
└── organization            a fintech customer; the tenancy root
    ├── members (users)     humans: owner / admin / developer / viewer
    ├── api_keys            sk_live_* / sk_test_*, scoped to ONE environment
    ├── webhook_endpoints   where WE call THEM, with their signing secret
    ├── fee_schedule        per-organisation pricing
    └── customers           the organisation's END users
        ├── kyc_record      tier, evidence, verification state
        ├── accounts        sub-ledger balances (one per currency)
        │   ├── virtual_account   a dynamic NUBAN for inbound money
        │   └── postings          immutable, the only source of balance
        ├── agents          OPTIONAL: AI principals with spend policy
        └── payouts         outbound money, with holds
```

Three things here are load-bearing:

- **`organization_id` sits on every non-global row**, denormalised rather than
  reached through a join — a tenant predicate that depends on a join chain is
  one refactor away from being dropped, and the resulting bug is cross-tenant
  disclosure.
- **An agent belongs to a customer, not to an organisation.** Agent spend
  control then becomes something an organisation offers *its* users, which is
  the part worth selling.
- **`environment` ('live' / 'test') is a column, not a second deployment.** One
  schema, one process, hard-partitioned by a column every query filters on,
  because two deployments drift and a sandbox that behaves differently from
  production is worse than no sandbox. Test money is posted by a fake provider
  and can never reach a real rail.

### Five trust domains

| domain | credential | may do |
|---|---|---|
| agent | `sk_agent_*` | request payouts under policy, read own balance |
| customer | end-user session | see own accounts, move own money |
| organisation | `sk_live_*` / `sk_test_*` | create customers and accounts, move money between its OWN accounts, set agent policy, read its own data |
| org member | password session + TOTP | rotate keys, change fee-affecting settings, approve payouts, manage members |
| platform | operator key, separate credential store | pricing, partner settlement, reconciliation, suspend an organisation |

Enforced structurally, not by review: an org key can never widen its own limits
or fees; an org key reaching another organisation's object gets **404, not 403**
(distinguishing "does not exist" from "not yours" is itself a disclosure);
suspending an organisation stops every customer, agent and payout beneath it;
and a `test` key presented with a `live` resource id gets 404.

### What is uncontested about this

Every agent-payments player — Catena Labs, Payman, Skyfire, Visa Intelligent
Commerce, Mastercard Agent Pay, Stripe+OpenAI ACP, Coinbase, Circle — settles on
US cards, ACH or USDC. **None of them pays a Nigerian bank account.** The
open-source self-hosted niche is also already taken (AgentLedger, Steward). So
the defensible position is not "agent spend controls"; it is **African payout
and collection rails**, with the agent controls as the differentiating feature on
top.

---

## 5. Where development actually stands

The build order is eight steps, chosen so a bug in a lower layer cannot hide
under a higher one. Each step ends with a green suite and a commit.

| # | step | status |
|---|---|---|
| 1 | Tenancy migration (schema v6) | **done** |
| 2 | Tenant-scoped reads and writes + second-org attack test | **done** |
| 3 | Pooled settlement + the custody invariant | **next** |
| 4 | Inbound: virtual accounts + collections webhook | not started — identity is a prerequisite, see below |
| 5 | Outbound webhooks: signing, retries, dead-letter, replay | not started |
| 6 | KYC tiers + limit intersection | not started (research done; storage half is now a step-4 dependency) |
| 7 | Org-scoped fees replacing the single schedule | not started |
| 8 | Postgres | not started — blocks launch, not development |

**Step 1 — done.** `organizations`, `organization_members`, `customers` and an
`environment` column; `organization_id` denormalised onto every scoped table.
31 tests, including an upgrade from the old schema seeded with raw SQL and real
money in it.

**Step 2 — done as of 2026-10-21, and the dangerous half is now the tested
half.** Landed: a scoped tenant handle, with the data layer taking the tenant
predicate explicitly so a query that forgets it shows up in the diff; refusals
are 404 rather than 403, because distinguishing "not yours" from "does not
exist" is itself a disclosure.

Four holes were open on 2026-10-12 and are now closed, each pinned by a test
that fails if the scoping is removed:

- **The policy read and write paths were not tenant-filtered.** Reproduced with
  real money at the time: organisation B raised organisation A's agent ceiling
  from 1,000 to 39,000 NGN, and A's agent then moved 35,000 against a 1,000
  limit — the exact inverse of the product's central claim. Scoping now lives on
  the METHOD rather than in the HTTP router, so the library beneath the API is
  safe on its own; the router's check is defence in depth.
- The reconciliation listing was unfiltered — B could see A's flagged payouts.
  Platform scope is now an explicit branch rather than an absent predicate, so
  a tenant-scoped caller cannot silently inherit deployment-wide reach.
- Asking for another tenant's user balance raised a database integrity error
  instead of a clean 404, leaking existence through the error shape.
- **No second-organisation attack test existed**, which is why the above
  survived a green run. There are now 13, attacking through the bare service
  handle rather than over HTTP — an HTTP-only isolation test certifies the
  router, not the code that will outlive it.

The same audit found four further bugs the green suite could not see, plus a
socket leak on every refused provider call that only appears under
`-W always::ResourceWarning`. Both repos now run 0 resource warnings.

A related hole was found and fixed in **aibank**, the single-tenant parent:
new users defaulted to organisation `owner`, letting any account holder read
and rewrite another's agent limits. Same class of bug, same fix at the method
rather than the router.

Carried over from aibank and explicitly not to be rebuilt: spend controls, the
MCP server, TOTP second factor, the hold sweeper, verified backups, and the
deployment artifacts.

---

## 6. What is next

**Step 2 is done. Step 3 — pooled settlement and the custody invariant — is
next, and it is the step the whole custody decision in §3 exists to serve:**

1. An unexplained-difference account, so the gap between our ledger and the
   partner's statement lands somewhere visible instead of being quietly
   absorbed.
2. Reconciliation against the partner as the external system of record — the
   partner, not us, is the authority on what a balance is.
3. A reconciliation command that exits non-zero when there is a difference, so
   cron alerts without anyone parsing its output, and that refuses to
   self-correct. Auto-agreeing the books with the bank is how a shortfall
   becomes invisible.

Step 2's isolation work is what made this safe to start: cross-tenant writes
beneath a pooled balance are how one tenant spends another tenant's money, and
until the attack test existed nothing in a green suite would have reported it.

### Issuing account details: probed 2026-10-21, and it reorders the build

"Give each customer their own account number to pay into" is the feature a BaaS
is FOR, and it sat at step 4 with KYC deferred to step 6 — on the assumption
that identity tiers only cap what an account may do once it exists. A direct
probe of the provider's sandbox says the dependency runs the other way:

- A **permanent** per-customer account number is refused outright without a BVN
  or NIN: `400 — "BVN or NIN is required for static account number"`. That is
  not a provider preference. The number lands on a deposit-taking institution's
  core banking system, and under the CBN tiered-KYC framework that institution
  owes the identity record — so no amount of ledger work issues one. Identity
  capture, peppered hashing and verification must exist FIRST.
- A **dynamic** account issues with no identity at all, but comes back with a
  one-hour expiry and single-use frequency. That is a checkout primitive, not a
  wallet address: it cannot hold a recurring balance, so it does not satisfy
  the inbound requirement on its own.

So step 4 splits. The checkout variant is buildable now; per-customer account
details need step 6's storage half landed first. The limit intersection stays
at step 6.

Two further findings worth recording, because both would have surfaced as
confusing failures later rather than as design facts now. The sandbox mints
**one shared mock account number for every customer**, so a second issuance
collides on the uniqueness constraint — which is the constraint working, since
two live customers sharing a number would credit the wrong person's money. It
stays, and attribution gets tested against a faked provider instead. And the
response carries a bank **name** but no bank code and no account name, both of
which the schema requires — so the account name has to be derived from the
customer record we already hold, or the first real issuance fails on an
integrity error after the money side is already correct.

**The real blocker is not code.** §3 picks the sponsored shape, which needs a
licensed partner to sign before any of this can hold a naira. No amount of
engineering substitutes for that signature, and it is the item that will still
be blocking when every one of the eight steps is green.

### Blocked on a decision or on something outside the code

- **No partner. This is the real blocker on the whole venture, not a code item.**
  B1 requires a licensed partner whose name the pooled account sits in. Until
  one signs, corebank can only run against a sandbox. Candidate shapes to
  approach: a microfinance bank, or a BaaS aggregator (Anchor / Bloc / Fincra).
- **A Nigerian fintech lawyer** needs to confirm the B1 position before anything
  touches real money.
- **A CAC-registered company** is a precondition for partner KYB.
- **Live payout rails are unproven even on aibank**: the real provider returns
  "Please enable IP Whitelisting" on transfers. Reads are proven live; writes are
  not. Needs a fixed egress address — a phone on CGNAT cannot be whitelisted.
- **Postgres (step 8)** blocks launch and nothing before it. The current write
  path holds a process-global lock, so correctness depends on exactly one
  process existing — acceptable for a single-tenant deployment, an availability
  ceiling for a platform.

### Known traps, recorded so they are not rediscovered

- Idempotency keys are scoped per **organisation** now. A global key namespace
  returns someone else's payout record.
- Fail closed on the absence of a **bound**, never the absence of a row. The
  creation path inserts defaults, and a row of NULLs is not a limit — this was a
  real bug, where the least-configured agent had the most authority.
- Daily windows come from a **stored offset**, not the host's timezone database.
  A "daily" cap resetting at UTC midnight resets at 01:00 in Lagos and hands the
  agent a second allowance in that hour.
- Balances are derived, never stored. A cached tenant balance will be the first
  thing proposed for performance and the first thing to drift.
- Trial balance zero is asserted after every money test, and the custody
  invariant will be asserted alongside it.

---

## 7. Note on the code repo's own README

`Knox-mgk/corebank`'s `README.md` is still aibank's, describing a self-hosted
deployment that never holds money. That contradicts §1 and §3 above and is a
known stale document — **this brief is the accurate description of the shape**
until the code repo's README is rewritten (it is not worth rewriting before step
3 lands, since the custody model is what the README has to explain).
