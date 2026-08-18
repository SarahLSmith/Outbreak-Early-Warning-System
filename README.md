# UKHSA Local Outbreak Alerts

Keeps an eye on COVID-19 case numbers across local authorities (roughly,
council areas) in the UK, and flags when a specific area is seeing a
sharp week on week increase. Built as the first stage of a bigger idea,
giving clinicians a heads up that something might be spreading locally
before a patient's symptoms alone would suggest it.

## Quickstart

For anyone already comfortable with Databricks, Unity Catalog, and the
medallion pattern, here's the whole thing in a few lines.

- Stack: Azure Databricks, Unity Catalog, ADLS Gen2 (bronze/silver/gold
  containers), Delta Lake. Storage access via a Managed Identity backed
  storage credential, `ukhsa-managed-identity`, no mount points.
- Four notebooks, run in order:
  1. `01_Bronze_Ingestion` - pulls raw UKHSA JSON per local authority
     per metric, paginated, written to `ukhsa_bronze`.
  2. `02_Silver_Cleaning` - explodes/flattens/types Bronze, drops rows
     still in `in_reporting_delay_period`, dedupes on
     `geography_code + metric + date`, writes Delta to `ukhsa_silver`.
  3. `03_Gold_Alerts` - 7 day rolling sum vs prior 7 days per geography,
     `percent_change`, threshold at `THRESHOLD_PERCENT = 25.0`, writes
     Delta to `ukhsa_gold`.
  4. `04_Query_and_Visualise` - registers Gold as a temp view, runs SQL
     against it, charts top movers.
- Before first run, confirm storage's wired up: `SHOW STORAGE
  CREDENTIALS;` should list `ukhsa-managed-identity`, pointing at the
  `ukhsa-access-connector` Access Connector. If external locations
  `ukhsa_bronze` / `ukhsa_silver` / `ukhsa_gold` don't exist yet, see
  Storage setup below.
- Swap `<storage-account-name>` at the top of each notebook before
  running.
- Only COVID-19 is wired up currently, it's the only topic in this
  theme with Lower Tier Local Authority level data, see Limitations.

Everything below this point is the same information written out for
somebody without a data engineering background.

## What this actually does, in plain terms

Every day, the UK Health Security Agency (UKHSA) publishes how many
COVID-19 cases were confirmed in each local authority. This project
downloads that data automatically, tidies it up, then compares the most
recent week's case count against the week before it, for every area in
the country. If an area's cases have jumped by more than 25% week on
week, it gets flagged.

The process happens in three stages, each one handing its result to the
next, a bit like an assembly line.

**Stage 1, collect.** Download the raw numbers from UKHSA exactly as
they publish them, and keep a copy, unedited.

**Stage 2, tidy up.** Take that raw download, fix anything that's
formatted awkwardly, remove numbers that are still likely to be revised
by UKHSA later, and end up with one clean, reliable table.

**Stage 3, compare and flag.** For every area, work out whether this
week's cases are meaningfully higher than last week's, and produce a
simple list saying which areas are currently a concern.

A fourth, final step then looks at that list properly, answering
specific questions about it and drawing a chart, so the result is
something you can actually look at rather than just a table of numbers.

# Examples
Here is an example of a notebook cell making a query and returning the table below as a result

```
# Query 1: which local authorities are currently flagged as a spike?
 
