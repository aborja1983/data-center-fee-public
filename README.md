# What a data center fee would pay your household

Two pages modeling a per-kilowatt-hour **fee** on electricity used by data centers, paid back as a
**dividend** to households that spend the largest share of their income on energy.

- **[By state](public-by-state.html)** — all 50 states plus DC
- **[By grid region](public-by-region.html)** — 24 RTO, ISO and utility territories

Both are self-contained: no build step, no dependencies, and no network access at runtime.

## These pages are generated

They are built from a working repository that holds the model, every source document, and the
derivation script behind each figure. **Do not edit them here** — edits would be overwritten the
next time they are exported.

The calculation is not duplicated. Both pages carry a copy of one `computeScenario`, taken verbatim
from the canonical tool, and a gate in the source repository fails if the copy drifts.

## Where the numbers come from

| | |
|---|---|
| **EIA Form 861 / 861M** | utilities, household counts, retail prices |
| **DOE LEAD 2022** | energy burden — who spends the most of their income on energy |
| **EPA eGRID 2023 rev 2** | grid fuel mix per balancing authority and per state |
| **dcmap.us** | data center buildout pipeline |
| **Global Energy Monitor** | unit-level gas plants, for generation inside the fence |

The 6% and 10% energy-burden thresholds are **ACEEE's**, applied to DOE LEAD's data. DOE's own LEAD
pages define only the 6% threshold and no 10% severe threshold.

Four figures in the model are not sourced, and the full tool says so on screen: three are modeling
dials no source could settle, and the fourth records that no source quantifies behind-the-meter
renewable capacity at US data centers — a searched-and-not-found result.
