# From SAP BW to SAP Datasphere: My Learning Journey

Hi, I'm Cyril, an SAP BW consultant in Germany. I've worked with BW since 2020: data models, ABAP routines, process chains, queries and SAC.

🔗 **[Follow my journey on LinkedIn](https://www.linkedin.com/in/cyril-adeyemi-333629104/)**

Everyone in our world is talking about Datasphere. I didn't want to just read about it. I wanted to **build** with it, break things and understand *why* they broke. This repo is the result.

Everything here was built by me on training systems (S/4HANA 2025, BW/4HANA 2023 and a Datasphere system connected to them), with my own objects and my own data. The files are CSN/JSON exports of the Datasphere objects. They contain definitions only, no data.

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

## Project 5: From one plant to a schedule, with history

A load that works once isn't a pipeline. I wanted MD04 to run every morning, keep its history and cover more than one plant, all without me clicking "Run".

**The setup** (`MD04_HISTORY_AND_CHAINS.json`)

| Object | What it does |
|---|---|
| `RF_CY_MD04` → `CY_TAB_MD04_V2` | Plant 1710, **delete and load**: always the latest state |
| `RF_CY_MD04_1720` → `CY_TAB_MD04_1720` | The same for plant 1720 |
| `RF_CY_MD04_HIST` → `CY_TAB_MD04_H` | Plant 1710, **adds rows** on every run, so every snapshot date stays |
| `RF_CY_MD04_1720_HIST` → `CY_TAB_MD04_H_1720` | The same for plant 1720 |
| `TC_MD04_Current` | Task chain: the two current flows, daily at 05:00 (Berlin time) |
| `TC_MD04_History` | Task chain: the two history flows, daily at 06:00 (Berlin time) |
| `CY_VW_MD04_CURRENT` | Union All of the two current tables: one view for all plants |

**Why two flows for the same plant?**
My plant load had **Delete All Before Loading** switched on, so every run emptied the table first. A clean copy every time, but yesterday was gone. So I saved the flow as a new one, renamed the target to `CY_TAB_MD04_H`, switched the delete off and kept the same source and filter. After a second run the table held **11,696 rows**, exactly 2 x 5,848: two snapshot dates, nothing overwritten.

**What went wrong (and what it taught me):**

- 🔗 **Renaming the target detached my projection.** After the rename the flow showed *Projections (0)*. Without the filter it would have loaded every plant. I had to add the projection again, with `WERKS = 1710`.
- 🚫 **Two flows, one table: not allowed.** My plan was to send plant 1720 into the same history table. Deployment stopped: *a target table can't be used in two replication flows*. In BW I would load several sources into one ADSO with several DTPs. Here it is **one flow and one table per plant**, combined later in a view.
- 🕳️ **Plant 1720 loaded zero rows, with no error.** RSA3 returned nothing and MD04 was empty: the plant had materials, but no MRP data. The extractor can only return what MRP has planned. On my training system I created a purchase requisition (ME51N, 10 items) and then RSA3 showed **10 records** and the flow loaded 10 into 10.
- 🕐 **The schedule defaults to UTC.** 11:30 UTC is 12:30 in Berlin. I switched the time zone to Europe/Berlin. The clock popup did not keep my value, so I typed the time in.
- 📆 **One load per day.** Two loads on the same day gave plant 1720 **20 rows** for that date (my manual test plus the chain run). The snapshot date is the key, so a clean history needs one scheduled run per day.
- 👤 **No technical user.** The schedule owner list for a technical user was empty on my system, so I used my own user. In production a schedule should not depend on one person's account.

**It ran by itself.** The first scheduled run started at 06:00:41, without me, and finished 26 minutes later with all three flows completed. Next to the current state, the history grows by one snapshot a day.

**The union view.** `CY_VW_MD04_CURRENT` gives **5,858 rows**: 5,848 from plant 1710 plus 10 from plant 1720. The Impact and Lineage Analysis shows the whole path: two flows, two tables, one view.

---

## What I've learned so far

- Datasphere doesn't replace BW thinking. It **rewards** it. Granularity, keys, units and currencies matter just as much.
- Delta isn't magic. It's the same before/after-image logic as a BW change log, just picked up by a different tool.
- Graphical views are SQL underneath. Reading the generated SQL is the fastest way to understand what you built.
- A load that overwrites keeps no history. Keep a second flow that adds rows, with a snapshot date as part of the data.
- In Datasphere a table can serve only one replication flow, so it is one flow and one table per slice (here: per plant), joined later in a view.
- A pipeline that runs by itself still needs the boring things right: time zone, one run a day, and a technical user.
- The hard parts aren't the clicks. They're the *why*: why Template Type, why 1:n, why exception aggregation.
- Building it yourself beats watching ten tutorials.

I'm still learning, and I post the journey step by step in my weekly [LinkedIn series](https://www.linkedin.com/in/cyril-adeyemi-333629104/).

## Try it yourself

Datasphere → Data Builder → **Import → Import Objects from CSN/JSON File**.
The replication flows expect source connections named `S4_SOURCE` (S/4HANA) and `BW_SOURCE` (BW/4HANA), or you can remap them after import.
Import the objects of one file together, because the task chains refer to the flows.

---

*Questions, feedback or your own war stories? I'd love to hear them. [Connect with me on LinkedIn](https://www.linkedin.com/in/cyril-adeyemi-333629104/).*