spark.sql("""
SELECT geography, current_week_cases, previous_week_cases, percent_change
FROM covid19_outbreak_alerts
WHERE alert_status = 'SPIKE'
ORDER BY percent_change DESC
""").show(50, truncate=False)
```
<div align="center">

![A table of query results showing local authorities and their case data](https://github.com/user-attachments/assets/a661f9f3-8423-4569-940b-d13b43f22db9)

*Example output table*

</div>

<div align="center">

![A bar chart showing the top local authorities by percentage change in cases](https://github.com/user-attachments/assets/80ed14b1-9320-4235-a34c-11c8085d3bd3)

*Example chart*

</div>

## Where this lives and what you need

This project doesn't run on a personal computer the way a normal
program would. It runs inside **Databricks**, which is an online
platform for working with large amounts of data. Think of it as a
website where code runs on powerful computers in the cloud instead of
on your own laptop, since the data involved here is too large and
needs too much processing power for a normal computer to handle
comfortably.

The code itself is organised into four **notebooks**. A notebook is
just a file that mixes code with explanations, and runs one chunk at a
time rather than all at once, so you can see what each step produces
before moving to the next. If you've never used one before, imagine a
document where some paragraphs are instructions a computer can run,
and clicking a small play button next to one runs just that part.

<div align="center">

![A Databricks notebook cell with its play button visible](https://github.com/user-attachments/assets/67e03a73-ace3-469a-bad4-e2cb86827394)

*Example cell in a notebook (ignore the number 6)*

</div>

To actually use this project yourself, you'd need access to an Azure
account (Microsoft's cloud service, which is what Databricks in this
case runs on top of), a Databricks workspace already set up inside it,
and a storage account where the data gets kept. Setting all of that up
from nothing is a fair bit of work in its own right, and isn't
something this README walks through, it assumes that groundwork is
already in place and focuses on what the notebooks themselves do.

## Getting the notebooks into Databricks

If you already have access to the Databricks workspace this project
was built in, the four notebooks will simply be sitting there, ready
to open and run, no download needed.

If instead you've been given the code as plain files (for example, from
a GitHub page, download the code as a ZIP using the green Code button,
then unzip it), you'll need to bring each one into Databricks yourself.

1. Open your Databricks workspace in a web browser.
2. In the left sidebar, click **Workspace**, then navigate to wherever
   you'd like this project to live.
3. Click the three dots icon, then choose
   **Import**.
4. Select the notebook file from your computer, e.g.
   `01_Bronze_Ingestion.py`, and confirm.
5. Repeat for all four files.

Once imported, each one opens and runs the same way as any other
Databricks notebook.

<div align="center">

![The Workspace sidebar with the Create and Import menu open](https://github.com/user-attachments/assets/67621c4f-307e-4a02-aba1-0a1e3fd508af)

*Import location*

</div>

## Running it

**1. Confirm storage access is set up.** Before any of this can save or
read data, Databricks needs permission to read and write to the
storage account. If this has already been configured by whoever set up
the workspace, you don't need to do anything here. If you're not sure,
open any notebook, create a new code cell, and run `SHOW STORAGE
CREDENTIALS;`, if you see something listed, you're set, if the result
is empty, this needs setting up first, that's a more technical step
covered later in this document.

**2. Run the notebooks in order.** Open `01_Bronze_Ingestion` first.
Each notebook is split into small sections, run each one from top to
bottom by clicking the play button beside it, then wait for it to
finish before moving to the next section. Once that notebook is done,
move on to `02_Silver_Cleaning`, then `03_Gold_Alerts`, then finally
`04_Query_and_Visualise`.

**3. Look at the result.** The last notebook, `04_Query_and_Visualise`,
is where you'll actually see something meaningful, a short list of
which areas currently look concerning, and a chart showing the biggest
changes. That's the one worth opening if someone asks "so what does
this actually tell us".

## Understanding the structure (medallion architecture)

The three stage set up used here, collect, tidy up, compare and flag,
is a common pattern when working with data at this scale, often called
a "medallion architecture", named after the three tiers, bronze,
silver and gold, like medals.

**Bronze** is the raw, unedited data, kept exactly as it arrived. If
something later turns out to be wrong, this is what gets checked
first, since nothing has been changed or removed from it yet.

**Silver** is the cleaned up version, still just the facts, but
reliable and consistently formatted, ready to be used for real
analysis.

<div align="center">

![A messy pile of overlapping shapes representing raw Bronze data, tidied into one neat row representing Silver data](https://github.com/user-attachments/assets/b5158c00-2eb6-4582-b6e8-4c4ddbf94bd1)

*Bronze data starts out messy and mixed together, Silver tidies it into one clear row.*

</div>

**Gold** is the finished answer to the actual question being asked, in
this case, which areas are spiking. This is the layer meant for a
dashboard or a person to look at directly, everything unnecessary has
been stripped away.

<div align="center">

![Two bars comparing this week to last week, this week's bar crossing a 25 percent threshold line and raising a red spike flag](https://github.com/user-attachments/assets/6416e675-8bf0-462d-8914-0cfe8fc86821)

*Silver to Gold, comparing this week to last week and raising a flag*

</div>

Keeping these as three separate stages, rather than one big step, means
that if the underlying question ever changes, for example wanting a
different threshold than 25%, only the last stage needs to be redone,
the raw data and the cleaned data underneath it don't need to be
touched or re-downloaded.

## More detail, for anyone technical

This section assumes familiarity with Databricks, Spark, and Azure. If
that's not you, everything above this point is what you need.

### How it works

1. `01_Bronze_Ingestion` discovers every local authority available for
   COVID-19 from the UKHSA API, then requests daily case data for each
   one individually, since the API doesn't support pulling multiple
   areas in a single call. Every page of every response is saved as
   its own JSON file, unmodified.
2. `02_Silver_Cleaning` reads every one of those files, explodes the
   nested results into individual rows, and selects out the fields
   that matter, casting dates and case counts to proper types along
   the way. Rows still inside UKHSA's reporting delay window are
   dropped, since those numbers commonly get revised upward later and
   would risk a false spike. Duplicate rows, which can build up if
   ingestion is rerun on different days, are collapsed down to one row
   per area per date.
3. `03_Gold_Alerts` works out the most recent date actually present in
   the data, then sums cases for that 7 day window and the 7 days
   before it, for every area. The percentage change between the two is
   calculated, and anything over the threshold, 25% by default, is
   marked as a spike. The raw counts for both weeks are kept alongside
   the percentage, so a jump from 100 to 200 cases can be told apart
   from a jump from 1 to 2, both of which are technically a 50-100%
   change, only one of which is worth worrying about.
4. `04_Query_and_Visualise` registers the alert table so it can be
   queried with real SQL rather than DataFrame code, then produces a
   bar chart of the areas with the largest change, spikes shown in a
   different colour from the rest, with a line marking the threshold.

### Storage setup

Bronze, silver and gold each map to their own container in ADLS Gen2,
and each container is registered in Unity Catalog as an external
location, `ukhsa_bronze`, `ukhsa_silver`, and `ukhsa_gold`. All three
point at the same storage credential, an Azure Managed Identity with
Storage Blob Data Contributor on the storage account. This means the
notebooks write directly via `abfss://` paths, no mount point
involved.

