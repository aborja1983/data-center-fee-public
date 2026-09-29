# METHODOLOGY.md — Where Every Number Comes From

This is the document to hand someone who asks "how do you know that?" It explains, in plain
language, the source and method behind every figure the tool displays, and states plainly which
numbers are still not properly sourced.

Nothing here is a summary you have to take on trust. Every claim points at a file you can open,
a quote you can check, or a command you can run yourself.

---

## 1. The rule this project runs on

> Only use factual claims, figures, and data that come from files in `/sources`. Never use general
> knowledge, and never estimate a number that isn't backed by a file in `/sources`. If a number is
> needed but no source file exists for it yet, flag it clearly in the tool's own output as
> **"NEEDS SOURCE"** rather than guessing or filling it in.

That rule has caught real errors in this project — see §6.

---

## 2. Where the evidence lives

| What | Where | Generated? |
|---|---|---|
| Every displayed number, with source and exact quote | `files/CITATIONS.md` | **Yes** — from the tool's `CITATIONS` array |
| The documents themselves (PDFs, filings, data) | `sources/` | Downloaded, 18 files |
| Per-document notes: URL, retrieval date, verbatim quote, caveats | `sources/*/*.source.md` | Hand-written |
| Re-runnable computation scripts | `sources/*/*.py` | — |
| Region data status and confidence tiers | `files/REGIONS.md` | Hand-maintained |
| Expected calculation results | `files/TESTING.md` | **Yes** — from the tool |
| Automated checks that all of the above agree | `verify.js` | — |

In the tool itself, every number carries a bracketed citation marker — `[7]` — that links to the
matching row of the manifest, rendered on the page under "Where every number comes from".

---

## 3. The four methods

### 3.1 Household counts — EIA-861 × DOE LEAD

**The problem.** DOE's LEAD data gives households by *state*. Grid regions and utility territories
are *not* states. PJM covers "all or parts of" 13 states plus DC.

**What we do NOT do.** Earlier drafts bridged that gap by dividing a published population figure by
an assumed 2.54 persons per household, or by guessing what fraction of a state sits inside a region.
Both were unsourced. Neither is used any more.

**The method.** EIA Form 861 reports residential customer counts for every utility in the country,
tagged with the balancing authority it belongs to. That measures the geography exactly. It is *not*
a household count — vacant and seasonal homes carry meters — so it is used only to apportion:

```
households(region) = Σ   LEAD households(state) ×   EIA customers(region, state)
                   states                          ─────────────────────────────
                                                    EIA customers(all,    state)
```

EIA supplies the geography; LEAD supplies the households. Both are published, both are sourced.

**How we know it works.** Three independent checks:

| Check | Result |
|---|---|
| PNM vs PNM's own SEC filing ("more than 550,000 customers") | 548,346 — **0.3% apart** |
| NYISO vs LEAD New York directly (territory ≈ whole state) | 7,624,127 vs 7,633,936 — **0.1% apart** |
| ISO-NE vs LEAD six-state sum | 6,010,913 vs 6,039,540 — 0.5% apart |

The PNM check is the strongest: a published figure reproduced to within 0.3% by a derivation that
never used it.

**Script:** `sources/11-bill-benchmarks/region-households.py` · **Note:** `region-households.source.md`

### 3.2 Energy burden shares — DOE LEAD

**The question.** What share of households in a region spend more than 10% of income on energy
(severely burdened), or 6–10% (moderately burdened)?

**The source.** DOE's Low-Income Energy Affordability Data (LEAD) tool, 2022 update. Downloaded for
**all 51 US jurisdictions** from OpenEI, DOE/NREL's public data repository
(`data.openei.org/files/6219/`) — the official distribution point, not a third-party copy.

#### What LEAD actually is, in plain terms

This matters for how much weight the burden figures can bear, so it is worth spelling out rather
than hiding behind "built on Census data".

**It is survey-derived.** LEAD is built on the **American Community Survey**, the Census Bureau's
ongoing survey of roughly 3.5 million households a year, which asks each one its income and what it
pays for electricity, gas and other fuel. Those energy costs are **what people report**, not what
utilities billed. Nobody checks the answers against a bill, so they carry ordinary survey error.

**It uses microdata, and it has to.** "Microdata" means individual anonymised household records
rather than pre-summed tables. The burden calculation needs each household's income *and* its energy
cost **together** to form a ratio — a published table of average income and average energy cost
could never tell you how many households sit above 10%.

**It is a weighted sample, not a count.** Those 3.5 million responses are scaled up to represent
~131 million households. That is why LEAD's household counts are fractional: a row reading
`UNITS = 0.1274` is a statistical weight, not a tenth of a household. It is also why cells with
degenerate income are excluded — some are weighting artefacts rather than real households.

**"Calibrated to EIA utility data" is doing real work.** NREL adjusts the survey-reported energy
spending so the totals match what utilities actually reported collecting to EIA. Self-reported
spending drifts; utility revenue is measured. The consequence:

> **The totals are anchored to real billing data. The distribution across households is survey-derived.**
> Aggregate energy spending in a state is solid. *Which* households are burdened is a good
> statistical estimate, not a measurement.

**A vintage mismatch worth knowing.** EIA data appears twice in this project's chain, doing two
different jobs: NREL uses EIA-861 (**2022**) to calibrate LEAD internally, and this project uses
EIA-861 (**2024**) to apportion states onto regions (§3.1). Not circular — but the two are two years
apart.

**How to describe this to a skeptic.** Not "measured", but: *survey-reported household energy
spending, calibrated to actual utility revenue, 2022 vintage.* That is about as good as public US
household energy data gets, and it is still an estimate.

**The method.** LEAD does not publish "share above a threshold" — it publishes cells: census tract ×
income band × tenure × building type × heating fuel, each with a household count, summed income and
summed energy cost. We compute each cell's burden as (electricity + gas + other fuel) ÷ income and
total the households by tier. New Mexico alone has 163,014 such cells, about 5 households each.

**The thresholds** — 6% high, 10% severe — come from ACEEE, quoted verbatim:
> "Energy burdens above 6% are traditionally considered high, while burdens above 10% are considered severe."

**Four limitations, all pushing the same way — downward:**
1. Every household inherits its cell's average, which flattens both extremes
2. Households reporting no income are excluded (3.93% in New Mexico) and are plausibly among the worst off
3. Shares are computed per state, then apportioned by the §3.1 weights
4. Data is 2022

So the true severe share is more likely *above* our figures than below.

**Which households the share is applied to — a choice, not arithmetic (documented 2026-09-09).**
LEAD cannot compute a burden without usable reported income, so it drops households that report none:
33,780 of New Mexico's 812,852 (4.2%), 84,112 of Virginia's 3,289,829 (2.6%). The share is therefore
computed on the *usable* base — NM severe = 101,213 / 779,072 = 12.9915% — while the tool multiplies it
by the *total* household count, giving 105,589. Verified across all 50 states with histogram data: the
share is identical to four decimal places either way, so the divergence is entirely in the denominator.

There are three ways to treat the excluded households, and the tool takes the middle one:

| | severe households, NM | what it assumes about the excluded |
|---|---|---|
| LEAD's own count | 101,213 | never burdened |
| **share x all households** | **105,589** | **burdened at the same rate** |
| what §3.2's own limitation argues | above 105,589 | burdened *more* than average |

The third row is what this note has said all along — households reporting no income "are plausibly among
the worst off". So the tool's choice is conservative against the project's own stated belief, while still
counting the excluded households rather than discarding them.

This had never been written down. `region-households.py` apportions the real severe count (`sev`) at the
same time as the total and then **discards it**, writing only `sev/use*100` beside `hh = total`; the tool
multiplies the two fields the CSV happens to expose. The pairing was an artifact of that schema before it
was a decision. It is now stated at the multiplication site in the calc core, in the `burden-method`
citation a reader clicks, and in the bill panel where both numbers appear together.

The analogous geographic assumption — applying a statewide rate to a territory that is not the whole state
— was already documented in `lead-burden-shares.source.md` §3. This is the same kind of choice on a
different axis, and it should have been recorded at the same time.

**Script:** `sources/03-household-burden/lead-burden-shares.py` · **Notes:** `lead-burden-shares.source.md`, `aceee-city-energy-burdens-2024.source.md`, `doe-lead-tool.source.md`

### 3.3 Data center buildout

This is the weakest link in the whole tool, and it varies enormously by region.

**Best case — the grid operator publishes its own screened forecast.** Only PJM and NYISO do.
NYISO states a queue of 12,670 MW and its own expectation that "as much as roughly 2,880 MW of that
new demand will be on the system by 2040" — an implied completion rate of 22.7%.

**Middle case — a queue exists, but nobody has screened it.** ERCOT's filing projects 367,790 MW of
total demand by 2032 against a 108,023 MW base, a 259.8 GW large-load increment — but ERCOT itself
writes that it "has concerns with using the preliminary load forecast values." Converting a queue to
a forecast requires a completion rate, and the only published rates (LBNL, 13–19%) are measured on
*generation* projects, not data centers. Applying them is an analogy, not a measurement.

**Worst case — nothing at all.** MISO's own large load page says only "thousands of megawatts."

Regions in the last category display buildout figures labeled **"PLACEHOLDER — NOT YET SOURCED"**
with distinct styling, per `files/REGIONS.md`.

### 3.4 Tax rates and payout structure

These are **not** empirical findings and are not presented as such. The three tiers (1.3¢/0.65¢,
1.9¢/0.95¢, 3.0¢/1.5¢ per kWh), the 2:1 fossil-to-clean ratio and the 2:1 severe-to-moderate payout
ratio are **design decisions of this project**, recorded as such in the manifest.

The one external anchor is Virginia's enacted HB 30 rate of **$0.011/kWh**, quoted verbatim from a
tax-advisory analysis of the signed budget. Note two features of the real statute this tool does
*not* model: it is capped at $600 million a year with pro rata refunds above the cap, and it taxes
self-supplied electricity.

### 3.4a Does it close the gap? — burden, not dollars

Dollars say what a household receives. **Energy burden says whether it worked.** Burden is annual
home energy cost as a share of income, so a rebate lowers it directly.

| Region | Severe tier burden | After its own best-case rebate | Unburdened households pay | Gap closed |
|---|---|---|---|---|
| PJM | 18.26% | 11.44% | 1.81% | **56%** to threshold / 41% to unburdened mean |
| ERCOT | 18.55% | 0% | 1.82% | overshoots |
| NYISO | 18.00% | 16.54% | 1.81% | **9%** |
| PNM | 19.43% | 8.77% | 1.78% | 60% |
| APS | 19.44% | 14.49% | 1.72% | 28% |

#### Which gap? Two benchmarks, both reported

"Closing the gap" needs a target, and there are two defensible ones. The tool reports both, and
leads with the easier-to-defend one rather than the flattering one:

| Benchmark | What it is | PJM: distance closed | Rebate to reach it |
|---|---|---|---|
| **6% threshold** *(primary)* | ACEEE's own definition of where high burden begins — a target a policy could adopt | **56%** | $164.04/mo |
| Unburdened mean | What households under 6% in that region actually pay on average (1.8%) | 41% | $220.12/mo |

The unburdened mean is the harder target, because it is dragged down by high-income households the
policy is not aimed at — a family on $300,000 with a $200 bill has a burden near 0.8%. Reporting
only that would understate the policy; reporting only the threshold would flatter it. Both appear.

#### These are regional figures, not national ones

Every burden percentage, household count and dollar amount is computed **per region**, from state
LEAD data combined with EIA-861 customer weights. Burden varies enormously — **Utah 4.69% severely
burdened, Maine 18.60%** — so a national average would be meaningless for any actual region.

What is *not* regional, and is labeled as such in the tool:

- the **6% and 10% thresholds**, which are national definitions from ACEEE
- the **LIHEAP comparison** ($41.67/mo, ~20% reach), which is a national average

Two things stand out. **The gap is remarkably uniform** — severely burdened households pay 18–19% of
income on energy everywhere, unburdened households 1.7–1.8%, a 16–18 point gap in every region. And
**what closes it costs $119–$200 a month**, again similar everywhere. What differs wildly is what
each region's buildout actually delivers.

That reframes the burden-indexed tax idea. Indexing the rate to a region's burden barely helps,
because burden hardly varies — NYISO's burden weight is 2.5% above PJM's, so a burden-indexed rate
would lift its rebate from $24.40 to about $25, against the $200 needed. **The binding constraint is
buildout intensity, not burden.**

**Threshold crossings are harsher than the averages suggest.** At PJM the rebate moves 1,274,423
households out of *severe* burden (below 10%) but **zero** below the 6% threshold. It relieves the
gap without eliminating it for anyone.

#### The gas-heating limitation, quantified

Burden counts **all** home energy — electricity, gas and other fuel — per the DOE/ACEEE definition.
But the rebate is an **electricity** credit. Where it exceeds a household's electricity bill the
surplus arrives as cash, which does not reduce a gas bill.

The tool subtracts the full rebate from total energy cost anyway, so burden relief is **overstated
wherever heating is not electric** — and that varies enormously by region:

| Region | Mean electricity | Mean total home energy | Electricity share |
|---|---|---|---|
| ERCOT | $152.34 | $188.52 | **81%** |
| Arizona Public Service | $127.92 | $172.34 | 74% |
| PJM | $123.75 | $244.31 | 51% |
| PNM | $91.46 | $195.15 | 47% |
| **NYISO** | $130.78 | $300.47 | **44%** |

In NYISO the rebate could zero out a burdened household's electricity bill entirely and still leave
**56% of their energy spending untouched**. In ERCOT it reaches four-fifths of it. The overstatement
is therefore small in Texas and Arizona and substantial in New York and the mid-Atlantic.

This is shown as a **prominent warning at the top of the burden panel**, not in small print, and it
names the region's own figure.

Both sides of that ratio are means, computed the same way: mean electricity from the bill histogram
using bucket midpoints, mean total energy as summed energy ÷ households ÷ 12. (An earlier draft
compared a *median* electricity bill with a *mean* total, which understated the electricity share by
about 7 points in every region.)

**Fixing it properly** would mean splitting the rebate against electricity cost only and treating any
remainder as income rather than as an energy-cost offset — which changes the burden denominator, not
just the numerator.

**Scripts:** `sources/03-household-burden/lead-burden-income.py`,
`sources/11-bill-benchmarks/region-burden-gap.py`. Severe-tier household counts reproduce the
independent earlier harvest exactly in all 19 states.

### 3.4b Two programs, one gap — and the carve-out control

**The stated design is that the money follows the buildout.** Revenue raised in a region is rebated
to households in that region, compensating the communities that host data centers. LIHEAP, the
existing federal program, is intended to cover regions without them.

**The arithmetic shows LIHEAP cannot currently do that job.** From its own reporting: it serves
**about 6 million households of roughly 30 million eligible — one in five** — because it must be
applied for and runs first-come, first-served. Its $500 average benefit covers about **25%** of a
low-income household's ~$2,000 annual energy cost, funding has been flat at ~$4.05bn for three
years, and the FY2026 budget proposed eliminating it.

Extending today's benefit to every eligible household would cost **$15.0bn — about $10.95bn a year
more than is currently appropriated**, roughly 3.7× current funding.

For scale: the five regions currently in the tool, each at its own best-available buildout figure
and the moderate tax tier, would raise about **$12.6bn a year between them** — more than the gap,
from five of the twenty-four regions now in the tool.

**The tension.** The same dollar cannot both stay local (compensating hosts) and fund a national
floor. The tool therefore offers a **carve-out control**: a share of revenue diverted to the floor
before anything is distributed locally. It **defaults to 0%**, preserving the stated design, and the
tool **does not recommend a rate** — any non-zero value is a policy choice, and the trade-off is
shown explicitly as dollars forgone per local household.

NYISO is where this matters most. Its severely burdened share (10.87%) is *higher* than ERCOT's
(10.02%), and it has nearly as many severely burdened households — but 17× less data center
intensity per household, so the rebate is 19× smaller and falls *below* LIHEAP. That is the design
working as intended, and it is also the case the carve-out exists to address.

### 3.5a The two front ends, and the door between them

The tool has a **public view** (`?public=1`) over the same calculation core: **four decisions**
instead of eleven controls — where, how much gets built, how much the fee is, and who receives it —
with the dividend held back as a reveal rather than updating live. The buildout was the fourth, added
2026-09-28: it had been hidden on the reasoning that the pipeline is a forecast rather than a choice,
which is true and beside the point. *"What if half of this never gets built"* is the first question
most readers ask, and in New Mexico the answer is immediate — 10,710 MW becomes 5,400 and $705.54
becomes $355.74. State is
shared deliberately, so a scenario configured in the full tool carries over — and any assumption
sitting off its default is **named** in the reveal, with a reset, because the hazard that sharing
creates is a reader mistaking a configured scenario for the baseline.

**The door was missing until 2026-09-28.** The stylesheet had carried a `#pubSwitch` rule since the
public view was built and no element had ever used it: there was a way in, via the query string, and
no way back out. A lay reader who wanted to see what was behind the number had to know to edit the
URL. Switching is now a control in the header and a line under the reveal, and it is a **class
change on one page**, not a navigation — so state survives intact in both directions and nothing is
recomputed from defaults.

**The same sweep found three CSS rules matching nothing at all**, one of them a live defect:

- `body.public .subh-inflation` — intended to keep the inflation projection out of the public view.
  No element has ever carried that class, so a lay reader had always been offered a 30-year
  projection and an inflation rate among what the code comment calls *"four decisions instead of
  eleven controls"*. The block now has an id and is actually hidden.
- `body.public #pickFlat` and `body.public #instrumentNote` — for elements removed in earlier
  redesigns. Dead rather than wrong, and removed rather than kept "just in case".

**A CSS selector that matches nothing throws no error, logs nothing, and renders identically
forever.** Only reading the rules against the markup finds it. `render-check.js` now sweeps every
`body.public` selector and fails if any `#id` or `.class` in it names something no element has —
which also catches the opposite drift, an element renamed while the rule keeps the old name. This is
a **source** check rather than a render one, because the public view is appended after
`RENDER-CORE-END` precisely so that no verifier evaluates it, which had left its CSS unguarded
entirely.

### 3.5b The headline in context — the distribution, LIHEAP, and the transfer already running

Three things were added to the results card on 2026-09-28, all of them defensive rather than
computational. None enters the arithmetic.

**The distribution, because the default is an extreme.** At the moderate tier the headline runs from
**$0.00 to $3,039.50 a month** across the 51 jurisdictions. The middle of that range is **$76.64**
(South Dakota is closest to it). New Mexico, the tool's default, is **4th highest** and pays more than
three times a severely burdened household's entire energy bill. That figure is correct and
indefensible standing alone: a reader does not need to find an error, only to say *"that is more than
they spend on energy in total"*, and the sourcing never gets a hearing. So the card now states the
rank, the middle, the range, and the fact that **5 jurisdictions have no tracked pipeline and raise
nothing**, computed live from `headlineDistribution()`.

Every other jurisdiction in that comparison is run on **its own** pipeline and grid mix but the
reader's other settings, so the question answered is "where does what I am looking at sit" rather
than a comparison against someone else's defaults.

The two tails are extreme for opposite reasons and get opposite sentences. The first version had one
flag and told Massachusetts, at $0.00, that it had *"an unusually large pipeline against its burdened
population"*. `render-check.js` now fails if a jurisdiction outside the top five is given the
top-of-range explanation, if one paying $0.00 does not say the fee raises nothing, or if a mid-range
jurisdiction is called an extreme — none of which any check on the numbers would have caught.

**LIHEAP, because it is the only benchmark a reader already knows.** Its average benefit is
**$55.17/mo** and it reaches about **17%** of eligible households. The middle of this tool's range is
**1.4×** that figure — a policy-scale claim that survives being checked, where the headline alone
invites disbelief.

**The transfer already running the other way — written, then cut the same day.** A passage was added
to the results card arguing that households already fund data centers' grid costs, and removed within
the hour when the project owner asked what point it was making. The answer was: none that survives
contact, and the reasoning is worth keeping because the failure is instructive.

- **The scale was wrong by more than an order of magnitude.** Indiana's fee at the moderate tier
  raises **$1.22 billion a year**. The figure the passage leaned on was **$73.8 million** — one
  project, one-time capital, and contested. The fee raises **16.6× that, annually**. "This reverses a
  transfer already happening" is arithmetic a critic completes in thirty seconds.
- **It argued for a different policy than this one.** If the defect is that network upgrades get
  socialized, the remedy is to direct-assign them: a one-line tariff change needing no excise and no
  dividend. The passage raised that objection and did not answer it.

What the documents genuinely support is narrower — a point about **venue**. Missouri's commission
assigned costs to the causer, and the one category it could not reach was defined by a federal RTO
tariff, outside the proceeding where ratepayer advocates appear. That belongs in prose where its
scope can be stated, not under a dividend figure it does not defend.

**The citations stayed.** Both documents are good and remain in the manifest and the Sources card as
context. That required a new concept, because the tool's dead-weight guard requires every manifest row
to be cited somewhere in the UI: a `SOURCE_GROUPS` entry may now carry `contextOnly:true`, and rows in
such a group are exempt. The exemption is constrained three ways, because an exemption is exactly the
kind of guard that becomes a loophole:

1. A row can only be exempt by sitting in a declared context-only group, so it cannot be earned by
   deleting the sentence that cited it and saying nothing.
2. `render-check.js` proves each exempt row actually reaches the rendered Sources card **with its
   locator** — the part `verify.js` cannot see, because it never renders. The first version of that
   guard read an empty string and reported the card as empty: `buildCitTable()` runs at bootstrap,
   not inside `render()`.
3. The count is **pinned at 2**. A third context-only row fails the suite until someone raises the
   expected number deliberately.

- **Missouri**, approved and verified twice: a large-load customer *"shall pay all costs of
  distribution or transmission facilities, other than any transmission facilities classified as
  network upgrades by the Midcontinent Independent System Operator's Open Access Transmission
  Tariff"*.
- **Indiana**, contested and labeled as such: an intervenor in IURC Cause 46394 contends AES Indiana
  would recover **$73.8 million** of network upgrades through base rates. The source note forbids
  citing this as a Commission finding — the docket was pending as of 2026-09-15.

`sources/18-large-load-tariffs/` remains a **separate tool** and this one still cites no figure from
it. What was taken is the underlying primary documents, which live in `sources/04-tax-legal/`. The
survey's own `cost_allocation_language` column was considered as a source of a count and rejected: it
holds prose, not categories, and a keyword tally over it would be a search presented as a
classification — the first record matching "network upgrades" has the value `NOT ADDRESSED`.

### 3.6 What it costs the industry — and why elasticity is not estimated

**What large customers pay today**, computed from EIA-861 industrial revenue ÷ sales per balancing
authority:

| Region | ¢/kWh today | Moderate tax adds | Increase |
|---|---|---|---|
| **ERCOT** | **6.15** | 1.615¢ | **+26.2%** |
| PNM | 6.42 | 1.615¢ | +25.1% |
| PJM | 8.11 | 1.615¢ | +19.9% |
| NYISO | 9.17 | 1.615¢ | +17.6% |
| Arizona Public Service | 10.21 | 1.615¢ | +15.8% |
| CAISO | 23.33 | 1.615¢ | **+6.9%** |

> **These figures were corrected on 2026-09-03 and roughly doubled in retail-choice regions.**
> EIA-861 splits a retail-choice premise into separate "Delivery" (wires) and "Energy" (supply)
> rows that carry the *same* sales volume. This project counted only the wires revenue against the
> full volume, which understated every choice-state price — PJM read 4.11¢ before the fix and 8.11¢
> after; Pennsylvania alone read 1.46¢ against a true 7.87¢. The filter came from
> `region-households.py`, where excluding "Energy" rows is *correct* (it stops one premise being
> counted as two customers) and was wrongly inherited by the price scripts, where the job is
> different. Validated against EIA's own published state averages: the corrected method gives
> California 31.97¢ residential against EIA's published ~31–32¢. See §6.

The percentage varies nearly four-fold because prices vary while the tax does not vary nearly as
much. It bites hardest exactly where power is cheapest — which is where data centers are being built.

**It barely reorders the regions — but "barely" is not "not at all", and the difference is the
design working as intended.** After a moderate tax the price ranking is unchanged in 12 of 14
positions. The exception: **MISO and Duke Energy Carolinas swap.** MISO is marginally cheaper
untaxed (7.41¢ vs 7.54¢) but sits on a dirtier grid, so the *differentiated* rate charges it more
and it ends up the pricier of the two. That is the mechanism doing its job, not a defect — a
uniform flat rate (§3.10a) adds identical cents everywhere and therefore cannot reorder anything.

