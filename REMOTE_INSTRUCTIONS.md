# REMOTE_INSTRUCTIONS — National Chromebook Rollouts

You are running on a 6-hour cron. The prompt that triggered you is identical
every run; **all state derives from the files in this folder**, never from
the prompt or any prior memory. Read this file in full, then act.

## What this project is

Identify when each of the largest US public school districts adopted 1:1
student devices (Chromebooks, iPads, laptops). The CSV
`national_chromebook_rollouts.csv` is the canonical output. Each run, you
will research and append the next 300 districts in size order.

District size ranking is by **mean grade-3 enrollment** computed from SEDA
data. The pre-computed ranking is `all_districts_ranked.tsv` — TSV with
columns: `mean_grade3_enrollment`, `leaid`, `leaname`, `state` — sorted
descending. (CSV ranks 1-30 used a different size metric — cellcount —
see CAVEAT (A); ranks 31+ use mean-grade-3.)

## Files in this folder

| File | Purpose | Read/Write |
|------|---------|------------|
| `REMOTE_INSTRUCTIONS.md` | this file | read |
| `national_chromebook_rollouts.csv` | canonical output, **16 cols**, all-quoted UTF-8 with BOM | read + append |
| `all_districts_ranked.tsv` | top ~5000 districts by mean grade-3 enrollment | read |
| `research_strategy.txt` | methodology (Indiana origin; FB section is deferred — see below) | read only |
| `extraneous_notes.txt` | per-session history, leaid lists, early-adopter notes | read only (cross-reference) |

Dedup uses the `leaid` column in the CSV — every row has its `leaid`
populated. There is no separate `covered_leaids.txt`.

## STOP conditions — check these first, terminate if any are true

1. `national_chromebook_rollouts.csv` is missing, malformed, or has fewer
   than 700 rows. Something is wrong; do not append. Report and exit.
2. `wc -l national_chromebook_rollouts.csv` minus 1 (header) is **>= 6000**.
   Project breadth target reached; stop and report. Mike will redirect
   you to phase 2 (FB-search second pass) when he wants it.
3. After dedup, you find **fewer than 300** uncovered districts in
   `all_districts_ranked.tsv`. Report what's left and exit; Mike will
   regenerate the ranking from a larger SEDA slice.
4. A previous run is in progress, signalled by a file named `LOCK` in
   this folder. Exit immediately — do NOT remove the LOCK.

If none trigger, create `LOCK` (any content) at the start and `rm LOCK` at
the end (also on any error path you can reach). This protects against
accidental concurrent runs.

## Phase guidance (read every run, do not deviate)

- **Default model for parallel research agents: `sonnet`.** Per Mike's
  2026-05-05 head-to-head, Haiku quality is too low; Opus is overkill for
  breadth.
- **Do NOT do Facebook profile-search work.** The current phase is
  breadth — get a web-search row populated for as many districts as
  possible. FB searches are a deferred second pass that Mike will trigger
  later. Set `fb_search="0"` for every new row.
- **Do NOT add columns to the CSV** or change quoting style.
- **Do NOT touch CSV rows that already exist.** Only append.

## CSV schema (16 columns, all-quoted)

```
district_name,state,city,year_started,month_started,one_to_one,grade_levels,
device_type,notes,source_url,source_2,seda_rank,seda_row_count,web_search,
fb_search,leaid
```

- `year_started` — 4-digit year, or `2020 (by)` / `2014 (by)` etc. for
  upper bound, or blank if no evidence at all.
- `month_started` — month name, `Spring` / `Fall`, or blank.
- `one_to_one` — `Yes` / `Yes (phased)` / `Yes (likely)` / `Partial`.
- `seda_rank` — INTEGER rank position (e.g., 701, 702...). NOT enrollment.
- `seda_row_count` — the mean-grade-3 enrollment number from
  `all_districts_ranked.tsv` column 1 for that leaid.
- `web_search` — always `1` for new rows.
- `fb_search` — always `0` for new rows in this phase.
- `device_type` lives in EXACTLY ONE field. Do NOT duplicate it into the
  next field. (Past Sonnet bug: Dougherty Co GA was emitted with 16
  fields because device_type appeared twice — and now we have 16
  legitimate fields, so a duplicate would push it to 17.)
