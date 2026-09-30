# What a data center fee would pay your household

Two pages modeling a per-kilowatt-hour **fee** on electricity used by data centers, paid back as a
**dividend** to households that spend the largest share of their income on energy.

By Anais. Published 2026-09-29.

- **[By state](public-by-state.html)** — all 50 states plus DC
- **[By grid region](public-by-region.html)** — 24 RTO, ISO and utility territories

Both are self-contained: no build step, no dependencies, and no network access at runtime.

## The full versions

The two pages above keep most assumptions fixed. The full tools turn every one of them into a
control you can move, and show the source and confidence tier for each figure on screen:

- **[By state, full version](data-center-tax-tool-by-state.html)**
- **[By grid region, full version](data-center-tax-tool.html)**

## Checking a figure yourself

The derivation scripts and the raw source documents are not in this repository — the model is
developed in a separate working repository, where every figure is re-derived by running a script
that must reproduce its committed output byte for byte. What is here lets you check a figure. It will not
let you re-derive one. These four files cover the first of those:

| file | what it gives you |
|---|---|
| **[MASTER-DATA.csv](MASTER-DATA.csv)** | 420 rows, one per figure displayed anywhere: its value, confidence tier, citation, caveats, and **the script that produced it**. Opens in a spreadsheet. Start here. |
| **[CITATIONS.md](CITATIONS.md)** · **[by state](CITATIONS-by-state.md)** | every number in each tool mapped to its source document and locator, plus what that number does in the model |
| **[SOURCES.md](SOURCES.md)** | an inventory of every primary document the model is built on — publisher, size, and which figures derive from it |
| **[METHODOLOGY.md](METHODOLOGY.md)** | how each figure is derived, end to end. Section 6 is a log of the errors this process caught, and what was done about each |

The first three are not kept by hand. The two citation manifests are generated from the tools' own
citation arrays and the inventory from the source tree, and an automated check fails if any of them
stops matching what the pages display.

METHODOLOGY.md is written by hand, and it names files that are not published here — the derivation
scripts, the expected-result tables, the automated checks. Where it does, it is citing a check that
exists in the working repository rather than offering you a file to open. Its section 2 says which
is which.

## Where the numbers come from

| | |
|---|---|
| **EIA Form 861 (2024) / 861M** | utilities, household counts, retail prices |
| **DOE LEAD 2022** | energy burden — who spends the most of their income on energy |
| **EPA eGRID 2023 rev 2** | grid fuel mix per balancing authority and per state |
| **dcmap.us** | data center buildout pipeline — state aggregates, data as of 2026-08-31 |
| **Global Energy Monitor** | unit-level gas plants, for generation inside the fence |

Each figure is current as of the vintage beside it. The publication date at the top of this page is
the date the pages were built; it says nothing about when the data underneath was collected. The
buildout pipeline moves fastest of the five, and it is a snapshot taken 2026-08-31.

The 6% and 10% energy-burden thresholds are **ACEEE's**, applied to DOE LEAD's data. DOE's own LEAD
pages define only the 6% threshold and no 10% severe threshold.

Four figures in the model have no source, and the full tool says so on screen. Three are modeling
dials that no source could settle, because they are the assumptions a reader is meant to choose. The
fourth records that no source quantifies behind-the-meter renewable capacity at US data centers.
That was searched for and not found, which is a result in itself.

## Every file here is generated

Every file here is written out of the working repository by an export script, against an explicit
manifest. An automated check fails if the two copies diverge, and also if a file turns up here that
the manifest does not list. **Do not edit these files.** The next export would overwrite them.

The calculation exists once. Both public pages carry the same `computeScenario`, copied verbatim from
the tool it belongs to, and a second check fails if that copy drifts.
