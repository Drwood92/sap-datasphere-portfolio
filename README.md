# From SAP BW to SAP Datasphere: My Learning Journey

Hi, I'm Cyril, an SAP BW consultant in Germany. I've worked with BW since 2020: data models, ABAP routines, process chains, queries and SAC.

Everyone in our world is talking about Datasphere. I didn't want to just read about it. I wanted to **build** with it, break things and understand *why* they broke. This repo is the result.

Everything here was built by me on training systems (S/4HANA 2025, BW/4HANA 2023 and a Datasphere trial), with my own objects and my own data. The files are CSN/JSON exports of the Datasphere objects. They contain definitions only, no data.

---

## Project 1: MD04 has no extractor, so I built one

Every planner knows MD04, the stock/requirements list. But BW and Datasphere can't see it, because there's no standard DataSource. The list is calculated when you open it, not stored in a table.

So I built the whole chain myself:

1. A **custom function module** in S/4HANA that calls `MD_STOCK_REQUIREMENTS_LIST_API`, one material at a time
2. A **generic DataSource** on top, tested in RSA3 against MD04 itself
3. **ODP exposure**, so Datasphere can find it
4. A **replication flow** in Datasphere into a local table

**What went wrong (and what it taught me):**

- 🗓️ **My dates arrived as text.** The first load worked, but every date was a `String(8)`. The fix was the flow's content type: *Template Type* instead of *Native*. The catch is that changing it later resets your projections, so now I set it before the first deploy.
- 🔍 **Zero rows, no error.** My first run loaded nothing. The filter in the projection didn't match my test data. It's a silent failure, the most dangerous kind.
- ⏱️ **A timeout at scale.** One material worked. A whole plant hit a transfer timeout, because MD04 is calculated per material and that is slow. It retried and finished with **5,840 rows**, but in a real project I'd load per plant.

📁 `RF_CY_MD04.json`

---

## Project 2: BW/4HANA data, modelled in Datasphere

I wanted to see how my BW world translates. So I built sales header and item ADSOs in BW/4HANA, loaded them from flat files, replicated them to Datasphere and rebuilt the reporting model there.

**What surprised me as a BW person:**

- **One replication flow, two ADSOs.** In BW, every target gets its own DTP. Here, one flow carried header and item together.
- **Master data without texts.** My `0MATERIAL` had no material type filled. Instead of giving up, I derived one with a calculated column (`CASE WHEN MATERIAL LIKE 'DELL%' ...`) and built a hierarchy on top.
- **Associations feel like navigation attributes.** You define them once on the fact view and the analytic model inherits them.
- **Exception aggregation still matters.** "How many orders are above 10,000?" sounds simple. But an *order* is the sum of its *items*, so the check must run per document number, not per row. Without exception aggregation (SUM on DOC_NUMBER) you get a wrong answer that looks right. It's the same lesson as in BEx, only in a new tool.

**What's inside:**

| File | Contents |
|---|---|
| `CY_RF_BW_MASTER.json` | Replication flow for customer and material master data (ODP_BW `$P`) |
| `CY_RF_BW_SALES.json` | Replication flow for the sales header and item ADSOs |
| `CY_AM_SALES.json` | The full model: 2 dimensions (with a derived material type and a hierarchy), a fact view (join, order filter, calculated margin, currency and unit semantics), and the analytic model (year variable, count distinct, calculated measures, exception aggregation, % of big orders) |

---

## What I've learned so far

- Datasphere doesn't replace BW thinking. It **rewards** it. Granularity, keys, units and currencies matter just as much.
- The hard parts aren't the clicks. They're the *why*: why Template Type, why 1:n, why exception aggregation.
- Building it yourself beats watching ten tutorials.

I'm still learning, and I post the journey step by step in my weekly LinkedIn series.

## Try it yourself

Datasphere → Data Builder → **Import → Import Objects from CSN/JSON File**.
The replication flows expect source connections named `S4_SOURCE` (S/4HANA) and `BW_SOURCE` (BW/4HANA), or you can remap them after import.

---

## Connect with me on LinkedIn

https://www.linkedin.com/in/cyril-adeyemi-333629104/?isSelfProfile=true

*Questions, feedback or your own war stories? I'd love to hear them. Connect with me on LinkedIn.*