- `leaid` — 7-digit zero-padded SEDA leaid. Carries through verbatim
  from `all_districts_ranked.tsv` column 2 for the picked district.

## Step-by-step workflow

### Step 1 — Verify state and acquire lock

```powershell
# from the remote/ folder
$csv = Import-Csv national_chromebook_rollouts.csv
$rows = $csv.Count
$maxRank = ($csv | Measure-Object -Property seda_rank -Maximum).Maximum
$ranks = $csv | ForEach-Object { [int]$_.seda_rank }
$missing = (1..$maxRank) | Where-Object { $_ -notin $ranks }
$dups = $ranks | Group-Object | Where-Object { $_.Count -gt 1 }
$cols = $csv[0].PSObject.Properties.Name.Count
"rows=$rows maxRank=$maxRank cols=$cols missing=$($missing.Count) dups=$($dups.Count)"
```

- If `rows < 700`, `cols != 16`, `missing.Count > 0`, or `dups.Count > 0`
  → STOP (condition 1).
- If `maxRank >= 6000` → STOP (condition 2).
- If a `LOCK` file exists → STOP (condition 4).
- Otherwise: `New-Item LOCK -Value (Get-Date).ToString()`.

### Step 2 — Take a backup

```powershell
Copy-Item national_chromebook_rollouts.csv ("national_chromebook_rollouts_${maxRank}rows.bak.csv")
```

### Step 3 — Pick the next 300 uncovered districts

Build the covered-leaid set from the CSV's leaid column, then dedup:

```bash
# bash (works from this folder)
# Extract leaids already in the CSV (skip the header row)
awk -F'","' 'NR>1 { gsub(/"/,"",$16); gsub(/\r/,"",$16); print $16 }' national_chromebook_rollouts.csv \
    | sort -u > /tmp/covered.txt

# Filter ranking against covered set, take next 300
awk -F'\t' 'NR==FNR { gsub(/\r/,"",$1); covered[$1]=1; next } { gsub(/\r/,"",$2); if(!covered[$2] && $2!="4702940") print }' \
    /tmp/covered.txt all_districts_ranked.tsv \
    | head -300 > /tmp/next_batch.tsv

wc -l /tmp/next_batch.tsv  # should be 300
```

Each line of `/tmp/next_batch.tsv` is `mean_grade3 \t leaid \t leaname \t state`.
Assign rank numbers `maxRank+1` through `maxRank+300` in order. Split into
6 batches of 50 for parallel agents:

| Batch | Ranks |
|-------|-------|
| A | maxRank+1 .. maxRank+50 |
| B | maxRank+51 .. maxRank+100 |
| C | maxRank+101 .. maxRank+150 |
| D | maxRank+151 .. maxRank+200 |
| E | maxRank+201 .. maxRank+250 |
| F | maxRank+251 .. maxRank+300 |

### Step 4 — Sanity-check the batch

For each district in `/tmp/next_batch.tsv`, do a quick name+state collision
check against the CSV: if a district with the same `(district_name, state)`
already exists in the CSV (case-insensitive, trimmed), the leaid escaped
the dedup. Drop it from the batch and pick the next uncovered one to keep
the batch size at 300. Log the collision.

Watch for known near-collision pitfalls (different leaids, similar names):
multiple "Independence", "Lincoln County", "Jackson", "Newton", "Aurora",
"Auburn", "Roanoke", "Edgewood ISD", etc. The state code disambiguates;
trust it.

Also filter out `4702940` (defunct pre-merger Memphis City SD; appears
in SEDA panel due to old enrollment data; should never be researched).

### Step 5 — Spawn 6 parallel research agents

Use the Agent tool with `subagent_type: general-purpose` and
`model: sonnet`. Send all 6 in a single tool-use block (parallel). Use the
prompt template at the bottom of this file. **Do not run the agents
sequentially — that wastes wall-clock time.**

If your environment denies agent file-write (sessions 4 and 7 both saw
this), agents will fall back to returning CSV inline in their final
summary. Plan for it: parse each agent's final response and write the
staging files yourself.

### Step 6 — Validate staging files

Each `staging_<N1>-<N2>.csv` must:

- Have exactly 50 lines (no header).
- Have exactly **16 fields** per line, comma-separated, all-quoted.
- Have `seda_rank` in column 12 matching the assigned rank, NOT the enrollment.
- Have `web_search="1"` and `fb_search="0"` in columns 14 and 15.
- Have `leaid` in column 16 matching what was assigned in the batch.