> **Corrected 2026-09-03.** This section previously claimed the ranking was "identical, position for
> position", and that the cheapest region taxed stayed below every other region *untaxed*. Both were
> true under the understated pre-correction prices, where the tax loomed larger against a compressed
> price range. Neither survives the corrected prices: ERCOT taxed (7.69¢) now sits above seven other
> regions' untaxed prices, the cheapest of which is PNM at 6.42¢. The tool computes this claim live
> and names the swapping pair on screen, so it cannot silently go stale again.

ERCOT taxed is 7.69¢, still 3.0× cheaper than CAISO untaxed. The competitiveness objection is
weaker than the percentage implies — but it is not nil, and the tool no longer overstates the case.

*Caveat: these are average industrial rates. Large facilities negotiate bespoke tariffs and PPAs.*

#### Elasticity: what is assumed, and what is not

An elasticity is one number: how much less gets built for each 1% the price of power rises. At 1, a
1% rise costs 1% of the buildout. At 2, twice that. At 0, nobody reconsiders a project because of the
fee.

**The model assumes 0.** No behavioral response. That is stated in section 4 of both tools as the one
assumption the reader is given no control for.

**The usable form is megawatts per penny.** An elasticity is a ratio of percentages, which is hard to
hold in mind. The same number in the units the tool actually sets:

    MW lost per penny on the fee = pipeline × ε ÷ local industrial price

New Mexico at the moderate tier: a 5.43¢ industrial price, a 1.90¢ fossil rate and a 0.95¢ clean
rate, which the facility pays as a blended **1.4728¢** at the state's measured grid mix. That is a
**27.1%** rise on the price of power, against a tracked pipeline of **10,710 MW**.

| Elasticity | One more penny on the fee costs | The 1.90¢ in place therefore costs | Leaving |
|---|---|---|---|
| 0.00 | 0 MW | 0 MW | 10,710 MW |
| 0.37 | 564 MW | 1,071 MW | 9,639 MW |
| 0.92 | 1,409 MW | 2,677 MW | 8,033 MW |
| 1.84 | 2,818 MW | 5,355 MW | 5,355 MW |
| 2.77 | 4,228 MW | 8,033 MW | 2,678 MW |
| 3.69 | 5,637 MW | 10,710 MW | 0 MW |

Measured on the model by moving the fossil rate from 0¢ to 1¢, not derived from the formula, so the
table and the calculation core cannot disagree.

**One multiplication checks the whole table.** MW per penny × the fee in place = MW lost. At the
peak elasticity that is 2,818.3 × 1.90 = 5,355, half of 10,710. The figures in the table are rounded
for display, so multiplying the rounded ones gives 5,354; the model carries the unrounded values.

**Where 1.84 and 3.69 come from.** Both are properties of the fee, expressed in elasticity units.
Write `s` for the price rise the fee causes, here 27.1%.

- Buildout reaches zero when `ε·s = 1`, so `ε = 1/s = 3.69`.
- The fee is exactly as high as revenue allows when `ε = 1/(2s) = 1.84`. Past that, a larger fee
  collects less.

The second is always half the first.

**Why the revenue peak is always at half the buildout.** Write `p` for the local price, `t` for the
fee, `B` for the buildout with no fee, `k` for annual kilowatt-hours per unit of buildout. The
response is linear, so surviving buildout is `Q(t) = B(1 − ε·t/p)` and revenue is

    R(t) = k·t·Q(t) = k·B·(t − ε·t²/p)

Differentiating, `dR/dt = k·B·(1 − 2·ε·t/p)`, which is zero at `t* = p/(2ε)`. Substituting back, the
share of buildout lost at `t*` is `ε·t*/p = ½`. The `ε` cancels, so the result holds for every price,
every elasticity and every pipeline. Buildout reaches zero at `t = p/ε`, exactly twice `t*`.

Checked across all 153 state × tier combinations. In elasticity units the two landmarks range from
1.0168 to 14.5766 and from 2.0336 to 29.1531. Measured as share of buildout lost, both are constant:
0.5000 and 1.0000 in every case.

**The same result without algebra.** Raising the fee does two things that work against each other. It
collects more on every kilowatt-hour still being charged, and it leaves fewer kilowatt-hours to
charge. When the fee is small the load being driven away was paying almost nothing, while the higher
rate applies to everything that stays, so revenue rises. As the fee grows, the load being driven away
is load that was paying the full fee. The turning point is where one more percent on the fee costs
one percent of the remaining buildout. Under a straight-line response that point is always where half
the pipeline has gone.

Checked numerically as well as algebraically. Sweeping the fee at a fixed elasticity of 1.84, revenue
peaks at the moderate tier's own 1.47¢ and falls away in both directions, with half the pipeline
standing at the peak. Halving the fee and raising it by a quarter both collect less.

**Half is a property of the straight line, not a finding about data centers.** The linear form
`1 − ε·(t/p)` is a modeling choice. It is what creates an endpoint at all: under a constant-elasticity
form demand never reaches exactly zero. If builders ignored small fees and then left at once, or left
steadily from the first cent, the peak would sit somewhere other than half and the wipeout would not
be twice it.

**Two different elasticities are in play.** The first is buildout against the price of power, which
the model holds constant by assumption. The second is buildout against the fee itself, which is not
constant: it starts at zero, because a fee of nothing drives nobody away, and grows as the fee grows.
Revenue peaks when the second passes 1. Only the first is a number anyone is asked to supply.

**There is no control for it, and the reason is arithmetic.** Setting the elasticity to `ε` produces
the same revenue and the same dividend as leaving it at 0 and setting the buildout to
`B × (1 − ε·s)`. Checked rather than reasoned: 2,700 comparisons across both tools, every
jurisdiction × tier × four elasticities, largest relative difference exactly zero. A reader who
believes the fee will drive off a third of the buildout can say so with the buildout slider.
`render-check.js` fails if the control returns to the page.

**What the data shows.** Both tools compute the correlation between each jurisdiction's industrial
electricity price and the data center capacity tracked there, rather than quoting one. It is reported
with two specifications because they disagree:

| | Raw capacity, all jurisdictions | Logarithms, per household, those with a pipeline |
|---|---|---|
| By state | −0.21 (n = 51) | −0.49 (n = 46) |
| By region | −0.23 (n = 24) | −0.36 (n = 24) |

Neither is an elasticity. There are no controls for land, water, fiber, permitting speed,
interconnection queues or state incentives. The individual points make the same case: ERCOT has lower
industrial prices than PJM, 6.15¢ against 8.11¢, and 7.8× the buildout intensity, 10.76 MW per
thousand households against 1.38. PNM sits at 6.42¢, almost the same price as ERCOT, with 0.015 MW
per thousand households.

**One published elasticity exists in this project's source corpus, and it cannot be used.** LBNL
cites Papp et al. (2023) at 0.33–0.40 long-run and 0.69–0.71 short-run. That measures Bitcoin mining
energy against the Bitcoin price, not data center demand against an electricity fee. The source note
at `sources/02-load-forecasts/lbnl-load-factor.source.md` records it as a trap and states it must not
be borrowed for this dial.

LBNL's own position is that the relationship between the cost of cloud and AI services and their
power demand is "not well understood", and it names quantifying that an open research need. The
default of zero matches the state of the evidence rather than working around a gap.

**The best available future evidence is a live experiment.** Virginia began charging 1.1¢/kWh in July
2026 in the largest data center market in the world. Tracking its interconnection queue would produce
a better estimate than any model. AEP Ohio's large-load tariff is a second.


**What the tool can say without any assumption.** Two break-evens follow from the algebra alone.
With revenue `R = B(1 − εt/p)·k·t`, revenue peaks at `ε = p/2t` and buildout reaches zero at
`ε = p/t`. For PJM at the moderate tier:

| | |
|---|---|
| Above this elasticity, raising the tax **lowers** revenue | **1.27** |
| At this elasticity, buildout stops entirely | **2.54** |

Those scale with how large the tax is relative to the local price, so they differ by region — and
they tell you where the unknown starts to matter without claiming to know it.

**The live experiment.** Virginia began charging 1.1¢/kWh in July 2026, in the highest-concentration
data center market in the world. Tracking its interconnection queue over 2026–2028 would produce a
better elasticity estimate than any model. AEP Ohio's large-load tariff is a second natural
experiment already on the source-gathering checklist.

**Scripts:** `sources/11-bill-benchmarks/region-industrial-price.py`, `region-residential-price.py` · **Note:** `region-prices.source.md` — read this before touching the service-type filter; it documents why revenue and sales deliberately use *different* EIA-861 row sets, and what breaks if they are collapsed back into one

### 3.7 What counts as "fossil-matched" — the design, and what it needs

The tax charges double on fossil-matched electricity, so the definition of "fossil-matched" is
load-bearing. **The chosen design has three parts.**

**1. Default: the grid average where the facility sits.** Measured from EPA eGRID 2023 at balancing
authority level — the same level the tool's regions are already defined at.

| Region | Fossil share | vs the old flat 70% |
|---|---|---|
| Arizona Public Service | **83.5%** | +13.5 |
| MISO | 68.4% | −1.6 |
| ERCOT | 62.5% | −7.5 |
| PJM | 60.0% | −10.0 |
| ISO-NE | 58.3% | −11.7 |
| SPP | 53.5% | −16.5 |
| NYISO | 50.7% | −19.3 |
| CAISO | 42.1% | −27.9 |
| PNM | **40.4%** | −29.6 |

The flat 70% was wrong in **both directions**, and the correction moves revenue by −17% (PNM) to
+8% (APS).

*Pros:* the data exists today, it is free and official, it cannot be gamed by the taxpayer, and it
is administrable as one published number per balancing authority per year — which also makes it
robust to the uniformity-clause questions on the source checklist.
*Cons:* it rewards **siting** rather than **behavior**. A facility that funds new solar pays the
same as one next door that funds nothing.

**2. Clean credit: hourly matching only.** A facility reduces its rate only by demonstrating
**hourly-matched** clean supply — not annual REC matching. This is the part that restores the
behavioral incentive the grid-average default removes, and the hourly requirement closes the
additionality loophole whereby a facility claims 100% renewable annually while drawing gas at 2am.

*Cons:* **the share of load that would qualify is not known.** It is a policy dial, not an estimate.

**Now sourced (2026-09-07).** The documents were gathered into `sources/12-hourly-matching/` after
the earlier unsourced version of this passage was challenged. They separate two questions the old
wording had merged, and they point in **different directions** — which is why the original sentence
was wrong and a straight reversal would have been wrong too.

**Hourly matching as a measurement practice is established, and predates the certificates.** Google
published a full hourly methodology in February 2021 and reports per-facility hourly scores. For its
Lenoir, North Carolina data center in 2019 the *Annual CFE Score* was **65.5%**, with the best hour
at **91%** (16 April, noon) and the worst at **40%** (26 September, 7pm). That spread is also the
clearest published illustration of what this rate design is aimed at: an annual REC purchase would
have scored the same facility 100%.

**Microsoft's hourly matching is a 2030 target, not a current achievement.** Its 2021 commitment —
"100 percent of our electricity consumption, 100 percent of the time, matched by zero carbon energy
purchases" — is restated in the future tense in the 2025 report, which reports **34 GW of contracted
carbon-free electricity** rather than any hourly figure. The word "hourly" does not appear in that
report. Both companies are real evidence, but they are not the same kind of evidence and should not
be cited as one.

**Certificate accreditation is genuinely recent.** EnergyTag published its Matching Standard in March
2024 and Scheme Standard V2 in December 2024, and accredited its **first** issuers on **25 June
2025** — Energinet (the Danish TSO) and Flexidao, the latter covering 47 countries. So the practice
is a decade old at hyperscaler scale while the *auditable certificate regime* is about a year old.

**What is still not known, and remains a policy dial:** what share of US data center load could
obtain qualifying certificates today. The first accreditations are European and platform-based, and
no source found publishes a US market-wide figure. PJM-GATS and M-RETS remain ungathered.

**What survives unchanged** is the part the design actually rests on: none of this makes hourly
reporting *mandatory*. Google measures because it chose to and Microsoft commits because it chose to;
a tax authority cannot audit against a voluntary disclosure. That is why the reporting requirements
below are stated as a precondition of the design rather than as an observation about current
practice.

**3. Behind-the-meter gas is taxed at the fossil rate regardless.** Under a pure grid-average rule an
on-site gas plant would escape the tax entirely — the single largest loophole available. Virginia's
HB 30 taxes self-supplied electricity for exactly this reason.

*Cons:* **the tool applies each region's grid-average fossil share to all of its load, including any
served behind the meter.** A facility running on its own gas turbines has a ~100% fossil supply, not
its region's grid average, so wherever BTM gas is material the tool *understates* the fossil share and
therefore understates revenue under the differentiated rate. The slider lets a user model this by
hand; no data drives it and no default reflects it.

*Sourced 2026-09-07, updated to the August 2026 release 2026-09-08 (`sources/13-behind-the-meter/`).*
The scale is now documented even though the per-region share is not. Global Energy Monitor's Global Oil
and Gas Plant Tracker "now counts 189 gigawatts (GW) of gas-fired capacity in the U.S. across the
announced, pre-construction, and construction phases" tied to data centers, against a total US pipeline
that grew "from 252 to 378 GW" in six months. The report's own chart data gives both exactly —
189,344.15 MW of 378,265.55 MW — so **half the US gas pipeline, 50.1%, is now attributed to data
centers**, up from "more than one-third" in January. Texas holds "122 GW of gas-fired power capacity in
development", of which "nearly two-thirds ... (77 GW), is planned to directly power data centers".
Cleanview's mid-2026 summary puts announced BTM capacity at roughly 90 GW with about 2 GW operating
across four projects, and Ohio leading capacity under construction at 736 MW.

**GEM changed the definition between releases, and the newer number is the looser one.** January 2026
counted only capacity "produced at the facility to support data center operations independently of the
public grid" — strictly behind the meter. August 2026 counts a plant "built primarily to serve data
centers as its main load, behind-the-meter **or dedicated**". The 189 GW is therefore a *superset* of
BTM: a dedicated on-grid plant sells through the meter and already sits inside the grid-average fossil
share this tool applies. GEM publishes no split between the two halves, so the BTM portion is unknown,
and Cleanview's ~90 GW — which *is* a BTM figure — is for that reason not comparable with it. The
national magnitude is sourced; the quantity the model actually needs is still not.

**A Texas ratio recorded in the first pass is withdrawn.** That pass set GEM's 40,000 MW against
dcmap.us's 74,894 MW ERCOT pipeline and called the result "roughly 53% of that pipeline", glossed as
about half the buildout being behind the meter. The August figures run the same arithmetic to 103% on
ERCOT territory, or 90% against the like-for-like Texas state pipeline of 85,421 MW — and a share
cannot exceed 100%, which is the proof that it was never a share. It is a capacity-matching ratio:
roughly one MW of announced gas per MW of announced data center capacity, about what self-supply would
require. Nothing in either tool is derived from it.

**A prior claim here was wrong and is withdrawn.** This section previously said the trackers
"neither publishes a regional breakdown". Cleanview **does** hold a project-level dataset with state
detail — it is paywalled at $4,900 — and GEM publishes a Texas figure openly. *Not published* and
*not free* are different problems and should not be conflated.

**Behind-the-meter renewables remain genuinely unsourced.** No source was found quantifying on-site
renewable capacity at US data centers. EIA's small-scale solar series covers systems under 1 MW by
state and sector, not by facility type, so data center on-site solar cannot be separated from any
other commercial installation. The gas trackers exist because turbines need air permits and are
visible from orbit; on-site solar leaves no equivalent trail.

#### What is sourced, and what is not

| | Status |
|---|---|
| Grid-average fossil share, per balancing authority | **Sourced** — EPA eGRID 2023, 69 BAs |
| Fuel classification (fossil = coal+oil+gas+other fossil) | Sourced convention, but **nuclear counted clean is a choice** |
| Share of load achieving hourly-matched clean supply | **Not known.** Policy dial |
| Behind-the-meter gas share by region | **Derived and APPLIED 2026-09-08, as bounds.** GEM's unit-level release splits its 189 GW into 75 GW confirmed entirely captive, 31 GW exporting, and 83 GW (43.8%) unrecorded. The tool applies the *low* bound by default and exposes Ignore / Confirmed / Upper bound. PNM and five states carry no value at all, their numerator exceeding their whole pipeline. See §3.12 |
| Behind-the-meter *renewable* capacity at data centers | **Not known, and no source found.** EIA's distributed-solar series is <1 MW by state and sector, not facility-typed |

Further eGRID caveats: it reports where generation *occurs*, not where it is *consumed*, so imports
and exports are not captured; and 2023 data published 2025 is two years stale in a fast-moving grid.

#### What policymakers would have to require data centers to report

The design above is only enforceable if three things are reported, and none is mandatory today:

1. **Hourly metered consumption**, per facility. Several state large-load tariffs already require
   interval metering as a condition of service, so the precedent exists — but it is not federal and
   not universal. Without it, hourly matching cannot be verified and part 2 collapses back to
   annual REC accounting.
2. **Hourly granular certificates, with certificate serial numbers and timestamps, surrendered for
   the specific hour of consumption** — not an annual percentage and not an unbundled REC of any
   vintage. This is what EnergyTag's Granular Certificate standard is built to carry. **Explicit
   design decision: unbundled RECs do not qualify for the clean rate, full stop** — not annual ones,
   not ones marketed as "hourly," not ones bundled after the fact. Only a certificate that is itself
   timestamped to the hour of generation and surrendered against the same hour of consumption
   qualifies. This is a deliberate and considered choice, not a default the tool fell into: it
   forecloses a facility buying certificates from a wind farm commissioned a decade earlier and
   claiming credit for causing nothing. It does NOT require a physical wire or a same-node PPA —
   verifiability is enforced through time resolution (the certificate's timestamp matching the
   consumption hour), not through physical proximity, so a verifiable virtual hourly-matched
   contract qualifies exactly as a physical one does.
3. **Behind-the-meter generation: capacity, fuel and metered output**, reported separately from grid
   purchases. Without it, part 3 is unenforceable and the largest loophole stays open.

A fourth would materially improve every forecast in this tool, though it is not needed to run the
tax: **facility-level load by interconnection status** — requested, under study, contracted, energised.
That is what would let anyone compute a real large-load completion rate instead of inferring one
from two sources, as §3.6 has to.

#### If the goal is to reward facilities that actually DRIVE decarbonization

The three items above establish **what a facility consumed and what it was matched to**. They do not
establish whether that clean supply **would have existed anyway**. A facility can buy unbundled
certificates from a wind farm commissioned in 2012 and be classified 100% clean while having caused
nothing. Sharpening the fossil:clean ratio (2:1 -> 4:1) would simply pay it more for causing nothing.

Distinguishing *procurement* from *causation* — additionality — needs four further items, none of
which is reported anywhere today:

4. **Generator identity and commercial operation date** for every contracted resource. This is the
   minimum additionality test: a generator energised before the contract was signed cannot have been
   caused by it. Without commercial operation date, new build and repackaged existing supply are
   indistinguishable in the data.
5. **Contract execution date relative to the generator's financial close, plus contract tenure.**
   Sharper than item 4: an offtake agreement signed *before* the investment decision is what actually
   finances new generation. A 15-year pre-construction power purchase agreement plausibly caused the
   plant; a one-year certificate strip bought afterwards did not. Tenure is the part that carries the
   financing signal, and it is exactly what annual percentage reporting discards.
6. **Interconnection queue position and status of each contracted new resource.** Distinguishes
   "contracted with a plant that will exist" from "contracted with a project that may never clear the
   queue." Also the input this tool needs most for its buildout figures (§3.6).
7. **Demand flexibility actually delivered: hours curtailed, MW reduced, and on whose instruction.**
   Arguably the single most valuable thing a large load can do for grid decarbonization — flexible
   load is what lets a system absorb more variable renewables without firm fossil backup. It is
   completely invisible in every dataset this project has found, and a facility doing it well is
   currently indistinguishable from one doing none of it.

**If only one could be required, it should be item 7 paired with item 1.** Hourly consumption plus
verified curtailment is a direct measurement of whether a facility helps or hinders the grid it sits
on, and unlike additionality it does not require adjudicating a counterfactual.

**The honest difficulty.** Additionality is contested and expensive to verify. Carbon-offset markets
have litigated it for two decades without settling it, and a tax rate that depends on an
adjudicated counterfactual invites exactly the disputes a per-kWh excise is meant to avoid. Items 4-6
are directionally right and administratively hard; item 7 is directionally right and administratively
easy, because curtailment is already metered by the system operator that called for it. That
asymmetry should drive the drafting.

**A structural warning.** Rewarding clean facilities more means widening the fossil:clean ratio, not
raising `k` — `k` scales both rates together and leaves the ratio untouched at every value (§3.9).
And widening the ratio **changes the indexing formula**: at ratio R the correction becomes
`(1 + (R-1)*ref) / (1 + (R-1)*m)`, not `(1 + ref) / (1 + m)`. Nothing in the code announces this. A
future change from 2:1 to 3:1 that left the indexing formula alone would silently under-correct by
about 10 percentage points, and k = 1 would stop being revenue-neutral.

### 3.8 A limitation the tool does not model: the rate effect

The tool reports what a household **receives**. It does not model what the same buildout does to the
**rate that household already pays** — and those can move in either direction.

LBNL's *Retail Electricity Price Trends and Drivers* (July 2026), held in
`sources/11-bill-benchmarks/lbnl-retail/`, records both directions:

> "Georgia PSC approved utility to serve new large loads, conditioned on growth placing downward
> pressure on bills of $8.50/mo for residential customers from 2029-2031"

> "PG&E estimates that 10 GW of data center growth could lower customer bills by 10% by utilizing
> existing infrastructure and spreading fixed costs across more electricity usage"

Large loads can *lower* residential bills by spreading fixed system costs across more consumption,
or *raise* them by driving new investment. Which happens is not a physical fact but an allocation
decision:

> "Cost allocation practices will determine how cost increases or savings are allocated to different
> customers and will therefore determine the net impacts of demand growth on customer-specific prices."

That is exactly what the large-load tariffs and FERC's RM26-4 docket are contesting.

**Why this is not modeled.** The numbers are utility-specific, contested, and mostly projections
filed by parties seeking rate approval rather than measured outcomes. Both figures above are of that
kind. Putting them into the tool would import advocacy estimates into an otherwise measured chain.

**What this means in practice.** When presenting a figure like "$85.95 a month in PJM", the complete
statement is *"…and the underlying rate may also have moved because those data centers exist, in a
direction that depends on how their costs are allocated."* The tool does not net that out, and
should not be described as if it did.

### 3.9 Grid-mix indexing — the fix for a perverse incentive

**The problem, stated plainly.** Fossil-matched electricity is taxed at twice the clean rate. That
ratio is the point: it gives a data center a reason to procure clean supply. But it has a second,
unintended consequence. As the *regional grid* decarbonizes, less electricity falls in the fossil
category, so total revenue falls, so **the rebate to burdened households shrinks — even though
nothing about those households changed.**

That hands an energy-burdened household, and any regulator answerable to them, a financial reason to
prefer a dirty grid. The tax would be paying people to root against decarbonization. This surfaced
when the ERCOT numbers were examined: a high fossil share produces a large rebate, and the mechanism
does not care that the fossil share is the thing the policy is supposed to reduce.

**The fix.** Multiply the tax rate by a factor keyed to the region's generation mix:

    rate multiplier = ((1 + refFossil) / (1 + gridMixNow)) ^ k

where `refFossil` is the region's **measured** eGRID fossil share (the reference point) and
`gridMixNow` is the grid mix being modeled. `k` sets how hard the rate leans:

| k | Behavior | What a household sees as the grid cleans |
|---|---|---|
| 0 | No indexing | Rebate **falls** — the perverse case |
| 1 | Revenue-neutral | Rebate **unchanged** by the grid mix |
| 2 | Rewards decarbonization *(the default)* | Rebate **rises** |

**Why it must key to the grid, not the facility.** This is the one part that is easy to get wrong,
and getting it wrong is silent. If the index is keyed to the *facility's own* matched fossil share
`fs` rather than the regional mix, the algebra collapses:

    (fs × 1.9 + (1 − fs) × 0.95) × (1 + ref) / (1 + fs)  =  0.95 × (1 + ref)

