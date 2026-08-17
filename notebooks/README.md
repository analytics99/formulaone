# Formula 1 Data Notebooks

## `F1_All_Years_Data.ipynb`

Generalizes the single-season notebooks (`2024_Formula_1_Season_Data.ipynb`,
`2025_Formula_1_Season_Data.ipynb`) into one pipeline that pulls **every season**
available from the [OpenF1 API](https://openf1.org/), not just one hardcoded year,
and writes the result into a Google Sheet.

### One-time setup

- **Target spreadsheet:** this notebook writes to
  [this existing sheet](https://docs.google.com/spreadsheets/d/1FHF1yWmg5sv0e2WGVafTiM-zfzfaxvleRxG28nvjd9U)
  (`SPREADSHEET_ID` in the Configuration cell) — it is not created for you. Make sure
  it's shared with **Editor** access to whichever Google account you authenticate as
  below, or update `SPREADSHEET_ID` to point at a different sheet you do have access to.
- **Running in Google Colab (recommended):** nothing else to set up. The notebook
  authenticates via Colab's built-in Google sign-in
  (`google.colab.auth.authenticate_user()`) — no API key, token, or secret to create
  or manage.
- **Running outside Colab** (local Jupyter, CI, etc.): either point
  `GOOGLE_APPLICATION_CREDENTIALS` at a service account JSON file with edit access to
  the target spreadsheet, or leave it unset — the notebook falls back to
  `gspread.oauth()`, a one-time interactive browser sign-in whose token gets cached
  locally for future runs.

### Running it

Run the cells top to bottom:

1. **Pre-Requisites** — auto-installs any missing packages (`pandas`, `pandasql`,
   `requests`, `gspread`) and imports everything the notebook needs.
2. **Configuration** — set `YEARS_TO_INCLUDE` (`None` = all seasons, or a list like
   `[2023, 2024, 2025]`), and the target spreadsheet name / tab names.
3. **Authenticate & Open the Spreadsheet** — signs in and opens `SPREADSHEET_NAME`,
   creating it if it doesn't exist yet.
4. **Helper Functions** — fetch/format/push helpers.
5. **Fetch Core Reference Data** — sessions, meetings, drivers, session results,
   starting grid, pit stops. These OpenF1 endpoints return their full history in one
   unfiltered call, so this step runs once regardless of how many seasons are kept.
6. **Scope to Selected Years** — applies `YEARS_TO_INCLUDE`.
7. **Fetch Laps** — looped per race session (laps are too high-volume for one call).
8. **Shape Data for Google Sheets** — builds each table's identifying key column and
   the header names each tab will use.
9. **Exploratory SQL Joins** — season standings and lap analysis across all years
   (via `pandasql`), generalizing the original notebooks' single-season queries.
10. **Push to Google Sheets** — full-refreshes each tab (safe to re-run).
11. **Summary** — row counts per tab, plus the spreadsheet URL.

### Google Sheet

Data lands in [this spreadsheet](https://docs.google.com/spreadsheets/d/1FHF1yWmg5sv0e2WGVafTiM-zfzfaxvleRxG28nvjd9U),
with 7 tabs:

| Tab | Identifying column | Source |
|---|---|---|
| Meetings | Meeting Key | `/meetings` |
| Sessions | Session Key | `/sessions` |
| Drivers | Driver Session Key (`{session_key}_{driver_number}`) | `/drivers` |
| Session Results | Result Key (`{session_key}_{driver_number}`) | `/session_result` |
| Starting Grid | Grid Key (`{session_key}_{driver_number}`) | `/starting_grid` |
| Laps | Lap Key (`{session_key}_{driver_number}_{lap_number}`) | `/laps` |
| Pit Stops | Pit Key (`{session_key}_{driver_number}_{lap_number}`) | `/pit` |

Every tab also carries `Session Key`/`Meeting Key`/`Driver Number` columns and a
`Year` column, so filters and lookups (e.g. "all of 2024", or `VLOOKUP`ing Laps back
to Sessions/Meetings) work directly in Sheets without needing separate join tables.

**Refresh semantics differ from the previous Airtable version:** Sheets has no
native per-row upsert API, so each push is a **full refresh** per tab (clear → header
→ append) rather than a row-level upsert. Re-running the notebook always leaves each
tab matching the current pull exactly — idempotent, but any manual edits made
directly in the sheet get overwritten on the next run.

**Note on volume:** Laps and Pit Stops are the highest-row-count tabs (tens of
thousands of rows per season). If a full multi-year pull is more than you want in the
spreadsheet, set `PUSH_LAPS_TO_SHEETS = False` and/or `PUSH_PITS_TO_SHEETS = False` in
the Configuration cell to keep those two only in the notebook's dataframes.
