# Formula 1 Data Notebooks

## `F1_All_Years_Data.ipynb`

Generalizes the single-season notebooks (`2024_Formula_1_Season_Data.ipynb`,
`2025_Formula_1_Season_Data.ipynb`) into one pipeline that pulls **every season**
available from the [OpenF1 API](https://openf1.org/), not just one hardcoded year,
and upserts the result into Airtable.

### One-time setup

1. Create an Airtable Personal Access Token (PAT) at
   [airtable.com/create/tokens](https://airtable.com/create/tokens) with
   `data.records:read` and `data.records:write` scopes, granted access to the
   `Formula 1 Multi-Year Data` base (`appmZPMPRAjoKM5He`).
2. In Colab, click the **key icon (🔑)** in the left sidebar → **Add new secret** →
   name it `AIRTABLE_API_KEY`, paste the PAT as the value, enable **Notebook access**.
   The token then never touches the notebook file or git history — the notebook reads
   it at runtime via `google.colab.userdata.get("AIRTABLE_API_KEY")`.
   - Not on Colab? Export `AIRTABLE_API_KEY` as an environment variable before
     launching Jupyter, or the notebook will fall back to a hidden `getpass` prompt.

### Running it

Run the cells top to bottom:

1. **Pre-Requisites** — auto-installs any missing packages (`pandas`, `pandasql`,
   `requests`) and imports everything the notebook needs.
2. **Configuration** — set `YEARS_TO_INCLUDE` (`None` = all seasons, or a list like
   `[2023, 2024, 2025]`), then resolves the Airtable token from the Colab Secret set
   up above (env var / prompt as fallbacks for non-Colab environments).
3. **Helper Functions** — fetch/format/upsert helpers.
4. **Fetch Core Reference Data** — sessions, meetings, drivers, session results,
   starting grid, pit stops. These OpenF1 endpoints return their full history in one
   unfiltered call, so this step runs once regardless of how many seasons are kept.
5. **Scope to Selected Years** — applies `YEARS_TO_INCLUDE`.
6. **Fetch Laps** — looped per race session (laps are too high-volume for one call).
7. **Shape Data for Airtable** — builds the primary/composite keys and column names
   each Airtable table expects.
8. **Exploratory SQL Joins** — season standings and lap analysis across all years
   (via `pandasql`), generalizing the original notebooks' single-season queries.
9. **Push to Airtable** — upserts each table (safe to re-run).
10. **Summary** — row counts per table.

### Airtable base

Data lands in the **Formula 1 Multi-Year Data** base
(`https://airtable.com/appmZPMPRAjoKM5He`), which has 7 tables:

| Table | Primary Field | Source |
|---|---|---|
| Meetings | Meeting Key | `/meetings` |
| Sessions | Session Key | `/sessions` |
| Drivers | Driver Session Key (`{session_key}_{driver_number}`) | `/drivers` |
| Session Results | Result Key (`{session_key}_{driver_number}`) | `/session_result` |
| Starting Grid | Grid Key (`{session_key}_{driver_number}`) | `/starting_grid` |
| Laps | Lap Key (`{session_key}_{driver_number}_{lap_number}`) | `/laps` |
| Pit Stops | Pit Key (`{session_key}_{driver_number}_{lap_number}`) | `/pit` |

Every table also carries `Session Key`/`Meeting Key`/`Driver Number` foreign keys and
a `Year` field, so downstream joins/filters (e.g. "all of 2024", or joining Laps back
to Sessions/Meetings) work directly in Airtable without needing native record links.

Pushes use Airtable's upsert API (`performUpsert`, keyed on each table's primary
field), so re-running the notebook updates existing rows instead of duplicating them.

**Note on volume:** Laps and Pit Stops are the highest-row-count tables (tens of
thousands of rows per season). If your Airtable plan has a low per-base record limit,
set `PUSH_LAPS_TO_AIRTABLE = False` and/or `PUSH_PITS_TO_AIRTABLE = False` in the
Configuration cell to keep those two only in the notebook's dataframes.