— a constant. The facility would pay the same rate whether it procured 0% or 100% clean supply,
destroying the 2:1 incentive the tax exists to create. Keying the index to the regional grid, which
the facility does not control, preserves both effects: the facility still halves its own rate by
procuring hourly-matched clean supply, *and* the household's rebate grows as the region cleans up.
The distinction is enforced in the calc core by keeping `fossilShare` (facility) and `gridMixNow`
(region) as separate inputs, with a comment saying why.

**What is measured and what is chosen.**

- **Measured:** `refFossil`, the reference grid mix — EPA eGRID balancing-authority fossil share,
  computed by `sources/.../egrid-fossil-share.py`, one value per region.
- **Chosen:** `k`. There is no empirical value for it and there could not be. It is a statement about
  how strongly a legislature wants to reward decarbonization, in the same category as a tax rate.
  It is flagged **Illustrative** in the manifest and appears in the gap table above.

**Why the default of k = 2 is safe to ship.** At `gridMixNow = refFossil` the multiplier is exactly
1.000× for every value of k. Since the tool opens with each region's grid set to its measured mix,
the default changes **no** headline figure, no known-answer table, and no citation elsewhere. The
dial only bites once the user moves the grid slider — which is precisely its purpose.

**The symmetry, stated on purpose.** A mechanism that pays households more for a cleaner grid
necessarily pays them less for a dirtier one. The tool shows both directions rather than only the
flattering one. Whether that downside symmetry is acceptable is a policy judgment, not a modeling
one, and it should be decided deliberately rather than discovered after adoption.

**What indexing does not do.** It does not change the 2:1 fossil-to-clean ratio (§3.7), so
procurement incentives at the facility level are untouched. It does not model any behavioral
response by generators. And it is not a cost-causation argument: the rate rises with grid cleanliness,
not with what a data center costs the system — those are different justifications and should not be
presented as the same one.

### 3.10 Rate structure: flat vs differentiated

**The choice.** The tool models two ways of levying the same money.

- **Differentiated** — fossil-matched electricity taxed at twice the clean rate (§3.7).
- **Flat** — one rate on every taxed kWh, no fuel classification at all.

**How the flat rate is set.** It is the differentiated design's *blended* rate at the region's
**measured** grid mix:

    flatCents = refFossil x fossilCents + (1 - refFossil) x renewCents

PJM at the moderate tier: 0.6004 x 1.9 + 0.3996 x 0.95 = **1.52 c/kWh**. This is chosen so the two
structures raise **identical revenue at enactment**, making the comparison like for like. It is *not*
a proposal for what a flat rate should be — a legislature would set a rate directly. It inherits
every uncertainty in both the tier rates and the eGRID fossil share.

**What flat changes.**

| | Differentiated | Flat |
|---|---|---|
| Tax base | kWh **classified by fuel** | metered kWh |
| Reporting required | hourly-matched clean supply, separately metered BTM gas — **none of it required today** (§3.7) | consumption, already metered and audited |
| Revenue as the grid cleans | **falls** | unchanged |
| Perverse incentive | must be corrected by the k dial (§3.9) | **cannot arise** |
| Facility procurement signal | 0.95 c/kWh at the moderate tier | **none** |

**Why this matters more than it looks.** The perverse incentive in §3.9 is a property of the *tax
base*, not of the rates. Under a flat structure the base is kWh, and kWh do not get cleaner, so
revenue is structurally independent of the grid mix. The k dial — the only parameter in this entire
model with no source and no possible source — becomes unnecessary rather than justified.

**What is genuinely given up.** The 0.95 c/kWh gap between fossil and clean rates is the facility's
reason to procure clean supply. Against PJM's measured 8.11 c/kWh industrial price that is **12% of
what they pay for power** (16% at the low tier, 37% at the aggressive tier). That is a real signal,
and dropping it is a real cost — it should not be waved away.

**The argument for dropping it anyway** is the Tinbergen rule: one instrument per objective. This
tax's objective is moving money to energy-burdened households. Clean procurement is a different
objective, already addressed by production and investment tax credits, state renewable standards, and
the hourly-matching rules regulators are currently writing. A clean *supply standard* — procure X%
hourly-matched, rising — is a mandate rather than a price, so it does not leak the way a rate
differential does through power purchase agreements for already-existing generation.

**The argument for keeping it** is that the differential may not be an incentive at all but
**compensation**: dirtier generation imposes more health damage, so a higher rate on fossil-matched
load is payment for harm rather than a nudge. That framing survives the leakage critique entirely.
Note that it also implies the k dial should be *dropped*, since compensation should fall as harm
falls — the two justifications are not compatible and picking one is a policy decision.

**This tool takes no position.** Both structures are implemented, both are verified, and the tool
displays the divergence rather than recommending a side. What it will not do is present the flat
rate as sourced: it is derived arithmetic on top of two cited inputs, flagged as such.

### 3.10a Choosing a flat rate

**The anchor.** Virginia enacted **1.1 c/kWh**. It is the only rate a legislature has actually
adopted for this kind of tax, so it is the only external reference point that exists. The three
options the tool offers — **1.5c, 3c, 6c** — are roughly 1.4x, 2.7x and 5.5x that figure. They are
round numbers of the sort a statute names outright, and they are flagged **Illustrative**: nothing
in the data picks them.

**What each buys, PJM at its sourced 35.1 GW case:**

| Rate | Severe rebate | Burden 18.3% -> | Added to 8.11c industrial price |
|---|---|---|---|
| 1.5c | $84.80/mo | 11.9% | +36% |
| 3c | $169.60/mo | 5.6% | +73% |
| 6c | $339.21/mo | 0.0% | +146% |

**The fairness test all three survive.** Compared against what households in the *same* region
already pay (cit `residential-price`), a data center taxed at even 6c still pays less per kWh than
its neighbours do:

| | Industrial +6c | Residential |
|---|---|---|
| PJM | 14.11c | 16.47c |
| ERCOT | 12.15c | 15.19c |
| NYISO | 15.17c | 24.44c |
| CAISO | 29.33c | 35.25c |

**Six exceptions, and they were caught by a check rather than by inspection.** The claim holds in
all twenty-four regions at 1.5c and at 3c. At **6c it fails in 6 of the 24 regions** — PacifiCorp,
Florida Power & Light, Bonneville Power, LADWP, Puget Sound Energy and Salt River Project:

| Region | Data center at 6c | Households already pay |
|---|---|---|
| PacifiCorp | 14.11c | 13.07c |
| Florida Power & Light | 14.24c | 13.68c |
| Bonneville Power | 11.91c | 10.63c |
| LADWP | 26.70c | 24.10c |
| Puget Sound Energy | 15.05c | 14.62c |
| Salt River Project | 13.73c | 13.44c |

Every one is a region where the industrial and residential prices already sit unusually close
together, so a 6c adder is enough to cross them over. The tool names the exception on screen in each
affected region rather than repeating a claim that is false in front of the reader, and it computes
the count of regions where the comparison still holds rather than hardcoding it. A permanent check
pins the exception set, so any change in the underlying prices fails the build.

**This set has already changed once, which is the argument for pinning it.** It was a single region
(PacifiCorp) while the tool covered fourteen; expanding to twenty-four on 2026-09-03 took it to six.
An earlier draft of this section had asserted "all fourteen regions" on the strength of a ten-region
spot check — exactly the error the checklist exists to catch, and it did.

**Where the real limit is — and it is not the industry.** At 3c, PJM's severe tier reaches **5.6%**
burden, already below the 6% threshold the tool targets. At 6c the PNM rebate is **$397.84/mo
against a $77.73 median bill**. Under the chosen delivery design (§3.5) the surplus becomes a
cashable check, so beyond a certain rate the policy stops being energy assistance and becomes an
income transfer. That may be defensible, but it is a *different* policy and should be argued as one.
The binding constraint on the high end is the household side, not affordability for data centers.

**The assumption that bites hardest at 6c.** Elasticity is **not estimated** and defaults to zero
(§3.6), so no relocation response is modeled anywhere. A 146% increase on PJM's industrial price is
exactly where that omission matters most. The tool does not claim 6c is survivable for the industry;
it claims only that the *price level* remains below residential. Those are different claims.

### 3.11 Indexing the rate to CPI

**The problem.** A per-kWh excise is a **nominal** figure. Left alone it loses real value every year
with no vote taken — Virginia's 1.1c will buy less rebate each year until a legislature revisits it.
A policy that shrinks by default is a policy that quietly repeals itself.

**The mechanism.** Indexing is modeled as a deflator on the result, not a change to the revenue
arithmetic, because that is precisely what it is:

- **Indexed** — the nominal rate grows with CPI, so the real rebate is **constant**.
- **Unindexed** — the nominal rate is fixed, so the real rebate decays by `(1 + cpi)^-years`.

**The rate.** BLS series `CUUR0000SA0` (CPI-U, U.S. city average, all items, **not seasonally
adjusted**), annual averages 1913–2025, downloaded to `sources/15-inflation/` and extracted by
`cpi-rate.py`. NSA is deliberate: it is the series BLS designates for escalation contracts.

| Window ending 2025 | CAGR |
|---|---|
| 10 years | 3.110% |
| **20 years (the default)** | **2.531%** |
| 30 years | 2.524% |
| 40 years | 2.778% |
| 50 years | 3.643% |

**2.53%** is used because the 20- and 30-year windows agree to within 0.01 points, so the figure is
not an artefact of window length. The 10-year window is visibly elevated by the 2021–2023 episode;
the 50-year still carries the 1970s–80s regime.

**What this is not.** It is **not a forecast.** It is a realized historical average offered as a
slider default, and the projection-years control defaults to **0**, so at the default nothing moves
and no figure elsewhere in the tool is affected. Any year above zero is a what-if and is labeled as
one on screen. Future inflation is unknowable and nothing here should be read as predicting it.

**Why it is worth a clause in the statute.** At 2.53%, an unindexed rate loses about **22% of its
real value in ten years** and **roughly half in twenty-seven**. That is a cut nobody votes for,
arriving on a schedule nobody debated. Indexing is one sentence of drafting and it is the difference
between a policy that holds the size its legislature chose and one that expires by arithmetic.

### 3.5 Delivery: bill credit first, then a check

**Design decision.** The rebate is applied as an **automatic credit on the household electricity
bill**. Where the rebate is larger than the bill, the surplus is **paid out as a cashable check**.

Because delivery is automatic, there is no enrollment step to fall out of, so the intended take-up is
**100%** and that is the tool's default. This is a *design parameter, not an estimate* — it is not a
claim about what any real program achieves.

**Why the split is modeled.** At modest buildouts the rebate is smaller than a typical bill, so it
arrives entirely as a discount. At high buildouts it can exceed the whole bill, at which point most
of it arrives as cash. Those are materially different things — one lowers a bill, the other is
income — so the tool reports them separately.

The bill benchmark comes from **EIA Form 861**: bundled residential revenue ÷ residential customers
÷ 12, computed per region from the same file used for household counts. PJM works out at
**$134.07/month**, PNM at **$91.57/month**.

That all-customer average is only a fallback. For regions where the work has been done, the tool
uses the bills of **energy-burdened households specifically**, which turn out to be materially
different.

**Burdened households do not simply have bigger bills.** Energy burden is a bill-to-income *ratio*,
so it is driven by low income. Measured from LEAD's electricity expenditure data:

| | Median monthly electricity bill |
|---|---|
| PJM, severely burdened | **$107.51** |
| PJM, all customers (EIA-861) | $134.07 |
| New Mexico, severely burdened | **$77.73** |
| New Mexico, *un*burdened (under 6%) | $82.66 |

In New Mexico the severely burdened median is *lower* than the unburdened median. Using the
all-customer average would overstate the typical burdened household's bill by roughly 25% in PJM,
and therefore understate how many of them receive a cash check.

**The spread is what matters, not the median.** PJM's severely burdened households run from about
$51 at the 10th percentile to $214 at the 90th — a 4× range. A single crossover point would hide
who actually gets a check, so the tool reports **the share of burdened households whose bill the
rebate fully covers**, computed from the distribution.

**How it is combined.** Percentiles cannot be averaged across states, so LEAD households are
bucketed into $10 bill bands per state and per tier, then summed band by band using the same
EIA-861 weights as the household counts. Percentiles are read off the combined curve.

#### Seasonality — why the headline coverage figure is an annual average

Every bill figure above is an annual total divided by twelve. Real bills swing hard with the
seasons, so a **flat** monthly rebate covers more of the bill in spring and autumn and less at the
winter or summer peak — least, in other words, exactly when households are most burdened.

That swing is now measured rather than assumed, from **EIA Form 861M** monthly residential revenue
and customers per state (calendar year 2025), combined into regions with the same EIA-861 weights:

| Region | Peak month | Trough month | Swing |
|---|---|---|---|
| PJM | 1.43 (July) | 0.73 (April) | **1.95×** |
| NYISO | 1.42 (July) | 0.77 (April) | 1.83× |
| PNM | 1.43 (Jul/Aug) | 0.74 (April) | 1.93× |
| Arizona Public Service | 1.67 (August) | 0.62 (February) | **2.67×** |
| Portland General Electric | **1.30 (January)** | 0.80 (May) | 1.61× |

Note the shapes differ. Most regions peak in summer, but Portland General Electric peaks in
**January** — Pacific Northwest electric heating. A model assuming one national seasonal shape
would get that region backwards.

**What this changes.** At PJM, 35.1 GW, moderate tier, the rebate covers the full bill for **10.9%**
of severely burdened households *on an annual average* — but only **5.5% in July**, against 21.9% in
April. The annual-average figure is not wrong, but quoted alone it overstates the months that matter
most. The tool now reports the annual figure, the months-covered count, and the worst and best month
side by side.

**Design implication, not yet decided.** A flat credit is administratively simplest but leaves
households short at the peak. A seasonally weighted credit — the same annual money, distributed
unevenly — would track need better, and is easy to administer given the rebate is already a bill
credit. LIHEAP effectively does this with a seasonal lump sum.

**Script:** `sources/11-bill-benchmarks/region-seasonality.py`

**Scripts:** `sources/03-household-burden/lead-bill-histograms.py`,
`sources/11-bill-benchmarks/region-bill-distribution.py`, `sources/11-bill-benchmarks/region-bills.py`

Currently compiled for **PJM and PNM only**. Other regions fall back to the all-customer average and
say so on screen.

The take-up slider exists only as a **stress test** of a non-automatic delivery mechanism.
Application-based energy programs routinely reach a fraction of those who qualify, so the slider
answers "what would happen if delivery were not automatic?". **Any figure below 100% is a what-if and
must not be presented as a forecast.**

**Parked for later: the reserve fund.** The machinery below is built and tested, but it only
activates below 100% take-up — which the design says should not happen. Treat it as a side scenario
awaiting a decision, not a live part of the model.

Being eligible and receiving a payment are different things, so the tool separates them.

**The per-household entitlement is set by the number of households that are *eligible*, not the
number that actually claim.** A low take-up rate therefore does **not** inflate other households'
checks — an earlier version did exactly that, which was backwards. Checks stay the same whatever the
take-up rate.

The remainder is **simply not paid out**. Below 100% take-up some revenue is raised and not
distributed, and the tool says so without asserting where it goes, because that is a policy choice
this tool does not model.

**This paragraph described a reserve fund until 2026-09-13.** An earlier version accumulated the
unclaimed remainder in a Treasury-managed fund and projected an annual draw on it. The fund was
removed at the project owner's request (see the correction log below), but this section was not
updated with it and went on describing a feature the tool no longer had. It never entered the
dividend arithmetic — it was downstream display — so no figure changed when it went.

Money is conserved throughout: paid out plus unclaimed equals revenue exactly, checked across 30
region/tier/take-up combinations.

---

## 4. What is still NOT properly sourced

Sixteen rows, covering seven distinct issues. All are flagged inline in the tool, not just here.

(The 100% take-up rate is *not* on this list: it is a design decision — an automatic bill credit has no enrollment step — not an unsourced estimate. See §3.5.)

| Citation id | Value | What it is | Why it matters |
|---|---|---|---|
| `ercot-buildout` | 59.7 / 99.0 / 259.8 GW | ERCOT buildout | **Estimate.** 23% completion on ERCOT's own filed increment, which ERCOT itself warns against using |
| 8, 13, 17, 21, 25, 29, 33, 37, 41, 45 | Round MW figures | Buildout for **APS, MISO, CAISO, ISO-NE, SPP, Georgia Power, both Duke territories, PacifiCorp, PGE** | **PLACEHOLDER — NOT SOURCED.** Ten regions where no data center forecast or queue total was found. Numbers exist only to give the slider a scale |
| `pnm-buildout` | 0 / 8 / 320 MW | PNM buildout | **UPDATED 2026-09-03**: migrated to dcmap.us, same method as the other regions, after the earlier IRP-capacity proxy was rejected ("you shouldn't use generation capacity at all") — see `files/REGIONS.md`'s Correction Log |
| `miso-buildout` | 16,500 / 35,500 / 48,000 MW | MISO buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a MISO-specific forecast |
| `caiso-buildout` | 120 / 1,400 / 3,200 MW | CAISO buildout | dcmap.us apportioned by state share, **net of a confirmed Imperial Irrigation District territorial exclusion (2026-09-03)** — a third-party facility tracker, not a CAISO-specific forecast |
| `isone-buildout` | 0 / 2,600 / 3,400 MW | ISO-NE buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a ISO-NE-specific forecast |
| `spp-buildout` | 3,000 / 22,000 / 24,500 MW | SPP buildout | dcmap.us apportioned by state share, **net of two confirmed Southwestern Public Service (New Mexico) projects reassigned here from PNM (2026-09-03)** — a third-party facility tracker, not a SPP-specific forecast |
| `gapower-buildout` | 500 / 7,300 / 9,800 MW | Georgia Power buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Georgia Power-specific forecast |
| `dukecar-buildout` | 250 / 2,100 / 3,500 MW | Duke Energy Carolinas buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Duke Energy Carolinas-specific forecast |
| `dukeprog-buildout` | 170 / 1,200 / 2,200 MW | Duke Energy Progress buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Duke Energy Progress-specific forecast |
| `pacificorp-buildout` | 3,400 / 15,500 / 18,500 MW | PacifiCorp buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a PacifiCorp-specific forecast |
| `pge-buildout` | 80 / 400 / 1,400 MW | Portland General Electric buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Portland General Electric-specific forecast |
| `aps-buildout` | 400 / 4,000 / 4,900 MW | Arizona Public Service buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Arizona Public Service-specific forecast |
| `tva-buildout` | 759 / 4,234 / 6,996 MW | TVA buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a TVA-specific forecast — **the least reliable buildout figure in the tool**: only 44% of it comes from Tennessee, the rest from co-op pockets in KY/GA/MS/AL that sit away from those states' known clusters, and dcmap.us's terms prevent the facility-level check that would resolve it |
| `fpl-buildout` | 8 / 785 / 1,390 MW | Florida Power & Light buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Florida Power & Light-specific forecast |
| `dukefl-buildout` | 3 / 264 / 467 MW | Duke Energy Florida buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Duke Energy Florida-specific forecast |
| `xcelco-buildout` | 524 / 1,138 / 1,293 MW | Xcel Colorado buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Xcel Colorado-specific forecast |
| `bpa-buildout` | 169 / 961 / 2,278 MW | Bonneville Power buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Bonneville Power-specific forecast, **net of a confirmed Wyoming territorial block (2026-09-03)**: Bonneville's only Wyoming presence is Lower Valley Energy in Jackson Hole, ~350 miles from the Cheyenne cluster where the state's data centers actually are |
| `ladwp-buildout` | 16 / 173 / 405 MW | LADWP buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a LADWP-specific forecast |
| `nvenergy-buildout` | 2,769 / 7,890 / 8,715 MW | NV Energy buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a NV Energy-specific forecast |
| `pse-buildout` | 105 / 534 / 1,085 MW | Puget Sound Energy buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Puget Sound Energy-specific forecast |
| `srp-buildout` | 324 / 3,413 / 4,172 MW | Salt River Project buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Salt River Project-specific forecast — Arizona's pipeline is split with APS by customer share (39%/33%), and data centers cluster far more tightly than households do |
| `alpower-buildout` | 169 / 896 / 1,101 MW | Alabama Power buildout | dcmap.us apportioned by state share — a third-party facility tracker, not a Alabama Power-specific forecast |
| `btm-renewables-gap` | no source found | Behind-the-meter **renewable** capacity at data centers | **NEEDS SOURCE.** No source found quantifying it; EIA's distributed-solar series is <1 MW by state and sector, not facility-typed. Gas is trackable via air permits and satellite imagery; on-site solar leaves no equivalent trail |
**The uniformity-clause question, recorded and not cited (2026-09-24).** Whether a state constitutional uniformity clause would restrict charging data center electricity at a rate other industrial customers do not pay is **not established by any document in `/sources`** — searched, not found. It sat in the tools as citation 254 for part of one day and was removed with the sentence that carried it. It is kept here because a reader is likely to raise it and the honest answer is that this project cannot answer it. What would settle it: the constitutional text of the states in question, plus any state supreme court decision applying such a clause to a utility or energy excise. `sources/04-tax-legal/` holds enacted large-load statutes and tariff filings, none of which reach the constitutional question.

