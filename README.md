# What a data center fee would pay your household

Two pages modeling a per-kilowatt-hour **fee** on electricity used by data centers, paid back as a
**dividend** to households that spend the largest share of their income on energy.

By Anais. Published 2026-09-29.

- **[By state](public-by-state.html)** — all 50 states plus DC
- **[By grid region](public-by-region.html)** — 24 RTO, ISO and utility territories

Both are self-contained: no build step, no dependencies, and no network access at runtime.

## The full versions

The two pages above hold most assumptions fixed. The full tools expose every one of them as a
control, and attach a source and a confidence tier to each figure on screen:

- **[By state, full version](data-center-tax-tool-by-state.html)**
- **[By grid region, full version](data-center-tax-tool.html)**

## Checking a figure yourself

The derivation scripts and the raw source documents are not in this repository — the model is
developed in a separate working repository, where every figure is re-derived by running a script
that must reproduce its committed output byte for byte. So what is here supports **checking** a
number, not re-deriving it. That distinction is the honest one, and these four files are enough for
the first job:

| file | what it gives you |
|---|---|
| **[MASTER-DATA.csv](MASTER-DATA.csv)** | 420 rows, one per figure displayed anywhere: its value, confidence tier, citation, caveats, and **the script that produced it**. Opens in a spreadsheet. Start here. |
| **[CITATIONS.md](CITATIONS.md)** · **[by state](CITATIONS-by-state.md)** | every number in each tool mapped to its source document and locator, plus what that number does in the model |
| **[SOURCES.md](SOURCES.md)** | an inventory of every primary document the model is built on — publisher, size, and which figures derive from it |
| **[METHODOLOGY.md](METHODOLOGY.md)** | the end-to-end derivation of every figure, and a running log of errors found and corrected |

All four are generated from the tools themselves and checked by an automated gate, never kept by
hand, so they cannot quietly disagree with what the pages display. METHODOLOGY.md refers in places
to `CLAUDE.md`, the working rules the project is built under; that file is not published here.

## Where the numbers come from

| | |
|---|---|
| **EIA Form 861 (2024) / 861M** | utilities, household counts, retail prices |
| **DOE LEAD 2022** | energy burden — who spends the most of their income on energy |
| **EPA eGRID 2023 rev 2** | grid fuel mix per balancing authority and per state |
| **dcmap.us** | data center buildout pipeline — state aggregates, data as of 2026-08-31 |
| **Global Energy Monitor** | unit-level gas plants, for generation inside the fence |

A figure is only as current as the vintage beside it. The publication date above is when these
pages were built, not when the underlying data was collected — the buildout pipeline is the one
that moves fastest, and it is a 2026-08-31 snapshot.

The 6% and 10% energy-burden thresholds are **ACEEE's**, applied to DOE LEAD's data. DOE's own LEAD
pages define only the 6% threshold and no 10% severe threshold.

Four figures in the model are not sourced, and the full tool says so on screen: three are modeling
dials no source could settle, and the fourth records that no source quantifies behind-the-meter
renewable capacity at US data centers — a searched-and-not-found result.

## Every file here is generated

Nothing in this repository is edited in place. It is written out of the working repository by an
export script against an explicit manifest, and a gate fails if the two diverge or if a file appears
here that is not on the manifest. **Do not edit these files** — the next export would overwrite
them. The calculation is not duplicated either: both public pages carry one `computeScenario`, taken
verbatim from the tool it belongs to, and a separate gate fails if that copy drifts.
