# From SAP BW to SAP Datasphere: My Learning Journey

Hi, I'm Cyril, an SAP BW consultant in Germany. I've worked with BW since 2020: data models, ABAP routines, process chains, queries and SAC.

🔗 **[Follow my journey on LinkedIn](https://www.linkedin.com/in/cyril-adeyemi-333629104/)**

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

## Project 3: Same data, written in SQL

After clicking my fact view together graphically, I opened **Preview SQL** to see what Datasphere had generated. It showed my join, my filter and my margin calculation, nested from the inside out like an onion. So I started writing views myself.

I built every query step by step and checked the result after each line.

**View 1: how many orders?** `CY_SQL_NO_ORDERS`
```sql
SELECT COUNT("DOC_NUMBER") AS "No_of_Orders"
FROM "CY_TAB_SALESHEAD"
WHERE "DOC_TYPE" = 'TA'
```
It returns **69**, the same number as my fact view and the count distinct measure in the analytic model. That's three layers agreeing, checked with three lines of SQL.

**View 2: annual sales, per currency.** `CY_SQL_ANNUAL_SALES`
```sql
SELECT "HEAD"."CALYEAR"        AS "Year",
       "ITEM"."DOC_CURRCY"     AS "Currency",
       SUM("ITEM"."NET_VALUE") AS "Net_Value"
FROM "CY_TAB_SALESHEAD" AS "HEAD"
JOIN "CY_TAB_SALESITEM" AS "ITEM"
  ON "HEAD"."DOC_NUMBER" = "ITEM"."DOC_NUMBER"
WHERE "HEAD"."DOC_TYPE" = 'TA'
GROUP BY "HEAD"."CALYEAR", "ITEM"."DOC_CURRCY"
```
The course exercise sums net value per year and ignores the currency. My data has two sales organisations, one in EUR and one in USD, and 100 € + 100 $ isn't 200 of anything. So I grouped by currency as well.

**Small things that tripped me up:**
- `'DOC_NUMBER'` in single quotes is a text value, not a column. Names need double quotes: `"DOC_NUMBER"`.
- A `JOIN` without `ON` isn't finished. The error "expecting 'on'" says exactly that.
- An aggregated view (only 4 rows) is fine for a check, but a poor base for an analytic model. Keep the fact at item level and let the model do the summing, as you would in BW.

**View 3: the whole fact view, rebuilt in SQL.** `CY_SQL_FACT_SALES`
The same as my graphical fact view: header + item join, order filter, calculated margin, item level, with no aggregation. It returns **183 rows**, exactly the same as the graphical version. One stubborn parse error ("expecting ',' or 'from'") disappeared once I deleted everything and pasted a clean copy. Something invisible had crept in.

---

## Project 4: Delta loads, watching a change travel from BW to Datasphere

My first flows were "Initial Only". In real projects you don't reload everything every night; you move only what changed. So I tested delta end to end.

**The setup** (`CY_RF_BW_SALES_DELTA.json`)
- Source: the sales header ADSO in BW/4HANA (a Standard ADSO, so it has a change log)
- Load type **Initial and Delta**, into a **new** target table `CY_TAB_SALESHEAD_D` with **Delta Capture** switched on. Datasphere adds two columns: `Change_Type` and `Change_Date`.
- Initial load: **80 rows in 52 seconds**. After that the run never "completes": run type *Permanent*, duration ∞, and it checks for changes every hour.

**The test:** three changes in BW
| Document | Change | Business story |
|---|---|---|
| 0090000005 | AG → TA | a quotation became an order |
| 0090000002 | customer 127 → 201 | customer corrected |
| 0090000081 | new | a new order |

**What happened**
- The BW change log showed **5 rows**: a before image (X) and an after image for each change, plus one new record (N).
- At the next delta window, Datasphere picked up **exactly 5 operations**.
- The target table ended with **81 rows** (80 + 1). The two changes were applied as updates, not extra rows.

**Reading the result**
| Column | From | Values I saw |
|---|---|---|
| `ODQ_CHANGEMODE` | BW / ODQ | C = created, U = updated |
| `Change_Type` | Datasphere | L = initial load, A = upsert (changed), I = insert (new) |

**What tripped me up**
- **Order matters.** Delta works like a bookmark: start the initial load first, then change the source. Change it before, and your delta is empty.
- **My first BW load brought 80 rows instead of 3.** In BW/4HANA the DTP keeps its own copy of the flat-file path. I had changed the DataSource, but the DTP still read the old file.
- `Change_Date` is stored in **UTC**, so it looked two hours "wrong" until I realised.
- "Retrying" in the monitor doesn't mean failure. It means "waiting for the next delta window".

---

## What I've learned so far

- Datasphere doesn't replace BW thinking. It **rewards** it. Granularity, keys, units and currencies matter just as much.
- Delta isn't magic. It's the same before/after-image logic as a BW change log, just picked up by a different tool.
- Graphical views are SQL underneath. Reading the generated SQL is the fastest way to understand what you built.
- The hard parts aren't the clicks. They're the *why*: why Template Type, why 1:n, why exception aggregation.
- Building it yourself beats watching ten tutorials.

I'm still learning, and I post the journey step by step in my weekly [LinkedIn series](https://www.linkedin.com/in/cyril-adeyemi-333629104/).

## Try it yourself

Datasphere → Data Builder → **Import → Import Objects from CSN/JSON File**.
The replication flows expect source connections named `S4_SOURCE` (S/4HANA) and `BW_SOURCE` (BW/4HANA), or you can remap them after import.

---

*Questions, feedback or your own war stories? I'd love to hear them. [Connect with me on LinkedIn](https://www.linkedin.com/in/cyril-adeyemi-333629104/).*