### Rerunning safely

Bronze ingestion is idempotent within a single day, running it again
today just overwrites today's files. Running it again tomorrow adds a
new dated set of files alongside older ones, since UKHSA's historical
numbers for past dates don't generally change, this is treated as
intentional, an audit trail of what the API returned on each day
rather than something to avoid. Silver and Gold both overwrite their
entire output on every run, so rerunning either one is always safe and
never causes duplication.

### Project layout

- `01_Bronze_Ingestion.py` - pulls and stores raw UKHSA data.
- `02_Silver_Cleaning.py` - cleans, types and dedupes it.
- `03_Gold_Alerts.py` - calculates spikes.
- `04_Query_and_Visualise.py` - SQL queries and a chart against the
  result.

## Limitations

- Currently only tracks COVID-19. Other respiratory topics UKHSA
  publishes, like Influenza, aren't available at local authority level
  at all, only down to UKHSA Region, so they'd need a different
  geography setting and can't be added into this pipeline as-is.
- The 25% spike threshold is a single configurable number in
  `03_Gold_Alerts`, not something that's been clinically validated,
  it's a starting point for testing the pipeline, not a real alerting
  rule yet.
- There's currently no floor on case counts before a spike can be
  flagged, so a very small area going from 1 case to 2 can technically
  register the same percentage change as a much larger, more
  meaningful jump. The raw counts are shown alongside the percentage
  specifically so this can be judged by eye, but the pipeline itself
  doesn't filter it out automatically yet.
- Data doesn't currently reach a normal relational database, it stays
  in Delta tables queried directly from Databricks. Pushing the Gold
  table out to a database that a separate dashboard could query
  directly is a planned next step, not built yet.
- This is stage one of a larger idea, correlating local outbreak spikes
  with individual patient symptoms to support clinical diagnosis. That
  second stage hasn't been started.