```bash
# field-count check
awk -F'","' 'NF != 16 { print FILENAME":"NR": "NF" fields" }' staging_*.csv
# rank coverage
awk -F'","' '{print $12}' staging_*.csv | sed 's/"//g' | sort -un | wc -l  # = 300
# leaid uniqueness vs already-covered
awk -F'","' '{gsub(/"/,"",$16); print $16}' staging_*.csv | sort -u > /tmp/new_leaids.txt
comm -12 /tmp/covered.txt /tmp/new_leaids.txt  # should be empty (no collisions)
```

If validation fails, fix the file (rewrite the offending row from the
agent's notes) before proceeding. Past common bugs:

- `seda_rank` and `seda_row_count` columns swapped (both contain enrollment).
- `device_type` duplicated, pushing the row to 17 fields.
- Inconsistent quoting (e.g., `,Yes (phased),` instead of `,"Yes (phased)",`)
  — only a problem if a field contains a comma; otherwise tolerable.
- `leaid` missing or wrong (agent forgot column 16); cross-check against
  the assignment in `/tmp/next_batch.tsv`.

### Step 7 — Append to canonical CSV

```bash
cat staging_*.csv >> national_chromebook_rollouts.csv
```

Verify:

```powershell
$csv = Import-Csv national_chromebook_rollouts.csv
"rows=$($csv.Count) maxRank=$(($csv | Measure-Object -Property seda_rank -Maximum).Maximum)"
$blanks = ($csv | Where-Object { $_.leaid -eq '' }).Count
"blank leaids after append: $blanks"  # should still be 0
```

Should be `oldRows + 300` and `oldMaxRank + 300`, with no new blank
leaids.

### Step 8 — Release lock and report

```bash
rm LOCK
```

Final report (terse, single response):

- New rows added: 300. Range: ranks `${maxRank+1}` to `${maxRank+300}`.
- Distribution: `<X>` firm pre-pandemic / `<Y>` firm 2020 / `<Z>` post-2020 firm / `<W>` "(by)" upper-bound / `<B>` blank.
- Notable findings: any pre-2014 firm-year adopters, any districts that
  surprised you (very late adopters, partial rollouts, etc.).
- Any agent failures or fallbacks used.

Do NOT update REMOTE_INSTRUCTIONS.md or extraneous_notes.txt — those are
canonical sources Mike maintains.

### Step 9 — Push branch and merge into main

After releasing the lock, push your work and merge into main.

**9a. Verify local branch has the expected commits before pushing:**

```bash
git log --oneline -6
```

Confirm you see your 8 commits (LOCK+backup, 6× staging files, append).
If the branch tip is at the same commit as `origin/main` (e.g. the branch
was silently reset), recover from the reflog:

```bash
git reflog --oneline | head -10
# Find the "Append ranks …" commit hash, then:
git reset --hard <that-hash>
```

**9b. Push the branch (force if needed after a reset recovery):**

```bash
git push -u origin <your-branch>
# If the remote already has the branch in a reset state, force-push:
git push -f origin <your-branch>
```

The git remote URL uses a local proxy whose **port changes every session**
(`git remote -v` will show the current port). This is normal — git uses
the configured remote automatically.

**9c. Merge into main and push:**

```bash
git checkout main
git merge <your-branch> --no-ff -m "Merge ranks <N1>-<N2>: 300 new district 1:1 device rollout rows"
git push origin main
```

If `git merge` says "Already up to date", your branch tip is probably
still pointing at `origin/main` — go back to step 9a and recover from
the reflog first.

**9d. Confirm:**

```bash
git log --oneline origin/main | head -3
```

The top commit should be the merge commit you just made.

## Caveats (read once, internalize)

(A) `seda_row_count` for CSV ranks 1-30 uses an OLDER metric (cellcount,
    NYC=13209). Ranks 31+ use mean grade-3 enrollment from SEDA. Don't
    compare values across the rank-30 boundary. New rows (rank 31+) all
    use the mean-grade-3 metric — match the value from
    `all_districts_ranked.tsv` column 1.

(B) Some districts (esp. UT — Davis, Granite, Mesa) are genuinely hard to
    pin a clear launch year for. Mark `(by)` upper bound rather than
    guessing. Blank is also fine — sometimes the launch is genuinely
    undocumented.

(C) `4702940` Memphis City SD is DEFUNCT post-2013 (merged into
    Memphis-Shelby at rank 16, leaid `4703810`). It still appears in
    `all_districts_ranked.tsv` because of pre-merger enrollment but
    should NOT be researched. The dedup naturally skips it because
    `4703810` is already in the CSV's leaid column at rank 16, but the
    OLD leaid `4702940` is not — so explicitly filter it out in Step 4.

(D) The CSV currently has ONE legitimate duplicate leaid: `4703810`
    appears at both rank 16 (active Memphis-Shelby) and rank 105
    (the dissolved suburban Shelby County district that merged in
    2013, kept as a historical placeholder). Don't be alarmed — but if
    a NEW duplicate leaid appears after a run, that's a bug.

(E) Leaid format is 7-digit zero-padded (e.g., `0605910`). Different
    SEDA waves sometimes drop the leading zero. The pre-computed
    `all_districts_ranked.tsv` here uses the zero-padded form
    consistently. Preserve the leading zero in CSV writes — strings, not
    integers.

(F) Quote any field containing a comma in the staging CSV. Past sessions
    had unquoted-comma bugs in Baltimore City, Volusia County, Buffalo
    Public, Clay County FL, Stafford County VA — all fixed in place.

## AGENT PROMPT TEMPLATE — paste this into each parallel Agent call

```
You are continuing a research project on US school district 1:1 device
rollouts. The CSV is at:
  ./national_chromebook_rollouts.csv

Read the strategy doc at:
  ./research_strategy.txt
for methodology.

## Districts to research (rank order)
[BATCH-SPECIFIC: paste the 50-line block from /tmp/next_batch.tsv with
 assigned rank numbers. Format each line as:
    <rank>. <leaname> -- <state> (leaid <leaid>) -- enrollment <mean_grade3>
 e.g.:
    701. ANYTOWN UNIFIED -- CA (leaid 0612345) -- enrollment 925
]

## Method
Use WebSearch as primary tool. Run 1-3 targeted queries per district like:
  - "<district>" 1:1 chromebook laptop initiative year history
  - "<district>" student device rollout begin launched
  - "<district>" chromebook distribution 2014 2015 2016 2017 2018 2019 2020
Look for the EARLIEST year a 1:1 (or near-1:1) student device program
existed. BYOD pilots count if they covered classroom learning. WebFetch is
allowed if needed for date confirmation.

DO NOT do Facebook profile-search work in this phase. Set fb_search="0".

## Columns (16 total, in this exact order)
district_name,state,city,year_started,month_started,one_to_one,grade_levels,
device_type,notes,source_url,source_2,seda_rank,seda_row_count,web_search,
fb_search,leaid

- year_started: 4-digit year, or "2020 (by)" / "2014 (by)" for upper
  bound, or blank.
- one_to_one: Yes / Yes (phased) / Yes (likely) / Partial.
- seda_rank: the assigned integer rank from the list above. NOT enrollment.
- seda_row_count: the EXACT mean-grade-3 enrollment from the list.
- web_search: "1". fb_search: "0".
- device_type goes in EXACTLY ONE field — do not duplicate.
- leaid: copy verbatim from the assignment above (7-digit zero-padded).

## Output
Write your 50 rows to a new file at:
  ./staging_<N1>-<N2>.csv

All-quoted CSV, 16 fields per row, no header. If your Write tool is
denied, return the entire CSV inline in your final summary so the parent
can write it.

## Final report (keep tight)
Counts of: pre-pandemic firm year (2009-2019), COVID-era 2020 firm,
post-2020 firm (2021+), "(by)" upper bound, blank. Note any pre-2014
firm-year adopters and any flagged ambiguities.
```

---

## What to do if something is unrecoverable

If validation in Step 6 fails for ALL SIX staging files in ways you
can't fix, restore from the backup and bail:

```powershell
Copy-Item ("national_chromebook_rollouts_${maxRank}rows.bak.csv") national_chromebook_rollouts.csv -Force
Remove-Item LOCK
```

Report the failure and exit.

If only ONE batch failed, still append the five good batches (250 rows)
and skip the failing 50 — but log it clearly so Mike can re-run that
slice manually.