| `rate-ratio` | 2 : 1 (adjustable 1:1 to 10:1) | Fossil:clean rate ratio | **Design parameter, no source exists or could.** Became a control on 2026-09-24, replacing the flat/differentiated structure choice: at 1:1 the two rates are equal and the fee IS flat. The range is offered because the trade runs both ways — a steeper ratio is a stronger procurement signal and a more fragile dividend. New Mexico, today's revenue held fixed: at 2:1 a facility avoids 50% of the rate by procuring clean and the dividend keeps 64.5% of its value on a fully clean grid; at 10:1, 90% and 16.8%. Holding the dividend whole needs a 1.55× rate climb at 2:1 and 5.95× at 10:1 — which is what makes indexing the price of a steep ratio. **The indexing formula moves with it:** `((R·ref + 1 − ref) / (R·mix + 1 − mix))^k`, reducing to the published `((1 + ref) / (1 + mix))^k` only at R = 2 |
| `loadfactor` | 90% | Data center load factor | Multiplies every revenue figure |
| `split` | 70% / 30% | *(retired as a default)* | Superseded by measured grid fossil shares per region — §3.7. Retained as the slider's fallback only |
| `elasticity` | 0 | Price sensitivity (elasticity) | **Cannot be estimated** — §3.6. Defaults to 0 |
| `rate-low` | 1.3¢ / 0.65¢ per kWh | Low tier fee rate | **Design parameter.** Anchored ~18% above Virginia's enacted 1.1¢ — the only per-kWh rate a legislature has actually adopted on data center electricity. The lowest tier because it sits nearest the one real-world precedent |
| `rate-mod` | 1.9¢ / 0.95¢ per kWh | Moderate tier fee rate — **the tool's default** | **Design parameter.** The rate behind every headline figure unless the reader changes tier, so it carries more weight than the other two and is flagged accordingly. ~73% above Virginia's 1.1¢. Being the default is not a recommendation |
| `rate-agg` | 3.0¢ / 1.5¢ per kWh | Aggressive tier fee rate | **Design parameter.** ~2.7× Virginia's 1.1¢. Marks the upper end of the range the reader is asked to consider, not a forecast of what any legislature would pass |
| `takeup` | 100% | Share of eligible households receiving the dividend | **Design parameter, not an estimate.** Delivery is an automatic bill credit, so there is no enrollment step to fall out of. The slider below 100% is a stress test of a *different* delivery mechanism and must never be quoted as a forecast |
| `takeup-redistribution` | not redistributed | What happens to unclaimed money | **Design parameter.** Unclaimed money does not inflate other households' checks — the per-household entitlement is set by the eligible population, so checks are unchanged by take-up. Where the remainder goes is a policy choice this tool does not model |
| `delivery` | credit first, then check | How the dividend reaches the household | **Design parameter.** Automatic credit on the electricity bill; surplus above the bill paid as a cashable check. This is *why* take-up is 100% by design, so the two rows stand or fall together |
| `fossil-design` | grid average default; hourly granular certificates only; BTM gas at fossil rate | How a kWh is counted fossil or clean | **Design parameter — the rule, not a measurement.** Every number the rule produces is measured (eGRID, GEM); the rule itself is a choice. Unbundled RECs do not qualify at any vintage, which is the design's central lever and its most contestable one. Was filed as Derived until 2026-09-09, and its note also claimed the hourly-matching documents had never been gathered — they were gathered 2026-09-07 and are in `sources/12-hourly-matching/` |
| `eligibility` | burdened only / everyone tiered / everyone equal | Who receives the dividend | **Design parameter — the divisor, not a finding.** All three settings raise identical revenue; widening eligibility divides the same money further. Added 2026-09-13 after the tool was presented and the commonest response was to widen it. The burden data can say who spends the most of their income on energy; it cannot say whether payment should be confined to them. Virginia at 8,180 MW: $126.07 burdened-only, $30.78/$15.39 tiered, $16.80 equal |
| `ratio` | 2 : 1 | Fossil:clean rate ratio, and severe:moderate payout ratio | **Design parameter — no study sets it and none could.** Was filed as "Derived" until 2026-09-09, while its own source line read "this project's design decision"; nothing is derived from a decision. What constrains it: the gap must be big enough to change procurement — 0.95¢/kWh at the moderate tier, about 12% of the 8.11¢ PJM's large customers pay. What it does **not** do: change how much revenue is collected. It sets who inside a territory pays more; the flat structure raises the same money with no ratio in it |
| `grid-indexing` | k = 2.00 | Grid-mix indexing strength | **Pure policy dial — no source exists or could.** §3.9. At today's measured grid mix the multiplier is exactly 1.000×, so the default changes no other figure |
| `rate-flat-options` | 1.5¢ / 3¢ / 6¢ | Round flat-rate options | **Policy choices, not derived.** §3.10a. Anchored to Virginia's enacted 1.1¢ — the only rate a legislature has actually adopted for this |
| `fund-return` | 0% | Reserve fund return | Parked feature; 0% so no growth is claimed |
| `nm-buildout` | 2,450 / 10,710 / 11,170 MW | New Mexico buildout (state-level companion tool) | dcmap.us's own state total, direct — see below |
| `va-buildout` | 8,180 / 29,460 / 47,700 MW | Virginia buildout (state-level companion tool) | dcmap.us's own state total, direct — see below |
| `nd-buildout` | 600 / 2,900 / 4,300 MW | North Dakota buildout (state-level companion tool) | dcmap.us's own state total, direct — see below |
| `ne-buildout` | 5 / 45 / 480 MW | Nebraska buildout (state-level companion tool) | dcmap.us's own state total, direct, and genuinely small — the lowest pipeline-record coverage of any state in this pilot (37.5%) — see below |
| `ia-buildout` | 1,600 / 4,300 / 6,200 MW | Iowa buildout (state-level companion tool) | dcmap.us's own state total, direct — see below |
| `or-buildout` | 170 / 870 / 3,200 MW | Oregon buildout (state-level companion tool) | dcmap.us's own state total, direct — see below |
| `al-buildout` | 300 / 1,600 / 2,000 MW | Alabama buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ak-buildout` | 0 / 3,100 / 3,100 MW | Alaska buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `az-buildout` | 980 / 10,400 / 12,700 MW | Arizona buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ar-buildout` | 400 / 2,200 / 2,300 MW | Arkansas buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ca-buildout` | 150 / 3,000 / 5,200 MW | California buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `co-buildout` | 700 / 1,500 / 1,700 MW | Colorado buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ct-buildout` | 0 / 660 / 1,300 MW | Connecticut buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `de-buildout` | 0 / 1,200 / 1,400 MW | Delaware buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `dc-buildout` | 0 / 0 / 1 MW | District of Columbia buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `fl-buildout` | 16 / 1,500 / 2,600 MW | Florida buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ga-buildout` | 990 / 14,300 / 19,200 MW | Georgia buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `hi-buildout` | 2 / 2 / 28 MW | Hawaii buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `id-buildout` | 210 / 280 / 440 MW | Idaho buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `il-buildout` | 1,100 / 11,700 / 13,700 MW | Illinois buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `in-buildout` | 4,500 / 8,700 / 11,100 MW | Indiana buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ks-buildout` | 0 / 2,100 / 2,200 MW | Kansas buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ky-buildout` | 400 / 8,400 / 8,800 MW | Kentucky buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `la-buildout` | 5,700 / 5,800 / 5,800 MW | Louisiana buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `me-buildout` | 0 / 2,000 / 2,100 MW | Maine buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `md-buildout` | 800 / 1,500 / 2,300 MW | Maryland buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ma-buildout` | 0 / 0 / 140 MW | Massachusetts buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `mi-buildout` | 1,000 / 2,200 / 2,800 MW | Michigan buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `mn-buildout` | 370 / 1,200 / 1,800 MW | Minnesota buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ms-buildout` | 1,500 / 1,500 / 3,500 MW | Mississippi buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `mo-buildout` | 1,000 / 4,600 / 5,500 MW | Missouri buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `mt-buildout` | 0 / 1,000 / 2,200 MW | Montana buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `nv-buildout` | 2,800 / 7,900 / 8,800 MW | Nevada buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `nh-buildout` | 0 / 0 / 14 MW | New Hampshire buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `nj-buildout` | 590 / 760 / 1,200 MW | New Jersey buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ny-buildout` | 360 / 5,100 / 6,900 MW | New York buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `nc-buildout` | 640 / 4,200 / 7,400 MW | North Carolina buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `oh-buildout` | 2,600 / 9,800 / 10,800 MW | Ohio buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ok-buildout` | 380 / 4,000 / 4,500 MW | Oklahoma buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `pa-buildout` | 5,800 / 19,600 / 20,900 MW | Pennsylvania buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ri-buildout` | 0 / 0 / 0 MW | Rhode Island buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `sc-buildout` | 20 / 2,200 / 3,100 MW | South Carolina buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `sd-buildout` | 0 / 570 / 1,200 MW | South Dakota buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `tn-buildout` | 170 / 1,900 / 3,700 MW | Tennessee buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `tx-buildout` | 30,200 / 85,400 / 98,800 MW | Texas buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `ut-buildout` | 4,300 / 18,700 / 19,600 MW | Utah buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `vt-buildout` | 0 / 0 / 2 MW | Vermont buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `wa-buildout` | 320 / 1,600 / 3,300 MW | Washington buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `wv-buildout` | 8,000 / 13,400 / 13,500 MW | West Virginia buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `wi-buildout` | 900 / 1,100 / 4,800 MW | Wisconsin buildout (state-level companion tool) | dcmap.us's own state total, direct |
| `wy-buildout` | 300 / 3,500 / 6,500 MW | Wyoming buildout (state-level companion tool) | dcmap.us's own state total, direct |

**A separate state-level tool exists alongside this one.** `data-center-tax-tool-by-state.html`
covers **all 50 states and DC**, using the identical calculation engine but a different
geographic unit — whole states instead of RTO/utility territories. Its buildout figures
(`nm-buildout` through `or-buildout` above) are dcmap.us's own state totals, used directly with no
apportionment step, which is the whole reason the state-level tool exists: dcmap.us, DOE LEAD, and
EPA eGRID's ST23 sheet all publish state-native data already, so the entire EIA-861-customer-share
apportionment layer this project built for the regions above — and the wrong-territory bugs that
layer produced (PNM, CAISO, SPP, all documented above and in `files/REGIONS.md`) — simply doesn't
apply. Its own manifest is `files/CITATIONS-by-state.md`; its own failure-condition checklist is
`files/TESTING-by-state.md`; its own source script is `sources/17-state-level/state-pilot.py`. It
shares this project's household/burden/price/bill/fossil-share source FILES (LEAD, EIA-861, eGRID)
but reads them by state directly rather than through `region-households.py`'s BA/utility crosswalk.

**Twenty-two of the twenty-four regions have no sourced buildout figure.** Only PJM and NYISO
publish forecasts; ERCOT publishes a queue it disclaims, and the remaining twenty-one carry a
dcmap.us state pipeline apportioned by EIA-861 customer share — an Illustrative estimate, not a
forecast by the operator or its regulator. The tool flags every one of them on its own panel. **Their household, burden, bill, seasonal and grid-mix data are
fully sourced** — those regions are useful for comparing *who bears the burden*, and not for
estimating revenue.

> Row numbers refer to `files/CITATIONS.md`, which is regenerated from the tool. They shift when
> rows are added — `verify.js` fails if any unsourced row stops being listed here, so this table
> cannot silently fall out of date.

If someone challenges the tool, these are the four places to challenge it first. Anyone presenting
this work should say so unprompted.

---

## 5. How to verify any of this yourself

```
osascript -l JavaScript verify.js              # 63 checks: arithmetic, sourcing, structure
osascript -l JavaScript sync-citations.js check # fails if the manifest drifted from the code
osascript -l JavaScript sync-testing.js check   # fails if expected results drifted from the code
python3 sources/03-household-burden/lead-burden-shares.py NM   # recompute one state from source data
python3 sources/11-bill-benchmarks/region-households.py        # recompute every region
```

`verify.js` enforces, mechanically, that every number displayed has a manifest row, that no manifest
row is unused, that unsourced numbers are flagged in the tool's own output, and that the generated
documents match the code.

---

## 6. Errors this process has actually caught

### Every source-note quotation, checked against its document (2026-09-24)

All 26 PDFs were extracted to text and every verbatim blockquote in all 25 `*.source.md` notes was
matched against the document it names. **71 fragments were checkable against a locally held
document. 70 are faithful. One was wrong.**

| outcome | count |
|---|---|
| matched exactly | 52 |
| matched once footnote markers, table gutters and page breaks are ignored | 8 |
| investigated individually and cleared | 10 |
| **wrong** | **1** |

The one error was in `doe-lead-tool.source.md` — the note whose stated purpose is warning that a
plausible attribution may not be in the cited document. It reported the DOE LEAD factsheet's national
low-income energy burden as **8.6%** where the document says **6.7%**, and appended a clause
("3 times higher than for non-low income households") absent from the document entirely. Nothing
downstream used it: no citation in either tool references that factsheet, and the 6% / 10% thresholds
come from ACEEE. Corrected, with the error recorded in the note rather than erased.

The ten cleared individually fall into three kinds, none of them defects:

- **Five quote web pages the project does not hold locally**, and each names its URL — PJM's Territory
  Served page, AES Indiana's Morgan County project page (three fragments), and the DOE LEAD *program
  page* as distinct from the factsheet PDF. Quoting a named URL is correct practice; only the checker
  was confused.
- **One quotes a document filed under a different folder** — `federal-congress.source.md` cites S. 5199
  text that lives in `04-tax-legal/`, and the quotation is exact.
- **Four failed on mechanics, not accuracy.** LBNL's "Over 750 GW of capacity withdrew in 2025" is
  present as `750 GW2 … requests3`, with footnote digits pressed against the figures; GEM's
  "189 gigawatts" carries a footnote marker mid-sentence that the note properly omits when quoting.

**What this exercise does not establish.** Eleven further fragments cite web pages or spreadsheets
with no local text form and cannot be checked this way at all. And a quotation being verbatim says
nothing about whether the passage supports the claim built on it — that remains a reading, which is
what pincites and source notes exist to expose.



Worth telling people, because it is the evidence that the discipline is doing work rather than
decorating.

- **A false attribution to DOE.** The 6%/10% burden thresholds were widely reported as DOE's. The
  downloaded DOE factsheet does not contain them, and DOE's own page defines *no* 10% severe
  threshold. The real source is ACEEE.
- **A phantom quote from PJM.** The household count was cited to a PJM page carrying "65 million
  people". That page has no population figure at all, and PJM's current fact sheet says 67 million.
