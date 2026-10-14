# The Town — an economics lab you can argue with

A small town that runs itself. Nine agents, two shared artifacts, and a
clock. Price is not a formula anywhere in this code — it is the residue of
firms undercutting each other and households deciding what they are willing
to pay.

Built on [JaCaMo](https://jacamo-lang.github.io/): Jason agents +
CArtAgO environment + Moise organization, in one JVM.

## Run it

```powershell
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-21.0.12.101-hotspot"
.\gradlew.bat run --console=plain
```

Then open the Mind Inspector at <http://127.0.0.1:3272> and watch `round`
climb on the Market artifact.

The run is 30 rounds and prints one line per round:

```
round 7 | price 10.5 | demand 3 | supply 3 | sold 3 | unmet 0 | seller firm_c | money 360
```

Columns, left to right: the volume-weighted price the town actually paid,
units households asked for, units firms put on the book, units that
changed hands, demand that went unserved, who captured the trade, and
money in circulation.

## What actually happens

Price starts at 14 and falls to **7.5**, then stops dead. 7.5 is
`firm_a`'s cost (7) plus its minimum margin. That is Bertrand competition
doing its thing: every firm's margin gets competed down onto its own cost,
the efficient firm takes the whole market, and the price pins there.

Demand rises as the price falls — 1, 1, 2, 2, 3 — because households differ
only in their reservation price and their cash. While households still have
money that spread behaves like a demand curve, produced by agents instead of
drawn on a graph.

Around round 14 you get rationing: `demand 4 | sold 1 | unmet 3`. Four
households want a unit, only firm_a is cheap enough for most of them, and
firm_a has one. The richest limits get served first.

**Then the money starts coming back, and demand stops dying.** Households pay
the firm that actually served them, and firms pay out again at the end of the
round:

```
households paid firms            429.83    <- one run; see the spread below
firms paid households            413.42    (labor 347 + overhead 23 + dividends 43.42)
retained by firms                 16.42   <- the only leakage, and it is real profit
```

Those totals are from a single run and they are **not** reproducible: across
14 runs household spending ranged from 351.25 to 429.83 and units sold from 38
to 46. What *is* reproducible, in all 14, is that household payments equal
`sum(price x units sold)` exactly, that units bought equal units sold exactly,
and that money stays at 360 to the last floating-point digit.

Demand never reaches zero. It settles at 4–6, because households recover most of
what they spend. Money stays at exactly 360 the whole way.

### Why the wage base is profit, not revenue

This is the mistake that makes a town look busy and mean nothing. Pay wages a
share of **revenue** and the firm never pays the cost of making the goods —
`unit_cost` gets used as a pricing floor and never debited, so the firm pockets
the whole revenue while households are slowly drained to finance it. Money is
conserved the entire time. No accounting check catches it. The town just
quietly becomes a machine for moving the households' endowments to the firms.

So a firm pays out its **labor cost** (`unit_cost × units sold`, which is real
production cost becoming household income), its **overhead** (fixed, every
round, sold or not), and a **dividend** of wage_share x profit. Production
cost comes first. Everything after that is surplus.

### Why firms go bankrupt — and why the round is not a finding

Overhead is what makes being *open* cost something, which is the only reason
exit can happen at all. But look at the constraint:

```
late-run margin per unit for firm_a = 7.5 - 7.0 = 0.5
```

Bertrand pins the efficient firm's margin at exactly 0.5. **Any overhead above
0.5 kills the most efficient firm as surely as the worst**, and the market
empties completely. There is no overhead a healthy firm can rest easy under.
That is not a flaw in the model; it is the entire reason firms have an
incentive to collude.

At `overhead(0.5)`, `firm_c` (cost 9) is always eventually undercut out. In one
run it had earned 8.00 of cumulative profit when it died — cumulative profit
is not liquidity. It paid dividends out of it as it went, had almost nothing in
reserve, and two rounds of rent finished it.

**But do not cite an exit round.** Across 14 runs:

| firm | runs it exited | exit rounds |
|---|---|---|
| `firm_c` | 14 / 14 | 4, or 21–28 — never in between |
| `firm_b` | 5 / 14 | 4, 4, 28, 29, 29 |
| `firm_a` | 1 / 14 | 4 |

Ten distinct exit patterns in 14 runs. `firm_c`'s exit is reproducible; its
*timing* is bimodal, and the cause is discrete rather than gradual — see
**Limitations** below.

Exit is triggered on **arrears** (wages and overhead owed and unpaid) that
*exceeded at the start of a round* crossing two rounds of overhead, tracked in
the Ledger, not on "can I afford this right now?". Testing affordability kills
a firm that merely sat out the opening round, because it owes overhead before it
has made its first sale. Bad debt with a grace period is the honest mechanism.
And arrears live in the Ledger because an AgentSpeak agent has no mutable
beliefs of its own.

**Read the arrears before paying, not after.** This cost a round of grace
period and looked fine while doing it. If you attempt the payment and *then*
read the arrears, the current round's own obligation is already inside the
number you are comparing against the threshold, so a firm that missed exactly
two rounds retires on the third. Nothing in the logs reveals this — the firm
just dies one round early, consistently.

### The trap I fell into, twice

Giving firms a working-capital float so they survive startup **created
21.60 of cash the town never issued** — `total_money` read 381.6 with nothing
in the log to explain it. Firms open at zero and bad debt carries the startup
period instead. Conservation is the invariant you protect last and check
always.

## The pieces

| File | What it owns |
|---|---|
| `src/env/town/Market.java` | Order book, matching, price discovery. Observable properties. Per-buyer bills, per-seller revenue, firm retirement. |
| `src/env/town/Ledger.java` | Accounts and transfers. The money supply. Household roster, payroll, bad debt. |
| `src/agt/firm.asl` | Bertrand bidding, production cost accounting, payroll, retirement. |
| `src/agt/household.asl` | Orders on `trading`, pays the matched seller on `settled`. Three separate moments. |
| `src/agt/timekeeper.asl` | Owns the round loop and prints the report. |
| `src/org/town-os.xml` | Moise roles, group, scheme, norms. |
| `town.jcm` | Every number worth changing. |

## Ordering is intent, payment is fact

A household runs three handlers, and the split between them is the whole
design:

| event | what it does |
|---|---|
| `+trading` | Reads price, limit and balance. Posts an order if it wants one. **Pays nothing.** |
| `+settled` | Asks the market `my_bill(Amount, Qty, Seller)`. Pays `Amount` to that `Seller` if `Qty > 0`. |
| `+payout` | Nothing. It exists so the firm-side signal has a handler. |

This is not a stylistic preference. The original code paid inside `+trading`,
before the order was even posted, and a full 30-round run proved it wrong:
households were charged for 34 units while the market reported 30 sold, and
the four surplus charges lined up exactly with the four unmet orders in
rounds 14 to 17. Money stayed at 360 the entire time, so the conservation
check everyone trusts stayed green and hid the bug completely.

An order is a willingness to pay at a ceiling. It is not an invoice.
Whether it becomes an obligation depends on whether it was matched, and only
the market knows that.

Note also that the bill is the **cleared** price, not the ceiling the buyer
posted. A household that pays the price it remembers rather than the price it
owes reproduces the original offset error even after the timing bug is fixed.

### Ordering is intent, payment is fact, settlement is atomic

A firm runs the same shape on the other side of the round, and there is a
fourth moment on top of it:

| event | what it does |
|---|---|
| `+trading` | Asks `am_retired`. Quotes if it is still trading, otherwise stays out. |
| `+payout` | Asks `my_revenue(Rev, Units)`. Pays labor + overhead + dividend. Checks arrears and retires if bankrupt. |

`+payout` is a **separate signal** from `+settled`, and the clock waits between
them. That wait is load-bearing: a firm cannot pay wages out of revenue it has
not received yet, and without the barrier a firm whose buyers were slow would
try to pay wages from an empty account and report a bankruptcy that never
happened.

So the round has three distinct phases — order, settle, pay — and the ordering
is a correctness requirement, not a style preference. `settled` means "buyers,
go pay"; `payout` means "firms, go pay". Both sides need a plan for both, or
Jason throws.

## Things that will waste your evening if nobody tells you

These cost me real time. They are all in the source comments too, but here
they are in one place.

**1. Prefix every belief read with `?`.** Once an agent focuses an artifact,
CArtAgO intercepts unprefixed terms in a plan body and tries to resolve them
as artifact *operations*. A bare `price(P)` is looked up as an operation
named `price`, fails, and kills the intention with an env failure. `?price(P)`
is a test goal, so it is checked against the belief base and never enters the
operation path. Operations you *want* to call stay bare: `post_order(P, 1)`.

**2. An event with no matching plan does not fail quietly.** Jason walks
into `findEventForFailure` and throws
`NullPointerException ... getPLForIM ... findEventForFailure`, which kills
that agent's entire reasoning cycle. The agent does not crash loudly — it
just goes permanently silent while the rest of the town carries on. Every
percept your agent can receive needs a plan, even `+settled <- true.` The
do-nothing handlers at the bottom of each agent file are load-bearing.

**3. Do not shadow a specific event handler with a catch-all.** Adding
`+focusing(_, _, _, _, _, _) <- true.` next to a real
`+focusing(LedId, led, ...) <- !enter(LedId).` makes the household stop
opening its account, and the market reports zero demand and zero money. A
catch-all is only for events you have no specific plan for.

**4. A failed test goal as the first step of a plan drops the intention, and
alternative plans are NOT tried.** This is why `!jcm::focus_env_art` gets away
with its `-!jcm::focus_env_art(L, Try)` retry: its first step is an *action*
that can fail, so Jason looks for another plan. If you write
`+!wait(N) <- ?focusing(...); ...` with a fallback plan, the agent dies
quietly on the first failed test and the fallback never runs. Polling for
readiness is a trap.

**5. A `+round(0)` event fires at startup.** The market is perceived once at
focus time with its initial values, so agents get a round-0 event before any
round exists. Acting on it logs a failure event for correct behaviour. The
market therefore announces itself with `signal("trading")` on open, and
agents trade off that.

**6. `.if_then_else` takes expressions, not goals.** `.if_then_else(B == "open", !do_thing, true)`
is a parse error. And a bare `.` is not a valid plan body — use `true`.

**7. `else if` does not exist in AgentSpeak.** A nested `if` has to live
inside the `else` block or the whole file fails to parse, silently taking
the agent out of the run.

**8. String observables, not booleans.** `defineObsProperty("book_open", "open")`
is testable from AgentSpeak as `?book_open("open")`. A Java `boolean` arrives
as an opaque object you cannot test.

**9. An observable property holds ONE value, shared by every reader.** So
you cannot publish a per-agent number. The natural way to bill buyers is an
observable named `owed` that each household reads and the market writes, and
it does not compile into anything that works: `ObsProperty.updateValue`
overwrites the single slot, so writing firm A's bill erases firm B's. Worse,
`AgentId` has no public `String` constructor, so `new AgentId(name)` is not
even available to key the map you were about to build. The fix is to expose
the bill as an **operation** instead, because an operation receives the
calling agent's identity as an implicit argument: `my_bill(Amount, Qty)`.
Answers that differ per agent, no shared state, no constructor to fake.

**10. A rule body ends with `.`, not `;`, and `;` fails as a behavioural
finding.** Writing `+trading <- !decide;` leaves the parser inside the rule
body, so it reports the error at the *next* rule's `!` rather than at the
`;`. The symptom is not a crash. Jason logs one parse error per agent, starts
the simulation anyway, and every household that failed to parse simply is
not there. The run completes 30 of 30 rounds with zero exceptions and prints:

```
round 30 | price 14 | demand 0 | supply 3 | sold 0 | unmet 0 | money 0
```

which reads exactly like a total economic collapse and is really six missing
agents. If a run reports implausible economics, read the `as2j: error
parsing` lines before believing any of it. Grep for `Encountered` first.

**11. A zero-argument operation is written BARE: `retire`, not `retire()`.**
This one cost a run. `retire()[artifact_id(MktId)]` fails to parse with
`Encountered ")" at line 200, column 22`, pointing straight at the character
you thought was fine. The parens are what is wrong.

**12. PowerShell 5.1 `Set-Content -Encoding UTF8` writes a BOM, and the
project parser rejects it.** A `town.jcm` edited that way dies with
`TokenMgrError: Lexical error at line 1, column 1. Encountered: "\ufeff"`.
The sim never starts. Write project files with:

```powershell
[System.IO.File]::WriteAllText($f, $txt, (New-Object System.Text.UTF8Encoding($false)))
```

**13. Seeding an account with a non-zero balance MINTS money.** `open_account(7.2, "firm")`
for three firms is `+21.6` that no household ever paid. Nothing throws, no
as2j error appears, and the corruption surfaces only as `total_money 381.6`
where 360 was expected - a discrepancy with no cause in the log. Accounts open
at 0 and the town's initial endowment stays the single source of cash.

## Deliberate simplifications

- **A firm's dividend is paid on profit, and profit is thin.** Late in a run
  `firm_a`'s margin is exactly 0.5 per unit, so the dividend is most of a
  rounding error. `wage_share` matters enormously in the first rounds, when
  margins are 5 or 6, and almost not at all at the end. That is a real property
  of Bertrand competition, not a modelling shortcut, but it does mean the
  dividend lever is most interesting early.
- **Overhead is the only fixed cost.** No rent, wages to non-household labour,
  or capital depreciation. Overhead stands in for all of it.
- **All production costs are paid to households.** The town has no other factor
  supplier, so labor cost and overhead both flow back to households as income.
  This keeps the money supply closed and the conservation invariant exact, at
  the cost of not modelling a capital-goods sector.
- **Bad debt is stranded, not written off.** A bankrupt firm's unpaid wages
  stay in its account. `total_money` still reads 360 and the households still
  did not receive that income - correct, but worth remembering when the
  household totals do not add up to households' spending.
- **The round has an 800ms settling window.** The model is time-dependent
  here, not fully deterministic. It is the one place where agent timing, not
  logic, decides the outcome. Replication shows it decides a lot: spending
  varies 351.25–429.83 and exit timing is bimodal. Replacing the sleep with a
  settlement barrier would remove that, and is deliberately not done — the
  variance is a finding, not a bug to paper over.
- **Moise is wired but not yet load-bearing.** Roles, group, scheme and norms
  are declared; no agent adopts the scheme, so the norms do not constrain
  anything yet. That is the org-behaviour slice.
- **Replicated: 14 runs. Exit timing is still not a finding.** The monetary
  identities are exact in all 14, but the exit round is bimodal, and the cause
  is a discrete tie-break rather than gradual drift. All three firms quote
  *identical* prices in the opening rounds (the undercutting rule is
  deterministic), so with supply above demand a tied round is won by iteration
  order, not merit. Win zero opening rounds and you owe overhead you can never
  clear — out at round 4. Win one and you survive into the late regime, where
  you are priced out on economics around round 21–28. One early sale is a
  threshold in its own right. `firm_a`, the *most efficient* firm, died in one
  run purely because it lost those ties.
- **There is no entry.** Firms start because `town.jcm` starts them. Nothing
  in the model decides that a firm should exist, only that an existing one
  should stop.
- **No collusion, obviously.** The one-line change that breaks Bertrand is
  commented in `firm.asl`. It is deliberately NOT made, because observing the
  efficient firm's margin pinned at 0.5 and therefore unable to afford *any*
  overhead is the argument for market power, and it is better felt than
  asserted.
