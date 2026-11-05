# corebank — the idea, and what is left to build

A read-only brief. **No code lives here**, by design: this repo exists so the
idea and the remaining work can be read from any device without opening the
implementation. The code is at
[Knox-mgk/corebank](https://github.com/Knox-mgk/corebank) — **deliberately
private**, and it will 404 for you unless you are the author. This brief is
public so the design can be read and discussed; the implementation is not.

**Nothing described here holds customer money, and nothing is open to the
public.** One instance is installed on the author's own VPS. It binds to
loopback and is reachable only through an SSH forward, except for **exactly one
public path** — the provider's collections webhook, behind TLS and HMAC — while
every other route, including the dashboard and the admin API, returns 404 from
the internet (§5). The service runs on that host at this brief's commit; the
status board in §5 lists unfinished and known-broken work openly because that
is what makes a status board useful, and §3 sets out why no real money can move
until a licensed partner signs.

**Last synced with the code:** 2026-11-05, commit `8c63074`.
**Suite at that commit:** 924 unit tests, 0 failures, re-run on 2026-11-05 on
the author's phone (560s) under `-W always::ResourceWarning` with 0 warnings.
Plus 530 live assertions against real servers, all re-run the same day: 76
HTTP, 82 MCP, 144 dashboard, 65 MFA, 56 inbound, 107 DOM, 33 packaging, and a
123-assertion 15-phase lifecycle simulation.

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
    ├── parent_organization  OPTIONAL: a holding company above it (one level)
    ├── members (users)     humans: owner / admin / developer / viewer
    ├── api_keys            sk_live_* / sk_test_*, scoped to ONE environment
    ├── webhook_endpoints   where WE call THEM, with their signing secret
    ├── fee_schedule        per-organisation pricing
    ├── org_accounts        the entity's OWN books: operating / intercompany / expense
    ├── provider_config     who this entity is to a provider, and where its money lands
    ├── invoices            what it has billed its counterparties
    └── customers           the organisation's END users
        ├── kyc_record      tier, evidence, verification state
        ├── accounts        sub-ledger balances (one per currency)
        │   ├── virtual_account   a NUBAN for inbound money
        │   └── postings          immutable (by TRIGGER), the only source of balance
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

### Group structure: a holding company and its subsidiaries

An organisation may have a parent, **one level only** — deliberately, because a
nesting depth no code reads is not inert, it is a lie in waiting. What the group
work adds is three things an entity holds in its own name:

- **`org_accounts`** in three purposes: `operating` (its own money),
  `intercompany` (what it owes or is owed by a sibling) and `expense` (recorded,
  unremitted cost). Operating and intercompany are deliberately **separate
  accounts** — they used to share one kind, which meant every intercompany
  transfer was two legs against a single account: perfectly balanced, and moving
  nothing.
- **Intercompany transfers** as two entries, one per entity's own books. The
  ledger still refuses legs that straddle two organisations, so no path here
  grants one subsidiary write access to another's balance.
- **`consolidation_scope`**: a holding company's owner may *read* a subsidiary's
  figures. Statements, consolidated reporting and recurring expenses are all
  derived from postings at call time — nothing is cached, so nothing can drift
  from the ledger backing it.

The asymmetry between those last two is the point and is defended by a test:
consolidation scope is a **read** privilege and must never widen into a write
one. A provider configuration is not a figure — rewriting it redirects where
that subsidiary's revenue *arrives* — so the parent is refused there with the
same `CrossTenant` a sibling gets. Letting "I may read your books" silently mean
"I may change where your money lands" is the exact read-scope/write-scope
collapse the design refuses.

### Invoices, and what happens to money that arrives

An entity can bill a counterparty, mint a payment reference, and have an
inbound collection **matched to the invoice** that quoted it. The decisions
worth recording are all about refusing to guess:

- **An overpayment is not stored on the invoice.** There is no `overpaid`
  status: an excess is not part of the obligation, and putting it inside the
  invoice puts it somewhere no queue watches. The invoice is simply `paid` and
  the excess is **parked** in a platform account that reconciliation reports and
  a queue surfaces.
- **The provider's fee is not netted off the applied amount.** The counterparty
  paid in full; the partner's cut is our cost of collecting. Netting it would
  leave every invoice permanently short by the fee and make `part_paid` the
  normal end state of a fully paid bill.
- **A duplicate payment is neither refused nor dropped.** It has already
  arrived, so it cannot be refused; nobody is owed it, so it cannot be credited;
  dropping it would make the pool disagree with the partner forever. Nothing is
  applied and the whole amount parks.
- **One payment naming two entities parks too** — a quoted invoice of one
  subsidiary arriving at a NUBAN belonging to another. Picking the invoice
  credits an entity that may not be the intended recipient; picking the wallet
  ignores an instruction the payer gave. Either is moving money between two
  entities' books on a guess.

`invoice-drift` re-derives the cached totals from postings and **reports**
without repairing, with the same two correct answers every reconciliation check
here has: empty, or act now. Its `limit` parameter was silently ignored and was
fixed by **removing** it — a capped report returns 200 rows while the 201st
fault goes unmentioned, and the cron reading the exit code sees the same number
either way.

### Who we are to a provider, per entity

"Which provider is this, and who are we to it" used to be a process-wide fact
read out of an environment variable. For a group that is wrong in a way that
costs real money: each entity signs its **own** agreement, holds its **own**
merchant identifier, and expects settlement in its **own** operating account —
and none of those three facts had anywhere to live, so the destination of
incoming money was decided by convention at the call site.

It is a row now, with an **append-only revision log** (enforced by trigger, in
the shape the postings table uses), so a rotation cannot be made to look like it
never happened. Two refusals define the surface:

- **There is no secret column here.** A merchant identifier is not a secret —
  the provider prints it on every settlement advice — but the key that
  *authenticates* as that merchant is. Adding an `api_key TEXT` would put live
  provider credentials in every backup, in `backup --verify` output, and one
  careless `SELECT *` away from an audit log. The authenticating key stays in
  the environment of whichever process is authorised to move money. **This table
  says who we are; it does not say how to prove it.**
- **Reads are not at a lower bar than writes**, which is the one place this
  codebase inverts its own habit. A merchant identity is the handle a payer
  quotes and the string an attacker needs to forge a settlement advice, so
  "which merchant id does my sibling use" is not a reporting question. Both read
  and write answer 404, never 403, so neither can be used as an existence oracle
  over another entity's banking arrangements.

A settlement account must be an account the **same** organisation holds in its
own name, checked against `org_accounts` rather than `organization_id` alone —
both catch a sibling's account, but only the index catches a wallet, a platform
custody account, or a legacy row carrying the right id that is not an account
the entity holds. And a merchant identity may be claimed by only **one** active
configuration, because a settlement advice names a merchant, not an
organisation, and two claimants means two valid destinations for real money.

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
| 3 | Pooled settlement + the custody invariant | **done** |
| 4 | Inbound: virtual accounts + collections webhook, and identity capture pulled forward | **done** |
| 5 | Outbound webhooks: signing, retries, dead-letter, replay | **done** |
| 6 | KYC tiers + limit intersection | **done**, inbound and outbound |
| 7 | Org-scoped fees replacing the single schedule | not started — **next** |
| 8 | Postgres | not started — blocks launch, not development |

Work that was not on the eight-step order and landed anyway, each closing a
section of the group directive:

| | state |
|---|---|
| Ledger append-only in the **database** (triggers), not just in the code | **done** (migration 12) |
| Group structure: subsidiaries, `org_accounts`, intercompany transfers, consolidation scope | **done** (§2.1, migration 14) |
| Invoices and invoice-matched collections | **done** (§2.2) |
| Subsidiary statements, consolidated reporting, recurring expenses | **done** (§2.3, migration 16) |
| Per-organisation provider identity and settlement arrangement | **done** (§2.4, migration 17) |
| CoreBank as a payout provider for an external platform | **done** (§4) |
| Payout lifecycle hardening: no second debit, no silent hold, no leaked conflict | **done** (migration 12) |
| Restore drill that is honest about a schema bump | **done** |
| Platform-admin role collapse and owner-scoped agent authority | **done** — both were live holes |
| Dashboard: a readable ledger view, and the backup half nothing automated | **done** |
| CSP: the inline block authorised by a per-response nonce, not `'unsafe-inline'` | **done** |
| Payout provider **selected explicitly**, never inferred from a credential | **done** |
| `connect()` closes its handle when the file turns out to be unreadable | **done** 2026-11-05 |
| Deployed on the author's VPS, one public path (collections webhook) | installed and **running** at this brief's commit, schema 17 |

Nothing is reachable by a customer, and the pooled balance is a sandbox figure.

**Deployed alongside the build, 2026-10-27: a public TLS edge.** Not a build
step, but it gated everything inbound. The service had run loopback-only since
deployment, so the collections webhook of step 4 was a code path with no
possible caller. A dedicated subdomain now terminates TLS and proxies **exactly
one path** — the collections webhook — while every other route, including the
dashboard and the admin API, returns 404 from the internet and is reachable
only through an SSH tunnel. A path that is not routed cannot be enumerated.
Verified from off-box with five negative controls (unsigned, forged hash,
valid HMAC over a different body, wrong secret, spoofed forwarding header)
before any positive one: 11 of 11.

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
`-W always::ResourceWarning`.

**The resource-warning check then found a third leak, on 2026-11-05, and the
interesting part is why it was nearly dismissed.** `db.connect()` opens the
file and then issues four PRAGMAs — and those PRAGMAs are the first *reads*, so
a corrupt or truncated database raises there: after the handle exists, before
the caller receives it. Nobody can close a connection they were never given, so
the descriptor and its WAL sidecar survived until the garbage collector ran,
and the warning was then printed against whatever source line happened to
trigger collection. It blamed a file in the standard library. Both repos now
close the handle on the failure path, with a four-assertion test — including a
negative control proving the leak detector can fail, and an end-to-end check
through the restore drill, which is the only caller that opens
deliberately-unreadable files by design.

The lesson is about diagnostics, not about sqlite: **a resource warning names
the line where collection happened, not the line that leaked.** A leak reported
inside unrelated code is the normal presentation, so "that warning is from a
dependency" is the wrong conclusion to reach cheaply — `-X tracemalloc` names
the real allocation site in one run.

**And then the guard against the first leak turned out to be scanning the wrong
half of the repo.** The AST sweep that has pinned "never read an `HTTPError`
body without closing it" since the original socket leak only ever walked the
package directory. Eighteen handlers in `tests/` and `scripts/` were still
leaking — and those leak harder than the package ones ever did, because a smoke
script's request helper runs on *every* refusal the script asserts. One test
helper alone accounted for 30 resource warnings in a full suite run; the count
is 0 now, and the sweep walks the package, `tests/`, `scripts/` and
`scripts/dev/`. Test and script helpers are also exactly the code a human
copies when writing the next adapter, so excluding them guarded the half that
mattered least. **A guard is only as good as its scope, and nothing re-checks
the scope once the guard is green.**

A related hole was found and fixed in **aibank**, the single-tenant parent:
new users defaulted to organisation `owner`, letting any account holder read
and rewrite another's agent limits. Same class of bug, same fix at the method
rather than the router.

Carried over from aibank and explicitly not to be rebuilt: spend controls, the
MCP server, TOTP second factor, the hold sweeper, verified backups, and the
deployment artifacts.

---

## 6. What is next

**Steps 1 through 6 are done, and step 7's commit closed the outbound half of
the KYC gate. The next code step is org-scoped fees.**

The real blocker is not code. Two items sit outside the repo and no amount of
engineering moves either:

1. **The payout product is not enabled on the provider account.** A live
   transfer attempt returns `400 "merchant is not enabled to make transfers"`
   — the IP whitelist is satisfied, the key authenticates, and the refusal is
   an account entitlement the provider grants after compliance review. Nothing
   in this repo changes that answer.
2. **The webhook URL is not yet registered in the provider dashboard.** The
   edge is live and verified, but until the URL is pasted in, zero real
   deliveries arrive — so inbound settlement is proven correct and still
   untested against actual provider traffic.

Nothing is deployed for public use, nothing holds customer money, and there is
no production instance.

### Found 2026-11-04: the fork had silently regressed three of aibank's authorisation fixes

The reason to record this is that it is a **class** of bug a fork creates and
a green suite cannot see. aibank fixed three cross-user authorisation holes
after the fork point. corebank's suite was 549 green at the time, so nothing
pointed at them — but the holes were back, because the fixes landed upstream on
methods corebank had already rewritten for tenancy.

Proved by probe before fixing, in corebank's own library (not over HTTP):

- **Agent limits.** One account holder could raise another's agent ceiling and
  freeze their agent. This is the exact inverse of the product's central claim,
  and it was the headline bug aibank fixed on 2026-10-21.
- **`freeze_agent`** took no owner check at all.
- **Payout approve and reject** reported another user's payout status before
  refusing, which both discloses the row and lets a stranger drive someone
  else's approval queue.

All three are now checked on the **method**, under the same ownership rule the
tenancy scoping uses, with a 14-assertion cross-user isolation test that fails
when any one guard is removed. Full suite 563 green afterwards.

The lesson, kept because the next fork will repeat it: **a fork inherits bugs
but not fixes.** A fix applied upstream after the fork point does not exist
downstream, and if the downstream copy rewrote that method for its own reasons,
no merge conflict will ever tell you.

### Found 2026-11-05 on *pushed* main: a merge deleted that fix, and the suite stayed green

The sequel is worse than the original, because this time the fix existed in
`git log` and not in the code. A merge resolution reordered
`require_agent_authority` so the organisation-staff bypass was consulted
*before* the caller's asserted identity — and that flag is true for the
unauthenticated local handle every CLI, sweeper and library caller receives. A
supplied `user_id` was therefore discarded entirely, and one account holder
could again raise another's agent ceiling or freeze their agent through the
bare library. Measured on the pushed commit: 2 of the 14 isolation assertions
failed.

The ordering *is* the fix. An explicit identity **narrows** authority and never
widens it, so the identity check goes first and the bypass is reached only when
no identity was asserted. The tenant's own `user_id` still outranks the
argument — trusting the argument first would be no authorisation at all, since
a caller would simply pass the victim's id.

Two things generalise:

- **A merge is a code change no review diff shows you.** The conflict
  resolution *is* the edit, the file reads plausibly either way, and nothing
  re-ran the one test file that attacks through the bare handle.
- **The suite was green because the only test that could see it was not run.**
  A security assertion that lives in exactly one file is a single point of
  failure in the process, not just in the code.

### Step 5 shipped 2026-10-23: corebank is now somebody else's provider

Being the platform means inheriting every obligation corebank currently imposes
on its own provider: sign the exact bytes sent, carry a stable event id, retry,
expose a dead letter, and allow a replay. That is what landed — 77 tests
against a real local HTTP receiver rather than a mock, because a mock proves
the code calls a function and nothing about what crosses the wire.

The guarantee that justifies the module is narrower than it sounds. A
**duplicate** event is something any integrator can catch for themselves using
the event id. A **missing** event is not — silence looks exactly like nothing
having happened, and the first time anyone notices is a customer asking where
their money went. So every event carries a sequence number that is monotonic
per organisation and shared by all of that organisation's endpoints, which
makes a gap provable from the receiving side. Numbering per endpoint instead
would have been the obvious choice and would have destroyed the only property
worth having.

Four decisions in the same shape:

1. **A destination is checked by resolved IP address, not by hostname.** Only
   https, no query string, and loopback, private and link-local ranges are
   refused — a public-looking name can resolve to the cloud metadata address,
   which hands out the server's own credentials. Redirects are refused rather
   than followed, because a redirect is precisely how an allowed public URL
   becomes an attack on our own infrastructure.
2. **The signing secret is shown exactly once and sealed at rest** under the
   same key material as the second-factor secrets. It never appears in a
   listing, an audit row or a delivery record: a stolen database dump must not
   confer the ability to forge corebank's own webhooks. A secret that will not
   unseal is dead-lettered immediately rather than retried, because retrying
   cannot help and would bury the real incident under a delivery failure.
3. **The event is written inside the money transaction; the HTTP call is not.**
   An event therefore rolls back with the payment it describes and can never
   announce a credit that did not happen — while a slow integrator endpoint can
   never hold a lock on the ledger.
4. **A 4xx is retried too, and everything is bounded** (8 attempts, backing off
   to an hour, then dead). A client error usually means an integrator just
   deployed a broken handler and will fix it within the hour; refusing to retry
   converts their short outage into permanent data loss. Bounded, because
   hammering a genuinely wrong URL forever is a self-inflicted outage with our
   name on it.

Replay resends the stored bytes verbatim and refuses an event that already
succeeded; re-serialising would change the signature and prove nothing about
what was originally sent. Isolation is attacked through the internal service
handle rather than over HTTP: no organisation can list, rotate, disable or
replay another's endpoints, and a test-environment event cannot reach a live
endpoint.

**The step also found a bug in the test infrastructure that had nothing to do
with webhooks, and it is the more instructive half.** The suite's throwaway
databases were only cleaned up when the process exited, so a full run needed
every one of them to exist simultaneously — roughly 280 MB. On a development
machine with a full disk the run died partway through and reported 269
failures, every one of which looked like a product regression and none of which
were. The rule extracted: when a wall of identical failures has its deepest
stack frame inside the standard library rather than in product code, it is the
environment failing, not the product. Read the innermost frame before believing
a test count.

### Step 4 shipped 2026-10-22: money can now arrive, and parked money has an owner

Each customer can be issued account details to pay into; a collections webhook
credits the right wallet; and money that cannot be attributed is **held**
rather than dropped or guessed at. Identity capture came with it, because the
probe below proved it is a prerequisite rather than a later refinement.

The finding worth carrying is a column that could not name a human. `who
resolved this parked payment` was first written as a foreign key to the users
table — the obvious, correct-looking choice. It is wrong, and a green test
suite would never have said so: the person who resolves parked money is an
operator at the command line, who has no row in that table at all. The key
therefore guarantees one of exactly two failures on the only path that ever
attributes money — a null actor, or an integrity error that aborts the
transfer itself. The column created to answer "who decided this" would have
been empty precisely when an auditor asked. It now stores a qualified
principal with no constraint, because naming a human the system does not
employ is the entire point. The audit log had the identical hole one layer
down and took the same fix.

Three smaller decisions in the same shape:

1. **Why money was parked is its own field**, not borrowed from the provider's
   narration. That narration is often the only clue to who sent the payment,
   so overloading it destroys the evidence needed to resolve the very row it
   describes.
2. **Resolving a payment by hand re-runs the identity limit check.** Hand
   attribution is the obvious back door around a tier cap, so it is refused by
   default; the override needs the owner role and a written reason, and it is
   recorded as an override rather than as a normal credit. An override that
   logs like an ordinary credit is not an override, it is a hole with
   paperwork.
3. **Unattributed money is a real ledger position**, so the custody invariant
   from step 3 still balances while the owner is unknown.

435 unit tests (93 new) and six live smoke suites pass, with negative controls
run on each new guard.

### Step 3 shipped 2026-10-21: the custody invariant can now fail

It could not before, and that was the real finding. A settled payout credited
an "external" account and left the settlement account untouched, so the
ledger's idea of what the partner holds never fell when money went out. The
invariant was wired to a number that only ever moved one way — an alarm that
cannot fire, which is worse than no alarm, because the dashboard reads green.

What exists now:

1. **The pool is a real ledger account.** Deposits raise it, settled payouts
   drain it, and the partner's fees come out of it explicitly — because a pool
   that silently shrinks by the provider's cut fails the invariant every single
   day, and the first instinct is to widen the tolerance until it detects
   nothing.
2. **Reconciliation records the difference and corrects nothing.** The gap
   between our books and the partner's figure posts to an unexplained-difference
   account. The only way it leaves the books is a human naming where the money
   actually belongs; there is deliberately no write-off function, and a test
   asserts the forbidden function names do not exist.
3. **The command exits non-zero while a difference is open** (2 for an
   unexplained gap, 3 for our own postings disagreeing about the pool — a
   different incident, and a worse one), so cron alerts without parsing output.
4. **Money in flight is reported, never subtracted.** A submitted payout may or
   may not have been debited at the partner yet. That band of genuine ambiguity
   is labelled and still alarms, because auto-forgiving it silences the check
   during exactly the window when a payout can go wrong.

342 unit tests (51 new) and five live smoke suites pass. Both negative controls
were run: reverting the settlement path fails a test, and making reconciliation
self-correct fails eleven.

Step 2's isolation work is what made this safe to build: cross-tenant writes
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

So step 4 split, and both halves have now landed: identity capture came first,
and permanent per-customer account details issue on top of it. The limit
intersection stays at step 6.

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
- **Live payout rails are still unproven, but the blocker MOVED on 2026-10-23.**
  From a fixed egress address the provider no longer answers "Please enable IP
  Whitelisting" — it answers "merchant is not enabled to make transfers". The
  network-level obstacle is cleared; what remains is an account entitlement the
  provider must switch on after compliance review. Reads are proven live (598
  banks, balance, account resolution); writes are refused at the account level.
- **Postgres (step 8)** blocks launch and nothing before it. The current write
  path holds a process-global lock, so correctness depends on exactly one
  process existing — acceptable for a single-tenant deployment, an availability
  ceiling for a platform.

### Known traps, recorded so they are not rediscovered

- **A fork inherits bugs but not fixes.** Three authorisation fixes applied to
  aibank after the fork point were absent here and invisible to a 549-green
  suite (see §6). Any fix landing upstream has to be checked against this tree
  by hand, because tenancy rewrote the methods the fixes live on and git will
  report no conflict.
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
  invariant is asserted alongside it.
- An invariant wired to a number that can only move one way reads as green
  forever. Before step 3 the custody check compared our books against a
  settlement balance that payouts never touched, so it could not fail. When a
  check has never fired, the first question is whether it is able to.
- The pool is carried NEGATIVE while it holds money, because wallets are
  carried positive-when-owed and the ledger sums to zero. The inversion lives
  in exactly one function; a second place that negates reports a shortfall as
  a surplus.
- **A wall of failures is usually one broken harness, not N broken features.**
  A new test file reported 528 errors; 265 of them were the harness itself — a
  single self-referential fixture failing dozens of unrelated assertions at
  once and making the product look broken in dozens of places. Debug the
  harness to honesty *first*; the assertions that still fail afterwards are the
  real findings, and there were several, because a harness written far enough
  to fail is exercising paths nothing else did.
- **A `-W always::ResourceWarning` run blames the collector, not the leak.** The
  warning is printed wherever the GC happened to fire, which is routinely a
  file in the standard library. Use `-X tracemalloc` before concluding the leak
  belongs to someone else's code.
- **A deployment that only ever pulls should hold a read-only, per-repo key.**
  This host pulled with a key authorised against the whole account, so a
  compromise of a box that never needs to write could have rewritten every
  repo the author owns. It now uses a repo-scoped deploy key marked read-only,
  verified by GitHub refusing a dry-run push rather than by trusting the
  checkbox. A pull-only role should be unable to push even when the attacker
  is root on the puller.
- **Scope a structural guard to the whole repo, not to the package.** An AST
  sweep that forbids a missing statement is only as good as the directories it
  walks, and nothing re-checks that scope once it is green. Here it skipped
  `tests/` and `scripts/` and missed eighteen live instances of the exact bug
  it existed to prevent — in the helpers a human is most likely to copy.
- **The demo seeder refuses to run as root**, because a convenience script that
  writes a known-credentials database is a production incident waiting for the
  one night somebody runs it on the server.
- A credential is not an authorisation: the payout provider is **selected
  explicitly**, never inferred from the presence of a key. Inferring it means a
  pasted environment variable is the only thing between a deployment and real
  money leaving a real account.

---

## 7. Note on the code repo's own README

**Fixed on 2026-11-04 (`dd03b19`).** This section used to say that
`Knox-mgk/corebank`'s `README.md` was still aibank's — describing a self-hosted
deployment that never holds money, contradicting §1 and §3 above — and that
this brief was the accurate description until it was rewritten.

That rewrite has now landed. The code repo's README opens with the BaaS shape,
the pooled balance, the custody consequence and the build-order position, and
it marks the inherited single-tenant sections as exactly that. The two
documents now agree, and `docs/architecture.md` remains the authority on the
regulatory shape.

One contradiction was found and fixed in both at the same time: both READMEs
told a reader to install over public HTTPS from a repo that has been private
since 2026-10-21, so the first command anyone ran 404'd.