- **General knowledge that slipped into the tool.** Capacity-factor examples ("solar ~20–30%,
  nuclear ~90%+") were written into the glossary from memory, with no source. Removed.
- **A placeholder that was badly wrong.** The assumed 25% moderately-burdened share was roughly
  2.5× the measured value in every region tested.
- **An answer key that contradicted itself.** 10 of 18 hand-computed test values violated the
  project's own "severe = exactly 2× moderate" rule.
- **An unsourced claim wearing a design-decision label — then sourced (2026-09-07).** The
  methodology asserted that "the granular-certificate infrastructure is early-stage", and the tool's
  citation described available hourly-matched supply as something "nobody publishes". Neither had a
  source. The project owner pushed back: hourly matching is practiced today by Google and Microsoft.

  Two things made this easy to miss. The claim sat inside the `fossil-design` citation, whose source
  field reads *"Design decision of this project"* — true of the rate structure, false of an empirical
  claim about how mature an industry practice is; the design-decision label made an unsourced
  world-claim look accounted for. And `source-gathering-checklist_2.md` already listed the documents
  that would settle it, all unchecked. The tool asserted an answer to a question it had recorded as
  unanswered.

  **The sources were then gathered** into `sources/12-hourly-matching/`, and they are more
  interesting than either the original claim or a flat reversal:

  - **Practice: established, and older than the certificates.** Google published a full hourly
    methodology in February 2021 and reports per-facility scores. Its Lenoir, North Carolina data
    center scored an **Annual CFE Score of 65.5%** in 2019, ranging from **40%** at 7pm in September
    to **91%** at noon in April. A hyperscaler was publishing hour-by-hour matching five years before
    this project called the infrastructure early-stage. That 40–91% spread is also the clearest
    published illustration of what the rate design targets: an annual REC would score it 100%.
  - **Microsoft is weaker evidence and is not cited as equivalent.** Its "100 percent of the time"
    matching is a 2030 commitment, restated in the future tense in the 2025 report, which reports
    34 GW of contracted carbon-free electricity and never uses the word "hourly".
  - **Certificate accreditation is genuinely recent** — which is where the original instinct had
    something to it. EnergyTag accredited its **first** issuers on **25 June 2025** (Energinet and
    Flexidao). The practice is ~5 years old at hyperscaler scale; the auditable regime is ~1 year old.

  What survives unchanged is the part the design rests on: none of this makes hourly reporting
  *mandatory*, and a tax authority cannot audit a voluntary disclosure. What remains unknown is the
  share of US data center load that could obtain qualifying certificates — the first accreditations
  are European, PJM-GATS and M-RETS are still ungathered, so that stays a policy dial.

  **The check that forced the rewrite was the useful part.** When the claim was first withdrawn, a
  guard asserted the sources were still listed as ungathered, so the NEEDS SOURCE flag was current.
  Ticking the checklist boxes made it fail immediately — exactly as intended, so a flag could not sit
  there forever pretending to be a substitute for the work. A second guard pins both withdrawn
  phrasings against return *as assertions* while allowing them quoted, since this log has to be able
  to say what it used to say.

- **An over-correction, and what it was reacting to (2026-09-07).** The project owner read Wyoming's
  output and said it did not make sense: the tool claimed 28,631 households moved out of severe
  burden, "100% of the way to the 6% threshold", while the line beneath said 100% of them had their
  whole bill wiped out and received a check. Those two statements sat badly together, and a
  footnote made it worse by claiming the burden figures "slightly overstate relief for gas-heated
  homes" — the opposite of what the arithmetic did.

  My first fix capped relief at the electricity bill and pushed the surplus into income, moving
  Wyoming from 0.00% to 7.82%. **That was wrong.** Under this tool's delivery design the surplus is
  paid as a *cashable check*, and money is fungible: a household receiving $900.63/mo pays its
  $119.49 electricity bill and its $199.30 gas bill and still has $581.84 left. Its net energy cost
  genuinely is zero. Capping relief assumed the household could not spend its own check on heating.
  Reverted the same day.

  The real defect was presentational, and the fix is a **breakdown** rather than a different number:
  the tool now shows how much of the rebate clears the electricity bill, how much covers heating,
  and how much is left over as cash — the last being an income transfer rather than energy
  assistance. Burden can still reach 0%, but only where the rebate demonstrably covers the whole
  bill, and the reader can see that rather than infer it. `verify.js` pins both halves: every dollar
  of the breakdown is accounted for and never exceeds the bill it pays, and burden reaches zero only
  where the full energy bill is actually covered.

  Two lessons worth keeping. A disclosed inconsistency is still an inconsistency — the footnote had
  been contradicting the arithmetic for some time and no check caught it, because checks tested the
  numbers and not whether the prose agreed with them. And when output looks wrong, the fault may be
  in what is *shown* rather than in what is *computed*; changing the number was the faster fix and
  the wrong one.
- **A correct filter applied to the wrong job — the largest error found so far (2026-09-03).**
  Electricity prices were understated in all 20 retail-choice states, some by more than 5×
  (Pennsylvania industrial read **1.46¢/kWh** against a true **7.87¢**; the District of Columbia
  1.89¢ against 10.80¢; PJM 4.11¢ against 8.11¢). EIA-861 splits a retail-choice premise into a
  "Delivery" row (wires) and an "Energy" row (supply) that carry the *same* sales volume. The price
  scripts summed Bundled + Delivery for **both** revenue and sales, charging only the wires revenue
  against the full volume. The `{"Bundled","Delivery"}` filter is *correct* in
  `region-households.py`, where its purpose is to stop one premise being counted as two customers;
  it was inherited by the price scripts, where the purpose is different. The error was asymmetric —
  industrial prices were hit far harder than residential — so it made the tax look like a larger
  percentage increase than it is, and made the "a data center still pays less than households do"
  comparison pass in three states where it actually fails. Fixing it moved the 6¢ exception set from
  10 states to 13, adding Montana, Washington and **Virginia**. Caught by checking the citations
  manifest, not by any automated test: `verify.js` confirmed the manifest matched the tool, and both
  were consistently wrong. Cross-validated after the fix against EIA's own published state averages
  (corrected method gives California 31.97¢ residential vs EIA's published ~31–32¢).
- **A ratio that was never a ratio, exposed by a newer release of its own source (2026-09-08).**
  The first behind-the-meter pass set GEM's Texas figure (40,000 MW of gas tied to data centers)
  against dcmap.us's ERCOT data center pipeline (74,894 MW) and recorded the quotient as "on-site gas
  equal to roughly **53%** of that pipeline", glossed as *about half the buildout is behind the
  meter*. Every caveat around it was correct — it was labeled context-only, and the note said in
  terms that it compared generation capacity against data center capacity — and the number was still
  wrong to present as a percentage, because a percentage invites being read as a share of something.
  GEM's August 2026 release settles it: the same arithmetic now gives **103%** on ERCOT territory, or
  **90%** against the like-for-like Texas state pipeline of 85,421 MW. A share cannot exceed 100%.
  What the quotient actually measures is capacity *matching* — roughly one MW of announced gas per MW
  of announced data center capacity, which is about what self-supply requires. **What made this hard
  to see:** at 53% the wrong reading and the right one are indistinguishable by eye, and only a
  doubling of the numerator pushed it past the boundary where the category error became visible. The
  lesson is not "add a caveat" — the caveats were there — but that a quantity should not be *formatted*
  as a percentage unless it is a part of the whole named in its denominator.
- **A definitional change inside a source, caught only by re-reading the footnotes (2026-09-08).**
  GEM's January 2026 report defined its data center category as capacity serving the facility
  "independently of the public grid" — strictly behind the meter. Its August 2026 report defines the
  same-named category as "behind-the-meter **or dedicated**". The headline number nearly doubled, from
  97,470 MW to 189,344 MW, and part of that growth is definitional rather than physical. Updating the
  figure without reading the footnote would have silently converted a behind-the-meter measure into a
  BTM-or-dedicated one, and the tool's fossil-share gap concerns *only* the behind-the-meter half — a
  dedicated on-grid plant already sits inside the grid-average share. **Refreshing a figure from a
  newer release of the same source is not a safe mechanical update**; the definition has to be re-read
  each time.
- **A zero-byte file posing as a source (2026-09-08).** `sources/13-behind-the-meter/` contained
  `gem-oil-gas-plant-tracker-jan2026.pdf` at **0 bytes**, left by a download that failed silently in
  the first pass. Nothing referenced it, and a directory listing made it look like a gathered PDF.
  Deleted. Worth a standing check: a source file of zero length is worse than a missing one, because
  the folder listing is itself used as evidence of what has been gathered.

- **A tool that contradicted its own stated design, for as long as the design existed (2026-09-08).**
  The rate design says behind-the-meter gas is taxed at the fossil rate regardless — the rule that
  stops a facility escaping the tax by building its own gas plant, and the reason Virginia's HB 30
  taxes self-supplied electricity. The arithmetic never implemented it. Dragging the procurement
  slider to 0% charged **every** kWh at the clean rate, on-site generation included, so the modeled
  loophole was exactly the one the prose claimed to have closed. Applying a grid-average fossil share
  to captive load had been described in these notes as the conservative choice; it was not a choice
  at all but an unexamined claim — that a gas turbine burns like its regional grid — and it is false
  by construction. **What made it survive:** the gap was between two things neither test nor citation
  compared, a sentence of design prose and a line of arithmetic, and both were individually correct.
  Fixed by a behind-the-meter floor, `effective fossil = btm + (1 - btm) x procured`, with a guard
  that now fails if the on-screen claim and the arithmetic ever disagree again.
- **Revenue-neutrality quietly broken by the fix, and caught by the checks (2026-09-08).** Adding the
  floor made the flat and differentiated structures raise *different* revenue at enactment, because
  the flat "matched" rate was still derived from the raw grid average while the differentiated side
  now taxed captive load at the fossil rate. Four separate checks failed together and pointed at it.
  The fix routes both through one `matchedCentsFor()`, so the rate printed in the button and the rate
  used in the arithmetic cannot diverge. **The lesson is about what a revenue-neutral comparison
  means:** "the same rate at today's grid mix" stops being the right definition the moment the two
  structures disagree about what today's mix *is*.
- **An invariant that was right to break, and worth asserting rather than silencing (2026-09-08).**
  The floor also broke `k = 1 is exactly revenue-neutral`. That identity holds only because a
  facility's classification moves with the grid — and a captive gas plant is precisely one that does
  not decarbonize when the grid does. So grid-mix indexing **cannot neutralise for self-supplied
  load**, which is a real limitation of the indexing design rather than a bug in the floor. The
  failing check was not deleted: it was narrowed to grid-supplied load, its complement was added
  asserting that k = 1 is *not* neutral once on-site gas is counted, and the on-screen prose was
  qualified to match.
- **A wrong intermediate in a hand-check comment (2026-09-08).** `verify-by-state.js` documented
  Virginia's hand calculation as 232,315,440,000 kWh. The correct product of 29,460 MW x 1,000 x
  8,760 x 0.9 is 232,262,640,000, and the tool had always computed the latter. Every *asserted* value
  was correct, so no test could see it. Found only by recomputing the derivation from scratch while
  updating it. A wrong intermediate in a hand derivation is worse than no derivation: it invites the
  next reader either to distrust the right answer or to "correct" it to match.
- **The same class of bug, three more times, all found by reading the page (2026-09-08).** Adding the
  behind-the-meter floor silently falsified three separate claims the tools make on screen, and the
  arithmetic checks passed throughout because each was a *sentence*, not a number:
  1. *"k = 1 is exactly revenue-neutral"* — true only for grid-drawn load. Survived in the grid-mix
     hint and both k labels after the first fix, because the guard only checked that a qualification
     existed **somewhere**. The label now routes through one `kNeutralLabel()`.
  2. *"k = 1 raises exactly the same revenue as the flat rate — identically, at every grid mix"* —
     the largest divergence of the three. True only at a region's **measured** mix, which is where the
     matched rate is defined. At a fully clean grid the differentiated structure raises **21.2% more**
     in ERCOT and 21.9% more in Texas, because the captive share does not clean up with the grid.
  3. *"Not derivable"* shown for **24 states where GEM records no on-site gas at all** — a measured
     nil reported as a dataset contradiction, the exact opposite of the truth. It happened because
     those states carried an explanatory *comment* instead of a measured zero, and the renderer could
     not tell "nil" from "no usable value". No arithmetic check could see it: a missing share and a
     zero share compute identically.

  **The pattern worth naming:** every one lived in the gap between a sentence and a number, and the
  suite tests numbers. Each is now pinned by a guard that asserts the *absence* of the unqualified
  form as well as the presence of the qualified one, and the quoted 21% is recomputed against a
  closed form rather than trusted.
- **A guard that could not fail, twice, in the same check (2026-09-08).** The first k=1 guard passed
  while three unqualified claims stood, because it only tested that a qualification existed. Rewritten
  to test for the absence of each unqualified form, it *still* passed a deliberately reintroduced
  label — it searched for a literal em dash while the file carried the `—` escape, the same
  mixed-encoding trap that has bitten this project repeatedly, now inside the check meant to catch it.
  Both were found by mutation-testing the guard, not by a green run. **A guard written the same day as
  the bug is written by someone with the same blind spot; the only way to know it works is to break
  the thing on purpose and watch it fail.**

- **A listed source that was never gathered, and turned out to be half-unavailable and half-valuable
  (2026-09-08).** EIA's RECS sat unticked on the source checklist under household burden. Asked why,
  the honest answer was that nobody had reached it. Checking properly split it cleanly in two. The
  *Consumption & Expenditures* tables — the spending half any burden ratio needs — are not published
  for the 2024 cycle at all: EIA states they arrive in **spring 2027**. And the published half is cut
  by census region and division with **no state detail**, so it could never have fed the apportionment
  both tools are built on. DOE LEAD 2022, at census-tract resolution, was and remains the right
  choice. But the published half contains table **HC11**, which is the thing the tools were missing:
  the burden thresholds are a *ratio*, and nothing said what that ratio means in behavior. It does —
  32.9% of US households report energy insecurity, 24.8% have gone without food or medicine to pay an
  energy bill, and the incidence runs from 63.7% below $10,000 of income to 7.5% above $200,000.
  **The lesson is about how a gap gets closed, not that one existed:** "we didn't use it" and "it
  wouldn't have worked" are different claims, and only one of them was true here — of a source that
  turned out to be worth adding for a different reason than it was originally listed for.

- **A comparison between one month and itself, on a quarter of the tool (2026-09-08).**
  `monthlyCoverage()` finds the best and worst month by seeding both indices at January and moving
  them only on a strict `<` or `>`. Where coverage is FLAT across the year neither index ever moves,
  so the tool printed *"Coverage in the hardest month vs the easiest — 100% (January) · 100%
  (January)"* and, beneath it, *"In January only 100% are covered; in January, 100%."* The
  arithmetic was correct throughout: the coverage genuinely is flat, saturated at 100% or stuck at
  0%. **It affected 13 of 51 states and 2 of 24 regions at their mid scenario — including both
  known-answer states, New Mexico and Virginia** — and no check saw it, because every number was
  right and only the sentence around them was wrong. Found by reading the page. Both cases now say
  what is true: the dividend clears the median bill in every month including the peak, or reaches it
  in none including the cheapest. The guard added for it asserts the branch is **reachable** as well
  as present, so it cannot quietly become dead code.
- **A guard that searched the rendered text instead of the source (2026-09-08).** That same guard
  failed on its first run, reporting the branch missing seconds after it was added. It looked for
  "the swing does not change who is covered here" — which exists only *after* string concatenation;
  the source splits it across two literals. Same family as the em-dash-versus-`\u2014` trap earlier
  the same day: **a guard has to match the file, not the page.** Both were caught only by running
  the guard against a deliberate mutation rather than trusting it to pass.
- **A link that did nothing, and the fix that overshot (2026-09-08, reported by the project owner).**
  The pinned-source panel offers *"See the full manifest row ↓"*. It is a plain `href="#cit-<id>"`,
  and its target sits inside the citations `<details>`, which ships **closed** — a browser will not
  reveal or scroll to a fragment inside a collapsed `<details>`, so the click was a no-op, measured
  at **0 pixels of scroll**. 91 of the tool's 93 in-page links point into that section; the `[n]`
  superscripts were unaffected only because their own handler calls `preventDefault()` and pins the
  source instead of navigating.

  **Why no check caught it:** the link existed, the target existed, the ids matched, and every
  citation assertion passed. The manifest was verifiably complete and correct — and unreachable by
  the one control offered for reaching it. *Nothing in the suite tested whether an interaction did
  anything*, because everything it tested was about content.

  **The first fix was wrong in an instructive way.** Opening the ancestor `<details>` and calling
  `scrollIntoView` in the same tick measures the target against the **pre-expansion** layout, then
  the newly-inserted manifest — over ten thousand pixels of table — pushes it away. The page scrolled
  about **1,100px past** the row: visibly working, landing nowhere useful. Deferring the scroll two
  animation frames fixes it. Both failure modes now have their own guard, and both guards were
  mutation-tested by reintroducing the bug.

- **The suite tested numbers; the bugs were in sentences (2026-09-08).** Four defects in one day —
  a month compared with itself, a share offered as a change from itself, a measured nil reported as a
  data contradiction, and a link that did nothing — all passed a 278-check suite and were all found by
  eye. They share a shape: **the arithmetic was right and the rendered output was wrong**, and nothing
  ran the renderer. `render-check.js` now does, against a stub DOM, across every region x tier x
  behind-the-meter combination, with each of those four bugs written up as a regression check.

  Two things about building it are worth keeping. **JXA owns the identifier `$`** — the tool's own
  `$()` element helper resolves to the Objective-C bridge instead, globally *and* inside a function
  scope when the code arrives via `eval`, so the harness rewrites `$(` in the eval'd copy. That is the
  reason `verify.js` was only ever able to eval the calculation core. And **the first version of one
  check was circular**: it compared the rendered words against the tool's own data fields, so a
  mutation that deleted those fields flipped the expectation with the output and the check passed
  while the bug was live. It was only caught by deliberately reintroducing the bug. Grounding it in
  the derivation CSV fixed it. All four regression checks were then verified the same way — by
  reverting each fix and confirming a failure — because **a regression harness that cannot detect the
  regressions it was built for is worse than none: it certifies.**

- **The same jump link, wrong a second time, and only visible deeper down (2026-09-08).** After the
  fix that made "See the full manifest row" work at all, it worked *for early rows*. Row 215 of 250
  undershot by **20,780 pixels** — the link fired, the section opened, the page moved, and the reader
  landed nowhere near the row. The first verification had used row 1 and row 39 and looked fine.

  The cause was not the smooth-scroll animation, which was the obvious suspect: an instant scroll
  after two animation frames still missed by 939px. Opening the manifest inserts a table whose row
  heights resolve **progressively** as long locator text wraps, so any single scroll — however
  deferred — measures a layout still settling, and the error compounds with depth. There is no fixed
  number of frames that is enough.

  Fixed by scrolling, re-measuring, and correcting until the row is actually centered, bounded at six
  attempts so a row taller than the viewport cannot spin. Instant rather than smooth: an animation
  across a fifty-thousand-pixel document is racing the same reflow that invalidates it.

  **The lesson is about how the first fix was verified.** It was tested on the row that happened to be
  first on screen, which was near the top of the table. A fix for a scrolling bug has to be tested at
  the far end of the thing being scrolled — and the guard was updated to assert convergence rather
  than the particular frame-deferral shape it originally had, because that shape was the wrong fix.

- **An affordance that only worked once (2026-09-08).** Presenter view collapses every explanation to
  a one-line stub with a `why ▾` control. Clicking it expanded the paragraph and left no way back:
  the handler only *added* the open class, and the open state then set `cursor:default` and blanked
  its own `::after`, deleting the very control that had just been used. The only escape was leaving
  presenter view entirely and re-entering it.

  Fixed by toggling rather than adding, and by keeping a `hide ▴` affordance and a pointer cursor on
  the open state. The first repair introduced a second bug of its own: it suppressed the collapse
  whenever `getSelection()` was non-empty, to stop a text selection from folding the paragraph shut
  at the end of the drag. But a selection made anywhere else on the page also reads as non-empty, so
  the collapse stayed dead — the guard was measuring the wrong thing. Replaced with pointer travel
  between `mousedown` and `click`, which is the thing actually being distinguished.

  **The lesson is that a reversible control needs its reverse tested, not just its forward action.**
  The original check confirmed the paragraph expanded and that citations still worked inside it. It
  never clicked twice. Four mutation tests now cover it: handler-only-adds, blanked affordance,
  non-clickable open state, and a removed drag guard.

- **Twelve results shown before the result (2026-09-08).** The three comparison panels — flat vs
  differentiated, CPI indexing, and the k-index — sat inline in Sections 3 and 4, next to the
  controls they belong to. That was the wrong instinct: between them they printed twelve monthly
  dividend figures **above** the results section, so the first payout a reader ever saw was a
  comparison between two hypothetical versions of a number the tool had not yet stated once. There
  was nothing to anchor either side against, which is precisely why the flat/differentiated choice
  read as confusing rather than informative.

  Fixed by moving the three panels below *What a household receives*, into a section titled *What
  each choice does to this payout*. The controls stayed where they are — they are inputs and belong
  with the other inputs; only their consequences moved. No arithmetic changed: the renderer fills
  these by element id, so the known-answer tables reproduced to the cent.

  A detail worth recording, because it is the recurring trap in this project: four directional
  markers had to flip (`↓` → `↑`) now that the controls sit above their consequences, but two
  neighbouring phrases — "or above 1" and "k above 1 was set deliberately" — are **numeric**
  comparisons about the value of k, not page directions, and had to survive untouched. A blind
  find-and-replace on "above" would have corrupted the meaning of the k documentation, in the same
  way an earlier prefix sweep turned `analysis` into `analyzis`.

- **The baseline stated only after the policy (2026-09-08).** The tool opened with a region picker
  and went straight into buildout, tier and structure. Nowhere did it say, up front, how many
  households live in the territory, how many of them are energy-burdened, or what electricity costs
  there today — the burden panel was buried inside *4 · Assumptions you can change*, carrying the tag
  "Measured — not an assumption you set", which made it the one thing in that section that was not an
  assumption. The two prices existed only as a row inside the results.

  So the reader was asked to choose a fee tier before being told what problem the fee addresses or
  what the current prices are. Fixed with a card, *The situation today*, placed directly after the
  region picker: household count, the two burden shares with their counts, the residential price, the
  large-customer price, and the multiple between them.

  Every figure in it reuses an existing citation — `<region>-hh`, `burden-<region>`, `thresholds`,
  `residential-price`, `industrial-price` — so the change added no new unbacked number and
  `CITATIONS.md` was untouched at 144 rows. The multiple is computed live from the two cited prices
  rather than stored, which is why it needs no row of its own.

  Two disciplines were applied while writing it. First, the panel refuses rather than estimates: a
  territory with no derived price says the comparison is not shown, instead of falling back to a
  national average. Second, a draft sentence claiming the multiple "does not depend on which year is
  read" was removed before shipping — that is a claim about EIA filings this project has not read.
  What the sources support is narrower and is what the panel now says: both figures come from the
  same filing, the same region mapping and the same class-block ordering, so the comparison is
  like-for-like.

  The guard is grounded in `region-residential-price.csv` and `region-industrial-price.csv` (and
  `state-pilot.csv` for the by-state tool), not in the tool's own region fields — the same lesson as
  the nil-versus-undecidable check, which passed while the bug was live because it read its
  expectation out of the thing it was checking. Writing it surfaced a real parsing bug: the state
  pilot CSV carries quoted fields containing commas, so a naive split gave 34 fields against a
  23-column header and every lookup silently missed. The check now uses a quote-aware splitter and
  treats a field-count mismatch as a failure rather than a skipped row. Mutation-tested down to a
  0.03¢ drift.

  Separately verified, because the card's whole claim is that it is measured: twelve controls —
  tier, structure, behind-the-meter, buildout, fossil share, grid mix, k, load factor, take-up, years
  and CPI — were exercised in sequence. The headline dividend moved; all six figures in the baseline
  card were unchanged by every one.

- **Traceable but not answerable (2026-09-09).** Every figure in both tools had a manifest row naming its
  source and quoting its locator, which satisfied the sourcing rule and still left the tool unable to answer
  the question people actually ask: *where does that number come from, and what is it doing in the model?*
  The manifest was one flat table, 1,939px wide, sorted by nothing, listing 144 rows (250 by state) of
  provenance with no statement of function. A reader challenging a large figure could confirm it was cited
  without learning how it produced the figure.

  Two sections replace it. **Sources** groups every citation by what it does — the fee, the load, the fossil
  classification, the households, today's prices, the reserve fund — and gives each one a "How it's used"
  line. **Methodology** restates the arithmetic in order, with the currently selected territory's numbers
  substituted into every step, so a challenge to a payout can be walked from megawatts to dollars per month
  on screen.

  Three decisions are worth recording. First, the usage lines are keyed by **family**: the 51 per-state
  household counts share one sentence through `citFamily()`, because it is the same sentence, and printing it
  51 times would bury the 46 entries that genuinely differ. That reduced 394 rows to 49 statements, which is
  the difference between something maintainable and something that rots. Second, the methodology section
  **reads the result object and never recomputes** — a second implementation of the model would be free to
  disagree with the first, and this project has been bitten by exactly that class of bug more than once. A
  guard fails the build if it calls `annualKWh` or `computeScenario`. Third, presenter view now defaults
  **on**: the tool is shown to rooms, and the prose belongs one click away rather than on screen.

  The completeness guarantee is enforced rather than asserted. Both verifiers fail if any citation resolves
  to no usage entry, if any usage entry matches no citation, or if two groups claim the same citation. That
  last one caught a real inconsistency immediately: `offterritory` and `pjm-territory` describe how an
  RTO territory is carved out of whole states, and the by-state tool — which uses no crosswalk at all — was
  carrying both. They are now removed there.

  One guard was written vacuously first. It tested `/renderMethodology\(res, r\);/` against the file, which
  matches the call even when the call is commented out — so the mutation that disabled the whole section
  passed. It is now anchored to a whole line, and both the commented-out and deleted mutations fail it. That
  is the third time in this project a guard has been written that could not fail; mutation-testing every new
  check is not optional here.

- **Three pixels (2026-09-09).** The baseline card's third row printed "1.60× what a large customer pays" in
  the value column, which is the narrow one. At 375px it measured 247px and pushed the page 3px sideways —
  the tool's own checklist forbids horizontal overflow on a phone. The words moved into the label, which
  wraps; the value is now just the multiple. All 51 states are clean at 375px. Worth logging because the
  fault was introduced by a change that was verified only at desktop width.

- **One array, three renderings (2026-09-09).** The "How it's used" statements were initially written for
  the tool's Sources section only, which left the markdown manifest and the citation pins still answering
  half the question. `SOURCE_GROUPS` now feeds all three: the in-tool Sources section, the pin that opens
  when a reader clicks `[n]`, and a generated section in `files/CITATIONS.md` placed before the row-level
  table. The pin matters most — it is where a challenge actually gets answered, next to the number rather
  than at the foot of the page — and it now leads with what the figure does before saying where it came
  from.

  The markdown deliberately does **not** gain an eighth table column. Repeating a family's sentence into
  51 consecutive rows would widen an already seven-column table to say the same thing over and over; the
  usage section states each of the 49 once, with a row count beside the shared ones.

  Wiring this exposed a genuine inconsistency between the gates, which is what they are for.
  `sync-citations.js check` reported the manifest in sync while `verify.js` reported drift, because an
  earlier edit script had aborted partway and left `verify.js` calling the renderer with the old
  three-argument signature. The two were reconstructing different documents. They agree again, and the
  disagreement was the only reason it was caught.

- **The reserve fund, removed (2026-09-09).** At the project owner's request the reserve fund is gone from
  both tools: the two sliders, the projection panel, the compounding arithmetic, the 5% percent-of-market-value
  draw, and its two citations with them. It never entered the dividend — the payment
  is `localRevenue / weightedEligible`, and the fund sat downstream of that — so no result moved. Verified by
  the known-answer tables reproducing to the cent and the PJM headline holding at $87.52.

  What survives is the accounting that was actually load-bearing: below 100% take-up some revenue is raised
  and not paid out, and the tool still reports it, now as **Raised but not paid out**. The claim attached to
  it is narrower than before and better supported — unclaimed money is not shared among the households that
  do claim, which is why the per-household figure does not move with take-up. Where the money then goes is a
  policy choice, and the tool no longer asserts one. `NEEDS SOURCE` accordingly falls from 5 to 4; the count
  is generated from the `CITATIONS` array, so the tool's own header sentence updated itself.

- **"Distance closed to that threshold" (2026-09-09).** Relief was reported as a normalized fraction of a
  gap — "37% of the way to the 6% threshold". Reading it required holding three numbers at once (where the
  household is, where the line is, where it lands) and then decoding a ratio between them. The same fact
  now reads: **18.3% of income today, 13.8% after, down 4.5 points, 7.8 short of the 6% line.** Nothing
  changed in the arithmetic; `gapClosedThreshold` is still computed. Only the sentence changed, in all four
  places it appeared — the headline, the two rows in the burden panel, and the one-sentence summary.

  Overshoot was already handled honestly (the old code refused to print "114% closed"), and the replacement
  keeps that: it says where the household actually lands rather than reporting a percentage above 100.

- **2,653,416 of 2,653,415 (2026-09-09).** Found while exercising the rewritten wording at maximum buildout,
  which is the only place it shows. The count of households crossing below a burden line is accumulated over
  income groups; the base printed beside it is the profile's own stated tier total. The two are rounded
  separately, so at full relief — when every group crosses — the numerator came out one household larger than
  the population it was drawn from.

  Clamped to its base, and guarded across every region and tier at maximum buildout. Worth recording for two
  reasons: it was invisible at default settings and only appeared at an extreme, and the fix is a display
  clamp rather than a correction to either number, because neither is wrong — they are rounded from different
  places and the tool was asking them to agree exactly.

- **The gap panel answered its question fourth (2026-09-09).** *Does it close the gap?* opened with a
  five-sentence warning box: the electricity share of home energy, the mean electricity bill against mean
  total energy, the credit-then-check delivery mechanism, and a pointer to a bar chart. The answer to the
  question in the heading — is the bill cleared, and what is left — had to be read off that chart. And the
  panel covered only the **severe** tier, so the same question about moderately burdened households could
  not be answered from it at all.

  Reordered to lead with the verdict, for both tiers, in the three states the arithmetic actually produces:
  the dividend does not reach the electricity bill; it clears electricity but not heating; or it clears
  everything. Each tier then gets its bar, and an explicit **left over as a cash payment** row — stated in
  every case including the ones where it is nothing, because "how much do they actually keep" is a separate
  question from "is the bill cleared" and is asked as often.

  The explanatory box became a one-line note below the bars, and the detail it carried moved to Methodology
  step 7, *How the payment lands on a bill*, which now shows the split with live numbers. A per-bar caption
  was deleted outright: once the verdict leads, all three of its branches restated it, and its one
  non-duplicative point — that past full coverage the policy is an income transfer rather than energy
  assistance — is in step 7.

  No new citations. The moderate-tier breakdown is the same derivation as the severe one through
  `rebateBreakdown(regionKey, tier, monthly)`, which already took a tier; the manifests stayed at 142 and
  248 rows.

  Guarded on both halves of the fault: that both tiers get an answer, a bar and a cash figure, and that the
  verdict precedes the explanatory note. The second guard exists because the ordering is the thing that
  regressed, and a check that only counted the pieces would pass with the explanation back in front.

  One consequence worth naming, since it changes what a room sees first: at PJM's defaults the honest answer
  is "No" for both tiers, and short by $36.24 and $77.44. That was previously softened by four sentences of
  context before the reader reached it.

- **Four rows cut from the gap panel (2026-09-09).** The 6% threshold row, *Does the dividend get them
  there?*, the unburdened-household average, and *Does it reach that — a harder target?* were removed at the
  project owner's request. They had been reworded earlier the same day, away from the normalized
  "distance closed to that threshold" phrasing; the rewording made them readable but did not make them
  necessary, and the panel was carrying the same idea three times over — the before/after burden figures
  above them, these four rows, and the two "dividend that would bring this tier to…" rows below.

  What remains says it once: burden before and after, how many households cross each line, and what
  dividend each target would actually require. The panel drops from 606px to 494px.

  `gapClosedThreshold` and `gapClosed` are still **computed** and still asserted by both verifiers — they
  remain null-guards in `burdenLine` and the arithmetic is unchanged. Only the display went. The
  "Two benchmarks" note was re-pointed: it described "the headline figure" and "the second", which were the
  removed rows, and now explains the two targets the surviving rows price.

- **One number per tier (2026-09-09).** The gap panel was cut to a single headline figure for each burden
  tier: **the share of the electricity bill the dividend covers** — PJM severe 71%, moderate 36% — with the
  existing stacked bar as its picture, the cash-left-over row, one crossing count, and the before/after
  energy burden. Everything else moved to Methodology step 7. The panel is 396px, down from 606px this
  morning.

  The framing decision is worth recording because it constrains what the number may claim. Energy burden is
  defined on **total home energy** (ACEEE/DOE), so there is no measured "electricity-only burden gap". Two
  things could have been meant by one: a share of the burden gap closed — the normalized ratio deleted
  earlier today for needing three numbers held at once — or the share of the electricity bill covered, which
  is a real, directly interpretable quantity about the part of the problem this fee actually touches. The
  second was chosen, and the panel now states the limit plainly under both blocks: **covering the
  electricity bill is not the same as ending energy burden**, because burden counts heating too. ISO-NE
  shows why that sentence has to be there — 17% of the electricity bill covered, burden barely moving from
  17.4% to 16.2%.

  A second decision, from the data rather than from taste: the crossing reported differs by tier. Moderate
  households sit at 6–10% by definition, so counting how many fall below 10% returns the entire tier every
  time (PJM: 2,562,584 of 2,562,584) and can never move. Moderate therefore reports **below 6%** — out of
  burden altogether, 936,721 — while severe reports below 10%. A guard asserts the split, because the
  always-total version looks like a spectacular result rather than a broken one.

- **Stale prose contradicting the model, found in passing (2026-09-09).** The note removed from the gap panel
  ended: *"The model accounts for that: relief stops at the electricity bill and the surplus is counted as
  income instead."* That describes the over-correction of 2026-09-07, which was **reverted the same day** in
  favour of offsetting against total home energy and showing the breakdown. The sentence had survived the
  revert and had been telling readers the opposite of what the code does for two days. It is gone with the
  rest of the note, and the replacement states the actual behavior.

  This is the second time prose has outlived the arithmetic it described. Both times the number was right and
  the sentence around it was wrong, which is the exact failure `render-check.js` exists for — and neither
  instance was in a place it checks.

- **Two phone-layout faults, both mine, both found by checking the other tool (2026-09-09).** Neither
  appeared at desktop width, and both were introduced by changes I had verified only at desktop width.

  **150px of horizontal overflow, on every page of both tools.** `.jump` carries `white-space:nowrap`,
  which is right for the short inline links it was written for ("how ↓", "k = 2.00 ↑") and wrong for the
  section-source links added earlier today, whose text is a full group name. At 375px the anchor
  "What households and data centers pay today →" measured 295px and pushed the document to 525px wide.
  `.srclinks a` now overrides the nowrap. The tell was that the overflow was *identical* across all 51
  states — uniform breakage is a layout fault, not a data one.

  **Every label squeezed to one word per line.** `.mv` is also `white-space:nowrap`, so on a narrow screen
  the value claims its full width and what remains goes to the label: "…of which, left over as a cash
  payment" wrapped onto six lines of one or two words. This one was latent before today and the rewritten
  gap panel exposed it, because the panel's new labels and values are both long. Below 560px the rows now
  stack — label, then value beneath it. Desktop is untouched: rows stay side by side and the panel stays
  at 396px.

  Both are guarded, and the second guard is worth having beyond this fix: it covers every `.mrow` in both
  tools, not just the panel that happened to reveal the problem.

- **Full phone audit, and the ceiling on a tap target (2026-09-09).** Both tools were swept at 375x812 in
  five interaction states — default, presenter view off, every `<details>` open, a citation pinned, and the
  flat rate structure selected — across all 24 regions and all 51 states. Zero horizontal overflow in every
  combination, including with the 248-row manifest and the methodology expanded. Charts fit (309px in 375),
  sliders run full width, segmented buttons stack at 59px tall, and all nine methodology formula blocks
  scroll inside their own containers rather than pushing the page.

  The one real finding was tap targets: citation markers measured **22x11px** against a 44x44 guideline,
  with adjacent markers 3.6px apart. They are the tool's primary provenance affordance, so this mattered.

  The fix has a hard ceiling, and finding it took two wrong attempts worth recording. First the override was
  written into the earlier of the two `@media(max-width:560px)` blocks — which is declared *before* the base
  `.cit` rule, so it lost on source order and did nothing. Moved after it, 9px of vertical padding grew the
  marker to 30-32px against line boxes as short as 18px; the padding then spilled over neighbouring content
  and **lost the hit test to later-painted siblings** — a headline, a slider label, a button. The target was
  bigger and less tappable than before. Raising it with `z-index` would have put invisible padding over real
  buttons, trading one mis-tap for a worse one.

  Bounded by the line box instead: **22x11 → 29x16**, roughly double the area, no overlap with any
  neighbour, and every visible marker hit-testable at its center and both edges (22 of 22 probed, in both
  presenter modes). An inline superscript cannot reach 44x44 without leaving its line, and that is the
  honest limit. Guarded as a number rather than an exact string, so the padding can be tuned but not pushed
  past the line again.

  A note on measurement, because it wasted a pass: the first probes reported 21% of taps landing, which
  looked like a serious defect. They were counting markers inside collapsed `.hint` blocks — laid out and
  returning a bounding box, but clipped away by `overflow:hidden` and not on screen. Any hit-test probe over
  this tool has to exclude content clipped by an ancestor, not just content outside the viewport.

- **Two counts of the same households, undisclosed (2026-09-09).** Raised by the project owner, who spotted
  that the bill panel said "100% of 101,213" while the top of the page said 105,589 severely burdened New
  Mexico households. Both figures are correct; nothing was miscomputed; the page simply never said they
  count different populations.

  DOE LEAD can only compute an energy burden where a household reports usable income. For New Mexico it
  excludes 33,780 of 812,852 households (4.2%) — 31,947 reporting zero income and 1,833 numerically
  degenerate — leaving 779,072 usable, of which **101,213 are severely burdened**. That is LEAD's own count,
  and it is the number the bill distribution is built from. The tool's headline instead takes the resulting
  share (12.9915%) and applies it to *all* 812,852 households, giving 105,589 — which assumes the excluded
  households are burdened at the same rate as the rest.

  Verified across every state with histogram data: the share matches to four decimal places in all 50, so
  the divergence is entirely in the denominator and never in the rate. The gap runs 1.8% to 6.3%, largest
  in Alaska.

  Neither number is wrong and the scaling assumption is the ordinary way to apply a rate, so the fix is
  disclosure rather than a change: the bill row is labeled "DOE LEAD's own count" and the note names both
  figures, states the assumption, and says which is more honest depends on it. `lead-burden-shares.csv`
  already carried `total_households`, `usable_households` and `severe_over_10pct` as separate columns — the
  distinction was in the sources all along and only the presentation had collapsed it.

  Guarded, and the guard was rewritten once during this fix: the first version pinned the ORDER the two
  counts appeared in, and broke the moment the sentence was rephrased. It now checks that the note names
  both counts and the assumption, in any order.

- **The severe count written to the CSVs (2026-09-09).** `region-households.py` apportioned the real severe
  and moderate household counts alongside the totals and then discarded them, writing only the shares. Both
  derivation files now carry all three denominators:

  | column | NM | what it is |
  |---|---|---|
  | `households` | 812,852 | every household |
  | `usable_households` | 779,072 | those LEAD can compute a burden for |
  | `severe_households` | 101,213 | LEAD's own severe count |

  `share x households` deliberately does **not** reproduce `severe_households` — that gap is the modeling
  choice in §3.2, and keeping both denominators in the file is what lets a reader see it rather than infer
  it. Pre-existing columns are byte-identical in both files: 24 region rows and 51 state rows compared
  column by column, zero changes.

  Two traps hit while doing it, both worth recording:

  **`state-pilot.py` defaults to a two-state pilot.** `states = sys.argv[1:] or ["NM","VA"]`, while the
  committed CSV holds all 51. Running the script the obvious way silently truncated a 51-row artifact to 2
  and reported success. Restored from a backup taken before the run — which is the only reason it was a
  non-event. Any re-derivation of that file must pass all 51 codes explicitly.

  **zsh does not word-split unquoted variables.** Passing the state list as `$ALL` handed the script one
  51-code argument; it warned "no LEAD row for NM VA ND NE…" and wrote nothing. The warning is what caught
  it. Use `$(cat file)` or an explicit array.

  `MASTER-DATA.csv` was regenerated in the same pass and moved 158 rows — all of them citation indices
  shifting after the reserve fund removed two entries, with **zero value changes**. It had been stale since
  that removal, which nothing had flagged.

- **"Households in X" read as the eligible count (2026-09-09).** The baseline card's household figure is
  the whole population; the two rows beneath it are the eligible subsets. The project owner asked which it
  was, which is the answer: the label was ambiguous. Now "All households in X (the whole population, not the
  eligible ones)".

  The guard written for it failed against a correct tool, twice over: it searched for `'All households in '`
  with a leading quote, but the phrase sits mid-literal — the string opens at `'<div class="mrow">`. The
  guard was wrong, not the label. Worth remembering that a string-matching guard over generated markup has
  to match where the literal actually begins.

- **MASTER-DATA.csv gains a drift gate (2026-09-09).** It is generated from the tool but was the one
  generated artifact with no `check` mode. `verify.js` compared its *values* against the tool, which is
  why removing two citations could shift 158 rows' citation numbers and go unnoticed for a day: no value
  had changed, only the numbering. Value-level checks do not catch renumbering.

  `build-master-spreadsheet.py check` now regenerates the file in memory and compares the whole thing,
  exiting non-zero and naming the first differing row without writing anything. It joins the documented
  gate sequence in CLAUDE.md, which is now five commands rather than four.

  Two things this exposed:

  **The file is CRLF** — `csv.writer`'s default, 421 line endings. Any hand-edit made in Python text mode
  rewrites every one of them to LF, and the gate correctly reports the whole file as drifted. That looked
  like a flaky check for several minutes; it was the check being right about an edit I had not intended
  to make.

  **A gate that repairs what it measures is not a gate.** The first version of the guard asserted only
  that *a* `return` appeared before the write. Deleting the check branch's own return still left the
  FileNotFoundError one inside the window, so the guard passed while `check` fell through to the write —
  exiting 0 and silently fixing the drift it should have reported. Demonstrated rather than assumed:
  with the return removed, introducing a value change and running `check` returned exit 0 and left the
  file repaired. The guard now asserts the write is *immediately* preceded by the check branch's return.

- **Three citations wearing the wrong usage text (2026-09-09).** Found by clicking `[62]` in the browser
  during a routine check: `burden-method` — a note about the *limitations* of the burden shares — was
  showing "Multiplied by the household count to give how many households qualify at each tier", which is
  the per-region burden-share line.

  `citFamily()` collapsed per-jurisdiction citations onto one shared usage entry by pattern:
  `/^burden-[a-z]{2,12}$/`. "method" is six lowercase letters, so `burden-method` matched. So did
  `burden-profile`, and `bill-dist` matched the bill pattern. Three citations inherited a sentence
  describing a different figure.

  The coverage guard passed throughout, and that is the part worth recording. It asserted every citation
  resolves to *an* entry and every entry is used by *a* citation — both true. Neither says the resolution
  is **correct**. A membership check cannot detect a wrong membership.

  Fixed by requiring the word beside the dash to be a real key in `REGIONS`, so a family member has to
  name a jurisdiction the tool actually has, and by giving the three their own entries. The new guard
  checks exactly the thing the old one could not: for every citation that resolves to a family, the
  jurisdiction it names must exist. Mutation-tested by restoring the pattern-only matcher, which now
  fails with "WRONG USAGE TEXT ON: burden-method, burden-profile, bill-dist".

  One knock-on: `citFamily()` now consults `REGIONS`, so `sync-citations.js` and its by-state twin had to
  eval the `REGIONS` block alongside `CITATIONS` and `SOURCE_GROUPS` — otherwise the generator and the
  tool would classify the same id differently and the manifests would drift by construction.

- **Audit of all 49 usage entries (2026-09-09).** After the three pattern-swallowed citations were fixed,
  every distinct entry was read against its citation's own `what` field and against where the tool actually
  renders it. Three more were wrong — none of them detectable by any mechanical check, because each
  resolved to a valid entry in a real group:

  **`offterritory` was filed under "What counts as fossil".** It measures how far the *buildout*
  apportionment is stretched — the share of a territory's pipeline drawn from states it barely serves —
  is derived in `dcmap-buildout.py`, and is cited beside the buildout caveats for TVA and Bonneville. It
  has nothing to do with fossil classification. Moved to "How much electricity is charged".

  **`sb253` was filed under "The fee itself"**, beside `va-enacted`. But `va-enacted` is a fee benchmark
  (the tiers are anchored to it) while SB 253 is a *residential relief payment* rendered as a reference
  line against the dividend — "Virginia SB 253 relief, $5.52/mo". Different axis. Moved to the
  comparisons group, which was renamed "The national floor, and what the dividend is measured against"
  to describe what it now holds.

  **`kwpermw` described the wrong conversion order.** It said "converts megawatt-hours into kilowatt-hours";
  the calc core and the methodology's own printed formula both do `MW x 1,000 kW/MW x 8,760 h`, converting
  megawatts to kilowatts first. Arithmetically identical, but it did not describe the formula on screen,
  and it omitted that the same constant documents megawatts per gigawatt for the buildout slider.

  All seven groups now read as coherent sets. Recording the limit honestly: this class of error —
  a citation filed under a plausible but wrong heading — has no guard, because "resolves to a valid entry"
  is exactly what a mechanical check can confirm and exactly what was true of all six wrong ones. It was
  found by reading the 49 pairs against their citations, and that is the only way it will be found again.

- **A link silently dropped by a rename (2026-09-09).** `SECTION_SOURCES` names source groups as plain
  strings, and `buildSectionSourceLinks()` skips any name it does not recognize. That skip is deliberate:
  it lets the shared table carry "Territory definition", which exists only in the region tool. It also
  means renaming a group removes its links with no error anywhere.

  "Where unclaimed money goes" was renamed twice in one day — first when the reserve fund was removed,
  then again when SB 253 joined the comparisons — and `SECTION_SOURCES` was never updated. *What a
  household receives* had been quietly rendering one link instead of two.

  Nothing caught it. `verify.js` did not, because the string is only compared at runtime against a list it
  has no view of. The browser check did not, because it asked whether every rendered link *resolves* — and
  the dropped link was never rendered, so there was nothing to fail. **A check over what is on the page
  cannot see what should have been on the page and is not.** The check now compares rendered links against
  the number `SECTION_SOURCES` asks for: region 12, by-state 11.

  Guarded in both verifiers: every group named must exist, with `Territory definition` the single
  documented exception. Mutation-tested on the exact scenario — rename a group, leave the reference stale
  — which now fails with the dropped name printed.

- **Verified, not a defect: the integer grid-mix slider (2026-09-09).** A sweep of every control looking
  for dead wires flagged the rate-structure toggle as having no effect, then — when tested away from
  defaults — showed flat and differentiated revenue differing by 0.05%. The tool claims they raise
  *identical* revenue at enactment, so this was worth chasing.

  The claim is exactly true. At each region's measured grid mix with the index neutral, all 24 regions
  agree to 2.23e-16 — floating-point noise, nothing more. The 0.05% came from the test, not the tool:
  the grid-mix slider is integer-stepped, so its default sits at the nearest whole percent rather than on
  the measured share. PJM measures 60.04% and the slider defaults to 60%; k = 2 amplifies that
  0.04-point offset into a 0.05% revenue difference, applied to the differentiated branch and not the
  flat one. 22 of 24 regions have a measured mix the integer slider cannot land on.

  No reader ever sees it: both structures print identically to the cent in all 24 regions and all 51
  states. Left as is — sub-integer slider steps would be churn for a difference that does not reach the
  display — but recorded so the next person who computes revenue two ways and gets 0.05% apart knows
  where it comes from.

  Also verified in the same pass, all behaving as documented rather than dead: take-up moves what is paid
  out and what goes unpaid while leaving the per-household entitlement fixed; CPI is inert at year zero
  and live at twenty years; `season-best` is absent in exactly the 13 states with flat monthly coverage
  and present in the other 38.

- **Pilot-era scope claims, surviving the expansion (2026-09-09).** Found by comparing the two tools
  against each other — the first check to ask whether they agree on the figures they share. Of 44 shared
  citation ids, 36 are scoped "All" in both; 13 differ, and all but three of those differ legitimately,
  each tool describing its own geography.

  The three that did not: the by-state tool's `industrial-price` and `residential-price` still cited
  "New Mexico and Virginia rows directly", and `bill-dist` declared its scope as "New Mexico and
  Virginia". All three date from when this was a two-state pilot — `state-pilot.py` still defaults to
  `["NM","VA"]`, which is what made the wording true when it was written. Verified against the shipped
  data before changing anything: 51 jurisdictions carry an industrial price, 51 a residential price, and
  51 a severe bill distribution. The citations understated the tool's own reach by 49 states.

  Nothing had looked at this. Every existing check asks whether a figure is *right*; none asked whether
  a citation's description of its own coverage is *still* right. A citation can be perfectly sourced and
  still lie about how much it covers.

  `burden-profile` mentions the same two states and was left alone: its locator says the ranges "were
  written when this tool covered only New Mexico and Virginia … and were recomputed", which is a
  deliberate record of a correction. The new guard therefore reads `region`, `source` and `what` but not
  `locator` — locators are where history belongs.

- **Accessibility audit, and a bug shipped through all five gates (2026-09-09).** The first pass over
  keyboard and screen-reader behavior. Much of it was already sound: 188 focusable elements, every one
  with an accessible name; all ten sliders carrying a real `<label for>`; every segmented control with
  `role="group"`, an `aria-label`, and `aria-pressed` on all buttons; `lang="en"`; a sensible heading
  outline; two live regions. Contrast passes throughout, 5.17:1 at the dimmest (collapsed presenter-view
  prose) against a 4.5:1 requirement.

  Two real defects:

  **No focus indicator.** The stylesheet contained exactly two `:focus` rules, for the region search box
  and the section-source links. The other ~185 focusable elements — every region button, tier button,
  slider, citation marker and `<summary>` — relied entirely on the browser default. Nothing set
  `outline:none`, so this was an omission rather than a suppression, but a tool meant to be presented and
  shared should not leave WCAG 2.4.7 to a UA default. An explicit `:focus-visible` ring now covers all of
  them.

  **Citation markers announced as "[1]".** 109 of them, each a link whose entire accessible name was a
  bracketed number. They now carry `aria-label="Source 1: Household count"` — the figure's own `what`
  field — so the provenance affordance is usable without sight of the page.

  The fix for that second one broke the tool completely, and that is the part worth recording. Inserting
  `CIT_BY_ID` was guarded by `if 'CIT_BY_ID' not in s` — evaluated *after* the same script had already
  written a reference to it, so the guard saw its own handiwork and skipped the definition. Every page
  load then threw `ReferenceError: CIT_BY_ID is not defined` and rendered **zero citations**.

  **All five gates passed on that build.** `verify.js` evals the calc core, which never touches `cite()`.
  `render-check.js` evals the render core, where `cite()` lives — but no check ever called it. A function
  can sit inside a block a harness evaluates and still never be executed by it. `render-check.js` now
  calls `cite()` for every citation id and for an unknown one, asserting a linked marker with an
  accessible name and a `NEEDS SOURCE` refusal respectively. Verified both ways: with the definition
  removed the harness exits 1 naming the missing variable, and with the `aria-label` removed it fails the
  new check by name.

- **Main landmark and a live region (2026-09-09).** Two gaps left open by the accessibility audit, both
  now closed. `<main id="toolMain">` wraps the eight sections between the header and the footer, so a
  screen-reader user can skip past the tool switcher and the title to the tool itself; the page now
  exposes HEADER / MAIN / FOOTER rather than HEADER / FOOTER.

  The dividend previously changed in silence. A polite, atomic `role="status"` region now carries a
  sentence naming the territory, the tier, the buildout and both figures — "PJM, moderate tier, 35.1 GW
  of data center buildout. A severely energy-burdened household receives $87.52 a month, a moderately
  burdened household $43.76."

  It is **debounced at 700ms**, and that is the load-bearing detail. `render()` runs on every `input`
  event of a slider drag, so a live region wired straight to the values would read out every intermediate
  number. Verified: a seven-step drag produced **one** announcement, not seven.

  The check found a bug in my own fix. The sentence is assigned with `textContent`, and I had wrapped the
  territory name in `esc()` — so "Florida Power & Light" became the literal string "Florida Power &amp;
  Light", which a screen reader reads out as the entity. Exactly one jurisdiction in either tool contains
  a character `esc()` would mangle, and the sweep across all 24 regions found it. HTML-escaping belongs
  with `innerHTML`, never with `textContent`.

  Six accessibility guards added to both verifiers — landmark, live region, the render-path call, the
  debounce, the no-escaping rule, the focus ring and the citation naming — each mutation-tested
  individually.

- **A third of the tab stops were invisible (2026-09-09).** Found by walking the focus order rather than
  the rendered page. Presenter view clips explanatory blocks to one line with `overflow:hidden`, but
  clipped content stays in the tab order: **66 of 188 focusable elements** — citation markers and jump
  links inside collapsed hints — sat outside their visible box. A keyboard user tabbing through the tool
  spent a third of the journey on elements they could not see, with the new focus ring drawn where it
  could not be seen either. The focus indicator added earlier the same day made this worse, not better:
  it drew a ring on content behind a clip.

  Fixed with `:focus-within` on the collapsed blocks, so a block opens the moment focus enters it and
  closes when focus leaves. Pure CSS, nothing to keep in sync, and it does not fight the existing
  click-to-toggle. Verified: all 66 clipped elements become visible on focus (0 remain clipped), the
  block still collapses to 18px at rest, the why/hide click toggle still round-trips 18 - 427 - 18, and
  the page is unchanged in height at 7,289px so presenter view still does its job.

  The alternative — removing the clipped content from the tab order — was rejected. It would have meant
  either an `inert` attribute on a block whose first line must stay clickable, or JavaScript keeping
  `tabindex="-1"` in sync with the collapsed state on 66 elements. Expanding on focus is one CSS rule.

  Worth naming the method, because the rendered page looked fine: this only shows up if you enumerate
  focusable elements and ask where each one actually sits. A visual check cannot see it, and neither can
  a check that asks whether the right things are on the page.

- **Presentation cleanup (2026-09-09).** Three changes at the project owner's request, all about what a
  room sees rather than what the tool computes.

  **The "Read this first" banner is gone.** Its flagged-figure counts moved to the footer rather than
  being deleted with it: those are a claim the tool makes about its own sourcing, they are generated from
  `CITATIONS` at load, and a guard requires them to stay generated. The narrative around them — which
  regions carry hand checks, which publish forecasts — was the part being removed.

  **Decision history removed from everything a reader sees.** Twelve passages carried dated narration:
  `CORRECTED (2026-09-03)`, `CORRECTION HISTORY:`, `found while investigating PNM`, `reverted the same
  day`, `previously cited`, `Supersedes`. Each was rewritten as a present-tense statement of what the
  figure is and what it excludes, or deleted where no fact survived the framing. Zero remain in any
  reader-facing string.

  One was kept, reworded, and the reason is worth recording: `pjm-hh` carried "Supersedes 26,400,000
  (67 million people / 2.54)". Mid-cleanup the project owner asked why PJM shows 25 million households
  when the figure they knew was 67 million — and that sentence was the answer. It now reads as fact
  rather than history: 67 million is PJM's **population**, 25,446,324 is **households**, the ratio is
  2.63 people per household against a US average near 2.5, and EIA-861's 27,894,330 residential **meters**
  exceed both because vacant and seasonal units carry meters. **Some history is the answer to a
  challenge**; the test is whether it can be restated as a fact about the number rather than a story
  about the project.

  **A way back from the manifest.** Following a citation into the manifest moves the reader 16,000-20,000px
  and the browser's Back button does not undo a scripted scroll — a presenter who checked a source had
  lost their place. A control now appears bottom-left naming the panel they left ("Back to 4 - Assumptions
  you can change") and returns to the exact scroll position. Bottom-LEFT because `#pinStack` owns
  bottom-right and a pinned source is precisely what a reader has open when they follow a manifest link.

  It names the section **on screen**, not the one containing the link: manifest links are followed from a
  floating pin card that belongs to no section, so the obvious implementation gives "Back where you were".
  Verified from four origins in both tools, returning to within 0px each time, with the manifest left open
  — returning should not undo what the reader went to see.

- **Which utilities each region is made of (2026-09-09).** A region's household count reads like a
  published statistic and is not one: it is DOE LEAD state households apportioned by EIA-861 residential
  **customer** counts, which EIA files utility by utility. Nothing on the face of the tool said so. Each
  region now carries a *Built from* line naming them — PJM "136 utilities filed under the PJM balancing
  authority. Largest by residential customers: Commonwealth Edison Co (3,727,675), Virginia Electric &
  Power Co (2,492,204)…", Georgia Power "a single utility — Georgia Power Co (2,452,488)".

  The two kinds of region needed different sentences. Seventeen are balancing authorities aggregating
  every utility EIA tags with that code, from 2 (Puget Sound) to 272 (MISO). Seven are a single named
  utility matched by name. A guard asserts a name-matched region resolves to exactly one.

  **A data-quality finding came out of it.** EIA-861 carries a reconciliation line, "Adjustment 2024",
  with a BA code and a state but no utility behind it: **3,437,809 residential customers nationally,
  2.4% of the total, across 44 states.** It was already inside the apportionment weights, unremarked.
  It is defensible there — the line appears in both the region's customers and the state total, so the
  *share* barely moves — but it is not a utility and listing it as one would be wrong. It is now excluded
  from every named list and its size disclosed where it is material: SPP 9.4% of that region's customers,
  MISO 5.0%, PJM 1.5%.

  New derivation `sources/11-bill-benchmarks/region-utilities.py`, with a `check` mode, joins the gate
  sequence — six commands now. It reuses `region-households.py`'s EIA loader and the same
  Bundled + Delivery service-type filter, so the utilities listed are exactly those whose customers carry
  the apportionment weight rather than a differently-filtered set that would merely look similar.

  Two self-inflicted faults on the way, both caught by the gates rather than by reading: the citation was
  first appended at the top of the `CITATIONS` array, renumbering all 142 existing entries, and moving it
  to the end dropped a comma and broke the array outright. Every gate failed loudly on a syntax error,
  which is the behavior to want.

- **Four quantities, three of them called "fossil" (2026-09-09).** Raised by the project owner: the
  section reads as though it is about the region's grid mix and then asks how much of "this power"
  counts as fossil, which sounds like a single facility. It was both, undifferentiated.

  Section 4 carried four numbers and no statement of what kind each one is:

  | | what it is | kind | PJM default |
  |---|---|---|---|
  | `gridFossilShare` | the grid mix EPA eGRID measured | **fact** | 60.04% |
  | `fossilShare` | what the fee is charged against | **dial** | 60% |
  | `btmShare` | on-site gas, fossil-rated regardless | **floor** | 7.3% |
  | `gridMixNow` | where the grid goes next | **scenario** | 60% |

  The dial defaults to the measured mix, so at defaults three of them read about 60% and nothing said
  which was measured, which was chosen and which was hypothetical. Nor that they combine as
  `btm + (1 - btm) x dial`, nor that the scenario feeds **only** k and never revenue.

  Fixed without touching any arithmetic. Two subheads split the axes — *This data center's power* and
  *The grid around it* — and an orientation block above the dial names all four with their current
  values, labels each as fact / dial / floor / scenario, states the combined share the fee is actually
  charged against (PJM 63.0%), and says plainly why the dial defaults to the measured mix: a facility
  procuring nothing in particular looks like the grid around it. The dial's own label now names its
  subject rather than saying "this power".

  The k explainer gained the sentence the confusion actually needed: **which two numbers k compares.**
  Only the two grid figures — the measured mix against the slider — and it does not read the facility
  dial at all.

  **On-site gas provenance is now on the face of the tool**, because it is the figure most likely to be
  challenged and its source is easy to misquote. Global Energy Monitor's Global Oil and Gas Plant
  Tracker, August 2026 unit-level release. GEM **widened the definition** between its January and August
  releases, from "behind-the-meter" to "behind-the-meter **or dedicated**", so the 189,344 MW headline is
  a superset and must not be quoted as a behind-the-meter figure — the tool says so. The per-region share
  is read from the unit-level file, where a plant GEM marks entirely captive is behind the meter; 43.8%
  of capacity has that field blank, which is why the tool publishes a low and a high bound and no point
  estimate. About 5,548 MW of the 189 GW is operating; the rest is announced, permitted or under
  construction.

  Across the 51 states: 21 have a confirmed on-site floor, 25 are a measured nil, 5 are not derivable.
  New Mexico's confirmed floor is 0.0% against an upper bound of 18.7%, so its floor row is correctly
  absent at the default setting while Virginia's 1.1% is shown with the combination spelled out.

- **The grid-index dial stops being called k (2026-09-09).** It read "k = 2.00" on screen — an exponent,
  which no reader has intuition for. It now names the **regime** it selects and anchors the strength in a
  consequence the tool already computes:

  | setting | readout | PJM at a fully clean grid |
  |---|---|---|
  | left end | households lose out as the grid cleans | $57.62/mo |
  | one third along | households held steady | $92.22/mo |
  | halfway (default) | households gain as the grid cleans | $147.59/mo |
  | right end | households gain as the grid cleans | $378.02/mo |

  Against today's $87.52. The held-steady figure landing at $92.22 rather than exactly $87.52 is the
  on-site gas floor: captive gas does not decarbonize with the grid, which is the caveat the tool already
  states and which the number now demonstrates.

  Roughly 37 on-screen mentions were reworded across label, readout, the three-position list, both
  comparison panels, the methodology formula and the usage entry. The letter survives in code
  identifiers (`gridIndexing`, `grid-indexing`) as CLAUDE.md requires, and in the citation locator,
  because a statute implementing this would have to specify a number.

  **The anchor was wrong when first written, and the check caught it.** It computed the clean-grid
  dividend by moving `gridMixNow` alone. But the grid slider's own handler also sets
  `fossilShare = gridMixNow` — a facility procuring nothing in particular follows its grid — so changing
  only the grid modeled something the reader cannot produce by dragging, and at the unindexed setting it
  reported no change at all. Verified by dragging the slider to zero at three settings and comparing
  against the readout: $57.62, $92.22, $147.59, exact at each.

  **And the natural misreading is now answered where the dial is.** Asked whether the dial makes it
  "progressively more expensive to be fossil", the honest answer is half yes: the fossil rate does rise —
  1.9¢ to about 3.4¢ if PJM's grid cleaned to 20% at the default — but **the clean rate rises by the same
  factor**, 0.95¢ to about 1.7¢. Measured across settings 0, 1, 2 and 4 at two grid mixes, the
  fossil-to-clean ratio is **exactly 2.0000 in every case**. So it is not a penalty dial and does not
  change the incentive to procure clean; it sets how much the fee collects once fewer kilowatt-hours are
  counted as fossil at all. That is now stated in a highlighted note directly under the dial rather than
  four paragraphs down.

### 2026-09-09 — Section 4: cutting the text, and two framing errors the cut exposed

**The complaint.** Section 4 put **2,132 words on screen** and the sentence that says what a slider
does was buried inside them. Each of the four controls now carries an always-visible `.lede` of two
or three sentences; everything else moved behind `why ▾` into short labeled sections (`.wsub`).
Section 4 is now **487 words visible** — 322 of prose plus the 165-word orientation table — with
1,528 behind the four notes. The ledes are deliberately NOT `.hint`: presenter view collapses
`.hint`, and the sentence that tells you what a control does must survive that.

**Errors found and fixed in the same pass.**

1. **The teaser became a heading.** A collapsed `.hint` shows its first line as a preview. Once the
   notes gained `.wsub` labels, that preview was an uppercase section title with nothing under it,
   sitting on the control. A leading `.wsub` is now hidden while collapsed.

2. **`.krow` shattered.** `.klist .krow` is a two-column grid, so *every child is a cell*. Three
   rows carried their description as bare text plus inline `<strong>`/`<em>`/`<a>`, which laid out
   as one grid row per element — the citation marker got a row of its own. Nothing computed was
   wrong; all gates passed. Descriptions are now wrapped in a single `<span>`, and a guard requires
   `<b>label</b><span>description</span>`.

3. **The variable was still named.** The existing "no k in prose" guard listed the words `k` might
   be followed by — a whitelist — and the tool said "the problem k exists to solve" and "k starts
   to bite" past it. It now scans **every reader-facing string literal**; `\bk\b` cannot match
   inside `kWh`. Six further reader-facing uses were removed.

**The framing error (raised by the project owner).** Section 4 said *"this data center's power"* and
*"the data center's own site"* throughout. Every figure in it is a **territory-wide aggregate** — one
grid mix, one on-site gas share, applied to the whole disclosed pipeline. The behind-the-meter note
had carried the caveat for a day; the headline framing contradicted it, and the summaries written
earlier the same day repeated the singular. All of Section 4 now reads as the buildout, and the
on-site gas summary states the consequence: **more of the buildout on its own gas → a larger
dividend.** That direction is checked, not asserted — `ignore ≤ confirmed ≤ upper bound` holds in
all 23 regions and 41 states with a derived share.

Two wording bugs fell out of that rewrite and were fixed before shipping:

- **"an estimated 0.0%"** where GEM *confirmed* none but left up to 31% of capacity unrecorded. A
  confirmed zero against a nonzero upper bound is a **bound, not a nil** — the same distinction
  `btm-gas.py` was written to preserve. It now names what the upper bound covers.
- **"charged against 73.1% fossil rather than 73.1%"** — the "58% rather than the 58% set above"
  bug, reintroduced. The three cases (real floor / confirmed zero / too small to move the share)
  now render differently, and a guard rejects `X% ... rather than X%`.

**The dial default (raised by the project owner, looking at Salt River Project).** *"If we know a
utility's fleet mix from eGRID, why isn't that number the default?"* **It was** — `regionDefaults()`
has always returned `r.gridFossilShare` unrounded. The **display** hid it:

- Both sliders stepped in whole percent and printed `Math.round()`, so the orientation block showed
  *"as measured 51.7%"* directly above *"counted as 52%"* for the same 51.68%. In SPP the two
  rounded in opposite directions — **53.5% above 53%**. Same number, adjacent rows, visibly
  disagreeing, which reads as an assumption rather than a measurement.
- The row also carried **"illustrative — needs source"** while displaying an eGRID measurement.

Both sliders now step in tenths and print one decimal, so the default lands exactly on the measured
value; and the row **cites eGRID while it is the measurement, the unsourced dial once moved**, with
the label changing from *"still at the measured mix"* to *"you have moved it"*. The `split` manifest
row no longer claims a `70% / 30%` default the tool stopped using when region defaults landed. The
whole-percent caveat in the indexing note was retired because the slider can now reach the measured
value.

**Guards added (11), all mutation-tested.** Four were vacuous on first write and were rewritten:

- `\\b` inside a regex **literal** is a literal backslash, not a word boundary — the singular-framing
  guard could never match.
- The same guard scanned raw source, but the offending phrase was split across two concatenated
  string fragments (`'…what a data '` + `'center's own power…'`) and never appeared contiguously.
  It now collapses `' + '` before scanning.
- The confirmed-zero check sat behind a filter that **excluded regions whose confirmed share is
  zero** — the branch it tested was unreachable.
- The dividend-direction check read the clause *before* the claim, in a different string fragment,
  so deleting the claim left it passing.

Each is a variant of the same failure: **a check whose expectation is drawn from the same place as
the thing it checks, or from a whitelist of the ways it has already failed.**

Suite: `verify.js` 337, `verify-by-state.js` 300, `render-check.js` 11 + 11. All ten gates clean.

### 2026-09-09 (later) — "LEAD says New Mexico is 2%, we say almost 13%"

Asked while reading LEAD's own site. **Both numbers are right, and both come from the same cells.**
LEAD's headline for a state is the *typical household's* burden; this tool reports the *share of
households* above a threshold. Summing New Mexico's 163,014 LEAD cells:

| | New Mexico | Virginia |
|---|---|---|
| Total energy cost ÷ total income (aggregate burden) | **2.34%** | 1.94% |
| Median household's burden band | **2%** | 2% |
| Households under 6% of income | 598,677 (**76.84%**) | — |
| Households at 6–10% | 79,182 (10.16%) | — |
| Households above 10% — **the tool's figure** | 101,213 (**12.99%**) | 9.19% |
| Of those, households at 60% or above | 9,866 | — |

The distribution is long-tailed: three quarters of the state is under 6%, which is what holds the
average near 2%, while nearly 10,000 households spend 60% or more of their income on energy. **An
average and a tail are different questions**, and a fee-and-dividend is aimed at the tail.

Nothing in either tool said so, which meant the first reader to open LEAD alongside it would think
the figures contradicted each other. The measured-burden note now opens with the reconciliation, and
a new manifest row (`burden-vs-average`) carries the arithmetic. `verify.js` recomputes all six
figures from `lead-burden-income.csv` and fails if the sentence and the source drift apart.

One guard was vacuous on first write and was fixed: it checked for the phrase *"share of
households"*, which appears elsewhere in the tool, so rewording the distinguishing sentence left it
passing. It now anchors both halves of the contrast. A second failed on the file's mixed encoding —
the check string held a literal `\u2019` while the tool source holds the escape — and now normalizes
both before matching.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 11 + 11. All ten gates clean.

### 2026-09-09 (browser pass) — one real bug, two false alarms worth recording

**Found and fixed: slider labels collapsed on a phone.** At 375px every `.slabel` squeezed its label
into a one-word column beside the readout — *"What share of their power counts as fossil"* became
four lines 93px wide. `.slabel output` is `white-space:nowrap`, so the output cannot shrink and the
label absorbs all the loss. Printing the fossil shares in **tenths** earlier the same day made the
readouts four characters longer, which is what pushed it over. The `.stacked` treatment written for
the grid-index row is now a `@media(max-width:560px)` rule applied to every slider label. Verified at
375px, 726px and 1280px on both tools: no squeezed label, no sideways scroll, no element past the
right edge, `.krow` correctly single-column on a phone.

**Two false alarms, both from the harness rather than the tools** — recorded because each looked
exactly like a real defect:

1. **`CIT_BY_ID is not defined` on load.** The console reported it at line 5136, which in the served
   file is a `<p>` inside a template string that cannot throw. The message was **retained from an
   earlier page load** — this session repeatedly wrote deliberately broken copies of the file to disk
   during mutation testing, and the browser had the page open. A fresh tab logs nothing. *Console
   messages survive navigation; check the served line before believing the line number.*

2. **"72 of 195 focus targets unreachable."** The scan filtered on `offsetParent !== null`, which is
   the wrong visibility test in current Chrome: a **closed `<details>` uses `content-visibility:
   hidden`**, so its content keeps a real bounding box and a non-null `offsetParent` while being
   unpainted and unfocusable. `checkVisibility()` returns `false` for it and `.focus()` leaves the
   summary focused — the content is correctly hidden. Rescanned with `checkVisibility()`: **all 170
   (region) / 192 (by-state) visible focus targets take focus, scroll into view and are
   unobstructed**, with the 25 links inside closed `<details>` properly excluded.

Also swept: 1,050 region and 2,145 state renders across every jurisdiction x tier x buildout x
on-site-gas combination with no `undefined`, `NaN`, `[object Object]` or `Infinity` anywhere on the
page; zero dead citation links; defaults correct on a cold load in both tools; notes open **and
close** again; the presenter toggle round-trips.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 11 + 11. All ten gates clean.

### 2026-09-09 (later still) — the rest of the notes, and the grid-index control rebuilt

**The other sections, checked for length.** The always-visible layer turned out to be healthy — the
whole region tool shows **1,446 words** before a reader opens anything, Section 4 the largest at 461
(165 of that the orientation table). The slabs had simply moved *into* the notes, which only Section
4 had been given labeled sections. Five notes ran 121–193 words as one undifferentiated block:
`billBox` (193), `structCompare` (165), `burdenBox` (160), `flatRateHint` (148), `industryBox` (121)
— 787 words in all. Each is now broken into `.wsub` sections. No claim changed.

One caution worth recording: shortening a bolded topic sentence into a section label **deleted the
claim**. "Covering the electricity bill is not the same as ending energy burden" became the label
"Covering the bill is not ending the burden" — losing the word *electricity*, which is the entire
point, since the share on screen is of the electricity bill alone. `verify.js` caught it. The label
is now short and the sentence survives underneath it.

**The grid-index control: rebuilt as a three-way choice.** Twice reported as not working, and the
diagnosis in the end was the control itself: a **17-position range input with no tick marks** for
what is really three options. A reader could see a number move without knowing what they were moving
or how many options existed.

It is now a `.seg` button group, like every other discrete choice in the tool — **Dividend shrinks /
Dividend holds / Dividend grows** — each button stating the fee it produces at a fully clean grid,
with the readout giving today's. The model is untouched: `state.gridIndexing` is still the
continuous exponent; the buttons set 0, 1 or 2.

Beside it sits a trajectory table: rows are futures for the grid, columns are the three choices,
cells are the blended fee rate. For PJM:

| If PJM's grid reaches | Dividend shrinks | Dividend holds | Dividend grows |
|---|---|---|---|
| 60.0% fossil — today | 1.55¢ | 1.55¢ | 1.55¢ |
| 40.0% fossil | 1.37¢ | 1.57¢ | 1.79¢ |
| 20.0% fossil | 1.20¢ | 1.59¢ | 2.13¢ |
| 0.0% fossil — fully clean | 1.02¢ | 1.63¢ | 2.61¢ |

The results-section panel keeps the same table with the dividend under each rate. The split respects
the earlier decision that dividend figures must not appear before the results section states the
dividend once — what sits beside a control is what the control *does*.

Held steady is exactly flat — **1.5204¢ at every grid mix** — once on-site gas is ignored. With it
counted it drifts 1.548¢ → 1.631¢, because captive gas does not clean up with the grid. The table
now shows that rather than the note asserting it.

**Four bugs found while building it:**

1. **"raise it to 1.47¢ there"** — in New Mexico today's rate is *also* 1.4728¢, because with no
   on-site gas the held-steady multiplier preserves the rate exactly. Every button label now states
   the clean-grid rate without a verb that presumes a direction.
2. **A global `table{min-width:640px}`** — written for the wide citation manifest, which does need to
   scroll on a phone — applied to the new tables too, pushing every rate off-screen at 375px. Neither
   `table-layout:fixed` nor an explicit width could override it; `max-width` on a cell does nothing
   under auto table layout either. The grid tables now set `min-width:0`.
3. **The rows read as a duplicate control.** Headed "PJM's grid" — the same words as the grid-mix
   slider directly above — they looked like a second setting. They are now "If PJM's grid reaches",
   with a line saying they are possible futures and the slider picks which one you are standing on.
4. **"Dial" survived the rebuild** in eleven reader-facing strings for a control that is now buttons.
   The remaining uses all refer to the fossil-share slider, which is still a dial.

Six guards added, all mutation-tested: the three buttons render with exactly one pressed and each
stating its own clean-grid rate; the control is not a range input; both tables print the rate
`computeScenario` computes in all 12 cells; and the rate table sits between its slider and the next
control rather than drifting into a later section.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 15 + 15. All ten gates clean.

### 2026-09-09 (evening) — both Section 4 panels rebuilt as problem → steps → answer

Two reports in a row: *"nothing happens"* when clicking the grid-index choices, and *"the data
centers taken together is really confusing to read — not complete sentences, not a holistic story."*
Both panels now follow the same shape.

**"Nothing happens" — intentional arithmetic, broken experience.** At the measured grid mix the index
multiplier is exactly **1.0000×** for all three choices, so they are identical. Move step 1 to 20%
fossil and they separate: 1.14¢/$546 → 1.47¢/$706 → 1.90¢/$912. The explanation existed — inside a
collapsed note *below* the table. It is now an always-visible box directly under the buttons,
carrying the move that resolves it: a **"Show me at 20% fossil →"** button that sets step 1 for the
reader. The flat-rate case (where the choices are also inert, for a different reason) is named too.

**The facility panel had no story.** It opened with a reconciliation table — *"Three shares, three
different things"* — which is a fix for three numbers that all say "fossil", shown before the reader
had met any of them, with row labels that were fragments (*"…plus on-site gas, fossil-rated whatever
they procure (a floor under the dial)"*). It now runs:

- **"How much of their electricity counts as fossil?"** — the question, as the heading
- an intro in sentences, in the jurisdiction's own numbers: the fee charges fossil at twice the clean
  rate, it starts at the measured grid mix because a facility procuring nothing looks like its grid,
  and two things move it — proving cleaner supply, and gas burned on site
- **Step 1 · What they draw from the grid** — the share slider
- **Step 2 · What they burn on their own site** — the on-site gas buttons
- **= So the fee is charged against** — the table, now the *answer*, headed "How the 63.0% is arrived
  at", its rows rewritten as sentences that point back up at the steps that feed them

The grid panel got the same treatment earlier the same evening: **"A cleaner grid raises less money —
so the dividend shrinks"**, with the loss computed per jurisdiction (PJM 1.55¢ → 1.02¢, **34% less**;
across the 24 regions the drop runs from 15% in Bonneville to 48% in Duke Energy Florida), then step
1 (how clean does the grid get) and step 2 (what should the fee do about it).

**Three defects found while building it:**

1. **`\u2014` printed literally in the subhead** — that escape only works inside a JS string, and this
   one was in HTML markup. A sweep now rejects any raw `\uXXXX` in the rendered text.
2. **The table's self-references pointed the wrong way.** It moved from above the controls to below
   them; *"the dial below"* did not move with it.
3. **A new guard poisoned four unrelated checks.** `renderCase` resets region, tier, buildout, shares
   and grid mix — but **not** `rateStructure` or `gridIndexing`. A flat-rate render inside the new
   inert-state guard leaked into every check after it. It now restores what it overrides.

**And a harness weakness worth recording.** Deleting a markup slot entirely — `<p id="gridIndexingInert">`
— left all 20 checks green, because the stub `document.getElementById` **creates** an element on
demand and the renderer wrote happily into a phantom. Same family as the CIT_BY_ID bug: the harness
invented what the page lacked. A new check asserts that **every id written during a full render
appears as an `id=""` in the shipped markup** — 70 of them — and it catches the removal of any slot,
not just this one.

Nine guards added across the two panels, all mutation-tested. Swept 210 region and 429 state renders
(jurisdiction × on-site gas × the three choices) with no broken values, no raw escapes, and the inert
box shown exactly when the multiplier is 1.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 21 + 21. All ten gates clean.

### 2026-09-09 (late) — the control that would not respond, and two hidden steps

**"The fee should change based on what mode you're in, but that's not happening."** Correct, and a
real defect. The model was fine — at 20% fossil the fee genuinely goes **1.20¢ → 1.59¢ → 2.13¢**
across the three choices — but every fee figure *beside the control* was frozen: the readout always
said "the fee is 1.55¢ today", and the buttons were permanently anchored to a fully clean grid
("1.02¢ / 1.63¢ / 2.61¢"). So a reader who moved step 1 to 20% and then clicked through the modes saw
nothing near the control change. Both now report the mix the reader is actually standing on:

> the fee is 2.13¢ at 20.0% fossil · 1.55¢ today
> Dividend shrinks — do nothing, at 20.0% fossil the fee is 1.20¢

**One test now decides "is the grid at today's mix".** Three different tolerances used to answer it:
the grid readout said "— today" within 0.005, the index control used 1e-9, and the inert box tested
the multiplier. They disagreed in LADWP — measured 76.16%, slider stepping in tenths — where at 76.2%
the readout said "today" while the index control said "at 76.2% fossil" and all three buttons showed
1.68¢ regardless. The test is now whether the three choices **render** the same at 2dp, which cannot
contradict what is on screen.

**The GW → MW conversion was correct but invisible.** Asked: *"you have 35.1 GW buildout but then in
the conversion you're using MW — don't you need to convert?"* It does convert (`region.unitToMW`),
and 35.1 × 1,000 × 1,000 × 8,760 × 0.9 = 276,728,400,000 kWh matches the model exactly. But the
derivation **opened** at "35,100 MW" while the reader had chosen "35.1 GW" — the first step was
silent, in a tool whose whole claim is that its figures are re-checkable. Now shown. Making it
visible immediately printed **"35 GW × 1,000 MW/GW = 35,100 MW"**, because `f()` rounds to whole
numbers: visibly broken arithmetic over a correct model. The buildout now keeps its decimal.

**The formula in plain language**, asked for and added — and every step of it is exact, not a
rounded story:

> A fossil kilowatt-hour pays exactly twice a clean one, so the average fee comes to **the clean
> rate × (1 + the fossil share)**. At PJM's 60.0% that is 1.60 clean rates, or 1.52¢. At 20% it would
> be 1.20 clean rates, 1.14¢ — 25% less money. Holding the dividend steady just means scaling back up
> by the ratio: 1.60 ÷ 1.20 = 1.33, returning the rate to 1.52¢. Growing it applies the same step
> twice: 1.33 × 1.33 = 1.78.

`render-check.js` verifies all three identities against the calc core in every jurisdiction at four
grid mixes — `blended = clean × (1 + share)`, `hold = (1 + ref)/(1 + mix)`, `grow = hold²` — to nine
decimals, with behind-the-meter off, since on-site gas is precisely what the note says the formula
cannot follow.

**Three guards were vacuous on first write, all the same way:** they searched for bare numbers
("1.33", "1.52") that appear more than once in the sentence, so corrupting one occurrence left them
green. They now assert whole rendered fragments. Two others were too strict rather than too loose —
Vermont's grid is **0.16% fossil**, so a third of it cannot produce three distinguishable rates, and
the control correctly keeps its clean-grid anchor there.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 23 + 23. All ten gates clean.

### 2026-09-09 (night) — "how are we defining data center in this tool?"

It never said. The answer is worth stating plainly because **the tool does not define the term at
all — it inherits it, and the definition is not the same in every territory.**

**Twenty-one of the 24 regions (and all 51 states) take dcmap.us's facility register.** Summing
`facility_type_counts` across the 52 state records in `sources/16-dcmap-buildout/states.json`:

| type | records | share |
|---|---|---|
| colocation | 3,433 | 68.7% |
| hyperscale | 749 | 15.0% |
| "other" | 545 | 10.9% |
| enterprise | 147 | 2.9% |
| neocloud | 29 | 0.6% |
| research/HPC | 25 | 0.5% |
| institutional | 18 | 0.4% |
| crypto | 16 | 0.3% |
| edge | 14 | 0.3% |
| government | 13 | 0.3% |
| telecom | 9 | 0.2% |
| **total** | **4,998** | |

Two things a reader should know from that table: **hyperscale AI campuses are about a sixth of the
records, not the whole of it**, and **crypto mining sits inside the same count**. Further, only
**1,906 of 4,779 counted records (39.9%) carry a megawatt figure at all**, and dcmap's own
methodology states that the figures which do exist are heterogeneous — a record may report IT load,
gross facility input, utility service, an interconnection request, or a full campus buildout.

**The other three regions are not counting data centers.** PJM, ERCOT and NYISO use the grid
operator's own **large-load** forecast — PJM's "large load adjustments grow by 35.1 GW", ERCOT's
large-load increment, NYISO's "51 large load projects in the queue". Those include data centers
without being limited to them, and the tool charges a data center fee against them.

Added: a `Data center` glossary entry, a `dc-definition` manifest row carrying the full taxonomy and
the coverage figure, and a sentence in Section 2 — where the reader meets the buildout — that names
which of the two definitions applies to the territory on screen. Swept: 21 regions cite the facility
register, 3 the large-load forecast, and all 51 states the register.

**Browser pass, both tools.** 210 region and 429 state renders (jurisdiction x on-site gas x the
three choices): no broken values, no raw unicode escapes, no "X% rather than X%", exactly one
pressed button and one highlighted column throughout. Console clean on cold load in both. 178 and
201 visible focus targets all reachable and unobstructed; no skipped heading levels; every slider
labeled; every citation link named. At 375px both grid tables fit (280px in a 279px wrapper), no
sideways scroll, no element past the right edge.

**One false alarm, recorded because it looked real:** a tab reported `tier=low, rateStructure=flat`
on what seemed a fresh load. A genuinely new tab gives the documented defaults — moderate,
differentiated, Confirmed, 35.1 GW — and the tool has no persistence for either field. It was my own
earlier probing in that tab, not the tool.

**One stale comment corrected.** The global `table{min-width:640px}` rule was described as existing
"for the wide citation manifest", but the manifest is div-based: the two `.gtab` tables are now the
only `<table>` elements on the page, so that rule applies to nothing else. The comment now says so,
and notes that `table-layout:fixed` is what makes the column width bind at all.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 23 + 23. All ten gates clean.

### 2026-09-09 (night, cont.) — the elasticity rationale, and the formula behind it

Asked: *"what rationale did you use for elasticity? it would be good to have some thinking behind
that"* — and then *"how did you figure that out? is there a formula?"* Both are worth answering in
the file, because the reasoning existed only in the manifest locator and part of it had gone stale.

**The dial defaults to 0 — no behavioral response — and that is a refusal, not an omission.** An
elasticity needs price variation matched to quantity variation with everything else held constant.
This project has buildout figures and industrial prices per jurisdiction and no controls at all for
land cost, fiber, water, permitting speed, state incentives or agglomeration.

**Two formulas are involved, and only one of them touches data.**

*The break-evens are pure algebra.* Let `s` be the fee as a share of the local industrial price and
`ε` the elasticity. The tool's buildout multiplier is `max(0, 1 − ε·s)`, so revenue goes as
`s·(1 − ε·s)`. Setting the derivative to zero gives `s = 1/(2ε)`, i.e. **revenue peaks at
ε = 1/(2s)**, and the multiplier hits zero at **ε = 1/s**. No estimation, no data fitting — which is
why the panel calls them "the two numbers that need no assumption". Verified exact against the calc
core in every jurisdiction.

*The fit is ordinary least squares on logs.* Elasticity is `%ΔQ / %ΔP`, which is exactly the slope
of `ln Q` on `ln P`:

    slope = Σ(ln Pᵢ − mean ln P)(ln Qᵢ − mean ln Q) / Σ(ln Pᵢ − mean ln P)²
    R²    = cov² / (var_lnP × var_lnQ)

with Q = mid-scenario buildout MW per 1,000 households and P = industrial ¢/kWh.

**What the fit actually returns is the argument.** Across the 24 regions: **−1.6 on an R² of 13%**.
Across the 46 states with a nonzero buildout: **−2.2 on an R² of 4%**. Both slopes are large enough
to wipe out most of the buildout, and both rest on almost no explanatory power. That is the case for
refusing to estimate: a cross-sectional fit here produces a confident-looking number that is nearly
all noise.

**A stale figure found and fixed.** The manifest claimed ERCOT carries "4.7× the buildout intensity"
of PJM. That was computed against ERCOT's 59.7 GW case, which is now its **low** scenario — on the
mid scenarios both tools default to it is **7.8×**. The locator now names the scenario, and the
on-screen version is computed at render time so it cannot drift again.

**A hand-picked pair was tried first and abandoned.** The region tool's ERCOT-vs-PJM comparison makes
the point vividly (24% cheaper power, 7.8× the buildout). Its by-state equivalent does **not**: Texas
is 31.9% cheaper than Virginia with slightly *less* buildout per household, so the pair version
silently printed nothing there — the guard's own `ratio > 1` test suppressed it. Fitting the whole
cross-section makes the point in both tools regardless of which way any single pair happens to fall.

The panel now runs: what it means and why it is left to you → why not just fit one → what can be said
without any assumption → what would settle it (Virginia's HB 30, 1.1¢/kWh from July 2026 in the
highest-concentration data center market on earth, over 2026–2028).

Guarded: the slope, the R², the jurisdiction count and both break-evens are recomputed in
`render-check.js` from `REGIONS` alone and compared against what the panel prints, with a check that
fails if R² ever rises above 25% — at which point "price explains almost none of it" would stop being
true and the argument would need rewriting rather than the number quietly updating.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 24 + 24. All ten gates clean.

### 2026-09-09 (night, cont.) — three lines in the derivation that did not add up as printed

A reader working through the arithmetic panel caught two of these and asked whether the tool was
wrong. **The model was right in every case; the printed steps were not checkable.** All three are the
same defect: *rounded operands, unrounded result.*

**1 · "Panel 4 says 174,209,062,104 kWh but panel 1 says 276,728,400,000."** Both correct —
174,209,062,104 fossil + 102,519,337,896 clean = 276,728,400,000 exactly — but panel 4 printed the
two split figures with no bridge from the total, so the larger number looked lost. It now states the
split under the total it came from and shows the two parts adding back to it.

The bridge I first wrote made it worse: `276,728,400,000 × 62.95% = 174,209,062,104` cannot be
verified, because the true share is 62.953084% and the rounded one gives 174,200,527,800 — eight
million kWh out. A share rounded for display must not be printed as an operand. It is now a label on
each part; what reconciles is the sum.

**2 · The divisor was off by one as shown.** `2 × 2,750,748 + 2,656,596` is **8,158,092**, but the
panel printed **8,158,091** — because the counts were rounded for display while the divisor came from
the unrounded 2,750,747.62 and 2,656,596.23. The printed divisor is now the one the printed counts
give. Measured before changing it: the model's own divisor differs by **0.53 households in 8.16
million (0.06 ppm)** and moves no displayed figure by a cent — $87.518547 against $87.518542, both
$87.52. A one-line note says the model carries the fractions.

**3 · `((1 + 0.600) ÷ (1 + 0.600)) ^ 2.00 = ×1.0000`** — flagged as "this math doesn't add up." It
does, trivially, but it reads as a typo because **two different quantities were printed as the same
number**: the numerator is the territory's *measured* mix, the denominator is *where the grid slider
sits*. At 3dp both showed 0.600 with nothing to distinguish them. They are now labeled and shown at
4dp, with a line stating that they coincide only because the slider is still at the measured mix —
which is exactly why the multiplier is 1.

**The systematic fix.** Rather than patch the two lines that were reported, I parsed every equation
the panel prints and checked each one reconciles. That found the third — the 62.95% multiplication I
had introduced hours earlier. `render-check.js` now re-derives the split, the sum and the divisor
from `computeScenario` and compares them against what the panel prints, in every jurisdiction, with
zero-buildout jurisdictions (DC, Massachusetts) skipped since they have no split to check.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 25 + 25. All ten gates clean.

### 2026-09-09 (night, cont.) — browser pass, and a defect my own fix introduced

**One real bug found, and I had caused it hours earlier.** Changing step 5 to print the divisor its
own rounded counts produce left **step 6 still dividing by the model's unrounded one** — so a single
derivation showed two divisors, `8,158,092` in step 5 and `8,158,091` in step 6, one household apart.
Step 6 now uses the same figure.

Worth recording *how* it escaped: the equation sweep I built to catch exactly this class of problem
parsed step 5's divisor line and confirmed it reconciled — it never compared step 5's divisor against
step 6's use of it. **Checking each line in isolation is not the same as checking the derivation
hangs together.** The guard now asserts the pair.

**Browser pass, both tools, fresh tabs and clean consoles.**

| | region | by state |
|---|---|---|
| Printed equations parsed and reconciled | 560 across 120 region × buildout renders | 1,075 across 255 state × buildout renders |
| Renders swept (jurisdiction × on-site gas × three choices) | 210 | 429 |
| Broken values / raw escapes / "X% rather than X%" | none | none |
| Visible focus targets — reachable, unobstructed | 199/199 | 222/222 |
| Console errors on cold load | none | none |
| Skipped headings, unlabelled sliders, unnamed citations | 0 / 0 / 0 | 0 / 0 / 0 |

Every equation the arithmetic panel prints is now parsed and re-derived in the browser as well as in
`render-check.js`: the GW→MW step, MW→kWh, the fossil/clean split summing back to the total, the
on-site-gas floor formula, the index multiplier, and the divisor. All reconcile in every jurisdiction
at every buildout scenario.

At 375px both grid tables fit (280px in a 279px wrapper), no page-level sideways scroll, nothing past
the right edge, no squeezed slider labels, and the monospace derivation blocks scroll inside their
own `overflow-x:auto` containers rather than clipping. Notes still open **and** close. Defaults
correct on a cold load in both tools.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 25 + 25. All ten gates clean.

### 2026-09-09 (night, cont.) — the payment section, and a sources inventory

**Asked to check the payment section for other inconsistencies. There were four**, all the same
family as before — a figure rounded for display then used as an operand.

1. **`$4.28 billion ÷ 8,158,092 = $525.11`** — a reader doing that division gets **$524.63**, 48
   cents out, because the revenue was rounded to three significant figures *as an operand*. It now
   prints in full: `$4,283,905,890 ÷ … = $525.11`.
2. **The bill column did not add up.** `$123.75 + $120.55` is `$244.30`, printed as `$244.31`,
   because the total came from the unrounded parts — and "still unpaid" inherited the extra cent.
   Every figure in that column is now derived from the two rounded lines shown.
3. **The three destinations did not sum to the dividend.** In Ohio, `117.30 + 16.42` exceeds the
   `133.71` dividend by a cent, so the clamped remainder left the column over. Each destination is
   now taken out of what is left, so the three always sum exactly.
4. **The divisor rounding was worse than I had claimed.** I had written that rounding the household
   counts "shifts the divisor by less than one household and no printed figure at all". Both halves
   were wrong: `2 × round(severe) + round(moderate)` can be **1.5 households** out, and in small
   jurisdictions that moved the printed annual payment by up to **11 cents** (Utah), one cent of it
   reaching a *monthly* figure. Written from PJM alone, where 0.5 households in 8.16 million is
   invisible. The counts are now shown to two decimals — they are estimates, not headcounts — and
   the divisor and the division are built from exactly those figures, so every line reconciles by
   construction while the monthly figures move by fractions of a cent.

Each was found by parsing what the panel *prints* rather than by reading the code, and the parser is
now part of `render-check.js`: revenue ÷ divisor, ×2, ÷12, the bill column's sum, the destinations'
sum, and unpaid = total − dividend, in every jurisdiction.

**And a sources inventory, asked for directly.** `files/SOURCES.md` now lists every primary document
held in `/sources`: **44 downloaded files** — 16 PDFs, 6 raw datasets, 22 saved HTML captures,
166 MB — with publisher, size, source URLs, and the scripts and provenance notes for each of the 14
topic folders. It is **generated** by `sources/build-sources-inventory.py` from the tree itself, for
the same reason `files/CITATIONS.md` is generated: a hand-kept list of 44 files goes stale the first
time anything is added, and a stale source list is worse than none because it invites a reader to
think the project holds evidence it does not. That makes **seven** drift gates; `CLAUDE.md` records
the new one.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 25 + 25. All seven gates clean.

### 2026-09-09 (night, cont.) — the raw datasets, and the last of the reserve fund

Asked which raw datasets the tool uses. **Six**, and answering it turned up two things the inventory
had not been asked to reveal.

| dataset | size | feeds |
|---|---|---|
| `egrid2023_data_rev2.xlsx` (EPA eGRID 2023 rev 2, sheet BA23) | 20.2 MB | every territory's grid fossil share, 69 balancing authorities |
| `NM-2022-LEAD-data.zip` + unpacked CSV (DOE LEAD 2022) | 54 + 51 MB | household energy burden, 163,014 New Mexico cells |
| `f8612024.zip` (EIA Form 861, 2024) | 4.4 MB | household counts, the region→utility map |
| `eia861m-sales-revenue.xlsx` (EIA Form 861M) | 2.2 MB | residential and industrial prices |
| `gogpt-august-2026.xlsx` (Global Energy Monitor, CC-BY 4.0) | 3.1 MB | unit-level gas tied to data centers |
| `recs-2024-HC11.1-energy-insecurity.xlsx` (EIA RECS 2024) | 20 KB | national context; enters no calculation |

**The "re-checkable locally" claim is qualified in one place, and it was recorded nowhere.**
`lead-harvest.py` downloads one state's LEAD archive at a time, derives its figures and **deletes
it** — only New Mexico's was kept, as a worked example; the full set runs to several gigabytes.
What is committed for all 51 jurisdictions is the derived output plus the script and the URL
pattern. So for household burden the other 50 are *reproducible* but not *local*. That limitation
lived in a single docstring line in the harvest script; it is now stated at the top of
`files/SOURCES.md` and in `lead-burden-shares.source.md`.

**Two folders held documents with no provenance note.** `05-renewable-classification` (eGRID —
the most-used dataset in either tool) has its provenance recorded thoroughly in the `egrid-fossil`
citation instead, down to the sheet and the four column names, but **no download URL is recorded
anywhere in the project**, so the note cannot be written without inventing one. Flagged rather than
filled in.

**The other was the reserve-fund captures, and they are now gone.** `06-fund-design` held
two HTML pages gathered for the reserve fund that was cut from both tools on the owner's request.
Nothing operational referenced them — checked across every HTML, JS, Python, Markdown and CSV file
in the project before deleting. Removed at the owner's instruction, along with the folder's row in
`sources/README.md` and its entry in the inventory generator's title map. The gathering checklist's
outstanding payout-methodology item is struck through as no longer needed rather than
left implying work outstanding.

What deliberately stays: `files/METHODOLOGY.md`'s account of the fund and its removal, and
`CLAUDE.md`'s note of the same — those are the decision record. The naming rule stays too; as of
2026-09-13 it no longer names the sovereign-fund comparison, at the owner's request.

Downloaded documents: **42** (was 44), 166 MB across 12 topic folders.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 25 + 25. All seven gates clean.

### 2026-09-09 (night, cont.) — browser pass: a hardcoded count that was wrong in both tools

**One defect found, and it was mine from an hour earlier.** The rounding disclosure in step 6 said
*"and does in six of the twenty-four territories"* — a number I had hardcoded. It was wrong twice:

- **In the by-state tool it was the region tool's count**, describing 24 territories in a tool with
  51. The real figure there is **32 of 46**.
- **In the region tool it was simply stale.** After the divisor moved to two decimals the real
  figure is **15 of 24**, not six.

Both are now counted at render time from the tool's own `REGION_ORDER`. `render-check.js` requires
the count to be present and rejects a count larger than the tool has jurisdictions — which is what
would have caught the cross-tool copy.

Two guard weaknesses fixed alongside it, both the same shape as ones already recorded: the
disclosure check tested only the *premise* ("rounded to the cent from the unrounded figure above
it") and not the *consequence* ("can land a cent away"), so a mutation rewriting the consequence to
say the opposite passed. It now requires both halves.

**Browser pass, both tools, fresh tabs and clean consoles.**

| | region | by state |
|---|---|---|
| Printed equations parsed and reconciled | **1,011** across 120 renders | **2,036** across 255 renders |
| Renders swept (jurisdiction × on-site gas × three choices) | 210 | 429 |
| Broken values / raw escapes / "X% rather than X%" | none | none |
| Visible focus targets — reachable, unobstructed | 199/199 | 222/222 |
| Console errors on cold load | none | none |
| Sovereign-fund references remaining | none | none |

The equation sweep now covers the payment section too — revenue ÷ divisor, × 2, ÷ 12, the bill
column's sum, the three destinations summing to the dividend, and unpaid = total − dividend. All
reconcile in every jurisdiction at every buildout scenario, in both tools.

At 375px both grid tables fit (280px in a 279px wrapper), no page-level sideways scroll, nothing past
the right edge, no squeezed slider labels, and all ten monospace derivation blocks scroll inside
their own containers rather than clipping. Defaults correct on a cold load in both.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 25 + 25. All seven gates clean.

### 2026-09-09 (night, end) — the last provenance gap closed

`sources/05-renewable-classification/` held eGRID — 20 MB, **the single most-used dataset in either
tool** — with no `*.source.md` note. Its method was recorded thoroughly inside the `egrid-fossil`
citation locator (sheet BA23, the four column names, the caveats), but nothing recorded **where the
file came from**, and I would not write the note from a guessed URL: a guessed address in a
provenance note is worse than none, because it looks verified.

The project owner supplied both pages — `epa.gov/egrid` and `epa.gov/egrid/detailed-data` — and
`egrid.source.md` now records them, with the honest qualifier that **the direct file URL was never
captured**, only the pages, so anyone re-downloading should confirm they have **rev 2** at
**21,213,301 bytes**; eGRID reissues revisions under similar names.

Every figure in the note was checked against the committed CSV before it was written:

| claim | verified |
|---|---|
| 69 balancing authorities | 69 |
| Total generation 4,190,974,905 MWh | exact |
| Of which fossil 2,526,058,297 MWh — 60.27% | exact |
| APZS 83.5% · PNM 40.4% · PJM 60.0% · ERCOT 62.5% · NYISO 50.7% | 83.55 · 40.41 · 60.04 · 62.54 · 50.67 |
| File 21,213,301 bytes | exact |

The note also records what the figure *does* — it is the measured row in the facility panel, the
default position of the facility dial, and the reference the grid-index choice normalizes against,
which is why all three choices give an identical fee until the grid slider moves — and the three
caveats that matter when it is quoted: eGRID is generation-located rather than consumption-located,
nuclear is counted clean, and 2023 data published in 2025 sits two years behind the buildout figures
beside it.

**And the gap is now self-reporting.** The inventory generator listed provenance notes but said
nothing when a folder had documents and none — which is why eGRID went unnoticed until someone asked
what the raw datasets were. `files/SOURCES.md` now prints a warning block for any such folder. There
are none today; the point is that the next one announces itself.

Every folder holding a downloaded document now has a provenance note.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 25 + 25. All seven gates clean.

### 2026-09-09 (night, end) — the Sources section was describing a retired method

Asked to check the Sources section for accuracy. **One row was wrong, and wrong in the worst way:
not a typo, but a full description of a method the tool no longer uses.**

`burden-pjm` stated **10.84% / 10.46%** and its locator explained, at length, how those figures came
from applying one uniform **34.3% coverage fraction** to PJM's partly-covered states so the total
would match "PJM's own 26.4M household count".

All of that was retired when `region-households.py` replaced the approximation with EIA-861 per-state
apportionment. The tool has computed with **10.81% / 10.44%** ever since, from
`region-households.csv` — 24,535,221 usable households, 2,653,417 severe, 2,562,585 moderate, across
**15** states, not "13 plus DC". `region-households.source.md` records the changeover explicitly and
even notes that the old approximation landed within 0.03 points. The manifest row was simply never
updated, so the Sources section spent that whole time explaining a superseded derivation to numbers
the tool does not produce.

The row now states what the tool computes, describes the apportionment actually used, notes that
shares are taken on *usable* households (LEAD can only compute burden where income is reported), and
keeps the retired method as recorded history rather than as the live account.

**Nothing compared a citation's stated value against the data the tool runs on.** That is why it
drifted for as long as it took someone to ask. Two guards added:

- **every per-jurisdiction citation must state the figure `REGIONS` holds** — household counts,
  burden shares and buildout scenarios, **153 values** checked in the by-state tool and 72 in the
  region tool;
- **every file path a citation names must exist on disk** — the provenance trail cannot end in
  nothing.

Both mutation-tested four ways: reverting the burden value, corrupting a household count, inventing
a buildout figure, and pointing a source at a file that is not there. All caught.

The guard was itself wrong once before it was right: it stripped zeros as noise, then reported DC,
Massachusetts, New Hampshire and Vermont as omitting a mid scenario they state correctly as
`0 / 0 / N MW`. **Zeros are data.**

Also verified in this pass: the Sources header counts are generated, not literals (145 rows, 28
flagged, 4 NEEDS SOURCE, all recomputing from the array); all 145 rows render with no orphans in
either direction; the 45 citation ids shared between the two tools carry identical confidence
labels; and all 39 / 25 distinct source paths named across the two manifests resolve.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 27 + 27. All seven gates clean.

### 2026-09-09 (night, end) — the household-accuracy note contradicted itself three ways

Asked to check the section on household accuracy. **The PJM burden note was half-rewritten when the
apportionment method changed, and had been contradicting itself ever since — three ways in five
sentences.**

1. **It named both methods as the method**, in one sentence: *"…weighting the 6 partly-covered
   states by a blanket fraction; each state is now weighted by how many of its residential
   electricity customers EIA actually records inside PJM."*
2. **It said the apportionment is measured, then that it isn't** — *"so the apportionment is
   measured"* two clauses before *"Treat this as 'about 10.8%', **not a measurement**"*.
3. **It editorialised past the source.** *"0.03 points away, which was luck rather than method"* —
   `region-households.source.md` says the approximation *"turned out to be sound"*, which is a
   different claim.

And a factual error underneath all three: the note said the apportionment draws on **13 states + DC**.
Running EIA-861 through the shipped loader, PJM-coded residential customers are filed in **15
jurisdictions — 14 states plus DC**:

| coverage | jurisdictions |
|---|---|
| essentially full | PA 99.9%, OH, NJ, MD, WV, DE, DC (100%), VA 98.8% — **8** |
| partial | IL 71.1%, KY 39.1%, IN 19.6%, MI 3.7%, NC 3.5%, TN 1.4%, **NY 0.1%** — **7** |

The fourteenth state is New York, 5,815 customers — a sliver on the border that PJM's own "13 states
+ DC" description does not mention and the old note therefore missed.

The note now states PJM's own territory description **and** the 15 jurisdictions the apportionment
actually reads, gives the coverage spread, names one method, quotes the shares in use (10.81% /
10.44%) alongside the sensitivity bounds and the retired figure, and says the shares are carried to
two decimals rather than implying more precision than they have.

**Also measured while checking, and left as-is deliberately:** `severeShare` is stored rounded to
two decimals (0.1081 against a true 0.108147), which costs **1,202 households** out of 2.75 million
— 0.04%. That is a reporting-precision choice, consistent throughout the tool, and the note now says
so; carrying more digits would imply precision the whole-state source does not have.

**A guard, mutation-tested five ways.** Each burden note must state the severe share the tool holds,
must not name two apportionment methods, must not say "is measured" and "not a measurement" together,
must not carry the editorialising phrasing, and its stated range must bracket the figure in use.
Caught: reinstating either contradiction, reinstating the editorialising, drifting the stored share,
and removing the share from the prose entirely.

The guard was itself too strict on the first write — it demanded *both* shares appear in every note,
which failed Alabama Power, whose note is about the severe share being the highest in the tool and
has no reason to restate the other. Narrowed to the condition that actually failed.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 28 + 28. All seven gates clean.

### 2026-09-09 (night, end) — the buildout, checked against its derivations

Asked to check the buildout for accuracy. **Every figure traces. Two things needed saying that were
not being said.**

**1 · PJM's low case is not from PJM's forecast.** The mid (35.1 GW) and high (65.7 GW) are quoted
verbatim in the citation — *"large load adjustments grow by 35.1 GW between 2026 and 2031"*. The
**10 GW low case is not in that document, or any other**: it is a round floor this project chose,
presented under a *"near-term, firm"* tag, resting on the on-screen claim that *"roughly the first
10 GW is close to booked business"*. No source states it.

Compare ERCOT, where the equivalent low case is handled properly: its locator says outright *"THE
59.7 GW HEADLINE IS AN ESTIMATE: 259.8 GW x 23%"* and names both sources for the 23%. PJM's had no
such disclosure. It does now — in the manifest and where the reader meets the scenario — saying the
tag describes the intent, not a published figure.

Its arithmetic does check where it is sourced: ERCOT's 367,790 − 108,023 = 259,767 MW = the 259.8 GW
filed increment, and 259.8 × 23% = 59.7 GW exactly. NYISO's three scenarios are all quoted verbatim.

**2 · The rounding convention is applied to half the territories.** `dcmap-buildout.source.md` says
the apportioned figures are *"rounded to avoid false precision (an apportionment calculation like
'35,523.2 MW' implies far more precision than the underlying method supports)"*. In the region tool
**13 territories are rounded and 8 are carried at MW precision** — and the split maps exactly onto
the two batches in which regions were added. Neither is wrong: the rounded ones follow the stated
policy, the exact ones reproduce the derivation. But the policy is stated as universal and is not.
**Left for the owner to decide** — applying it to the remaining 8 would shift every downstream
figure in those territories, which is their call, not a correctness fix.

**A guard, mutation-tested.** Every dcmap-sourced scenario must match its derivation, rounded or
exact, in both tools — 21 territories against `dcmap-buildout.csv`, 51 against `state-pilot.csv`.
Plus a guard that PJM's unsourced low case stays disclosed. Caught: inventing a scenario value, and
removing either half of the PJM disclosure.

Three things the guard got wrong before it got them right, all worth recording:

- It compared PJM, ERCOT and NYISO against dcmap's **cross-check rows** — figures the citations
  record as comparisons and deliberately do not use. It now identifies the source from the citation,
  not from the presence of a row.
- The by-state tool reads **`state-pilot.csv`**, not `dcmap-buildout-by-state.csv` — the latter is
  the per-state apportionment *detail* behind the region figures. Reading it matched nothing, and
  a guard that matches nothing passes.
- The two files order their columns differently and `state-pilot.csv`'s are **alphabetical**, so
  positional parsing read the wrong ones. Both are now read by header name.

Tolerance set from the data rather than guessed: the widest genuine rounding in use is **4.5%**
(Arizona Public Service, 382.7 → 400), so 5% admits every real case and still catches an invented
figure. The two 100% cases — PNM 0.2 → 0 and ISO-NE 0.3 → 0 — are documented in the source note as
real findings, and the absolute floor admits them.

Suite: `verify.js` 338, `verify-by-state.js` 301, `render-check.js` 30 + 29. All seven gates clean.
