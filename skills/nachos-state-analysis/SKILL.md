---
name: nachos-state-analysis
description: >
  Use this skill to run a full NACHOS complexity analysis for a single state using a NACHOS
  scored spreadsheet (e.g., NACHOS_[State]_V[n].xlsx). Triggers: any request to analyze NACHOS
  scores for a state, understand what drives implementation complexity, get a domain breakdown,
  identify high-complexity elements across all domains, or produce a NACHOS report. Also trigger
  when the user uploads a NACHOS spreadsheet and asks anything about scores, domains, complexity,
  extensions, or drivers. Do NOT trigger for multi-state comparisons or Ed-Fi standard change
  impact analysis — those are handled by other skills.
---

# NACHOS State Analysis Skill

## Purpose

Full NACHOS complexity analysis for a single state from a scored spreadsheet. Six steps:

1. **State Baseline** — element count, NACHOS/Adjusted totals and averages, extension burden, threshold distribution
2. **Domain Breakdown** — full per-domain profile table sorted by NACHOS contribution
3. **Attendance Deep-Dive** — entity-level profile, outlier detection, contribution summary
4. **High-Complexity Drivers** — pattern classification (SUM, COUNT, threshold, conversion, multi-branch) for ALL domains wherever NACHOS >= 2
5. **Granular Attendance Simulation** *(optional — ask first)* — removes only the extension elements in the Attendance scope (keeps core), with an all-attendance-extensions delta plus a per-entity breakdown
6. **PDF Offer** — offer to generate a formatted PDF report after inline analysis

## Input

Expects a NACHOS scored spreadsheet (.xlsx), **Details** tab. Required columns:

| Column | Purpose |
|--------|---------|
| `Entity Name` | Ed-Fi entity |
| `Domain` | Ed-Fi domain |
| `Data Element` | Element name |
| `NACHOS score` | Base score (0–3) |
| `Adjusted NACHOS Score` | Score with extension and cross-entity penalties |
| `Is an extension` | "Yes" / "No" |
| `Business Logic (Redacted)` | Plain-language description |
| `Business Logic (Formula)` | Formula-style representation |
| `Cross Entity` | "Yes" / "No" |

Always inspect column names at Step 0 and match semantically if names differ across states.

## Step 0 — Load, Validate, Detect State Name

```python
import pandas as pd

df = pd.read_excel("<uploaded_file>", sheet_name="Details", header=0)
df_valid = df.dropna(subset=['NACHOS score', 'Adjusted NACHOS Score'])
print(f"Columns: {df_valid.columns.tolist()}")
print(f"Valid rows: {len(df_valid)}")
```

Drop rows where both score columns are null. Do NOT drop rows where scores are 0.

**Require a file first.** This skill analyzes an attached NACHOS spreadsheet. If no file
has been attached, don't guess or run anything — ask the user to attach the NACHOS scored
spreadsheet (.xlsx) before continuing.

**State name resolution** (in order) — infer the state from whatever file the user
attached, since this skill runs for any state and filenames vary widely:
1. Look for a `State` or `State Name` column in the data; read the first non-null value
2. Parse it from the attached file's name. Filenames don't follow one fixed convention, so
   match flexibly against the actual attached filename:
   - a state name or abbreviation embedded anywhere in the name
     (`NACHOS_Ohio_V2.xlsx` → Ohio, `nachos-wa-final.xlsx` → Washington)
   - a state education agency code, mapped to its state
     (`GADOE_...` → Georgia, `TEA_...` → Texas, `CDE_...` → California, `NYSED_...` → New York)
3. If the filename is ambiguous or you can't resolve it confidently, ask the user which
   state it is rather than assuming

Use the resolved state name in all headings throughout the analysis.

## Step 1 — State Overall Baseline

```python
total_n       = len(df_valid)
total_nachos  = df_valid['NACHOS score'].sum()
total_adj     = df_valid['Adjusted NACHOS Score'].sum()
avg_nachos    = total_nachos / total_n
avg_adj       = total_adj / total_n
uplift_factor = total_adj / total_nachos

nachos_gte_05 = (df_valid['NACHOS score'] >= 0.5).sum()
adj_gte_05    = (df_valid['Adjusted NACHOS Score'] >= 0.5).sum()
nachos_gte_2  = (df_valid['NACHOS score'] >= 2).sum()
```

**Report as metric cards:**
- Total elements
- Total NACHOS + average per element
- Total Adjusted NACHOS + average per element
- Adjusted uplift factor — values above 2.0 signal heavy extension burden
- Count and % with NACHOS >= 0.5
- Count and % with Adjusted NACHOS >= 0.5
- Count and % with NACHOS >= 2

> The gap between NACHOS >= 0.5 (%) and Adjusted NACHOS >= 0.5 (%) shows how much
> complexity comes from extensions vs. base business logic.

## Step 2 — Per-Domain Breakdown

```python
domain_rows = []
for domain, g in df_valid.groupby('Domain', sort=False):
    n  = len(g)
    domain_rows.append({
        'domain':      domain,
        'n_entities':  g['Entity Name'].nunique(),
        'n':           n,
        'pct_ext':     (g['Is an extension'].str.lower() == 'yes').sum() / n * 100,
        'pct_high':    (g['NACHOS score'] >= 2).sum() / n * 100,
        'nachos':      g['NACHOS score'].sum(),
        'avg_nachos':  g['NACHOS score'].sum() / n,
        'adj':         g['Adjusted NACHOS Score'].sum(),
        'avg_adj':     g['Adjusted NACHOS Score'].sum() / n,
        'pct_nachos':  g['NACHOS score'].sum() / total_nachos * 100,
        'pct_adj':     g['Adjusted NACHOS Score'].sum() / total_adj * 100,
    })
domain_rows.sort(key=lambda x: x['nachos'], reverse=True)
```

**Table columns:** Domain · Entities · Elements · % Ext · % NACHOS>=2 · Total NACHOS · Avg NACHOS · Total Adj. · Avg Adj. · % of State NACHOS · % of State Adj.

Sort by Total NACHOS descending. Include totals row.

**Disproportionality flag**: For any domain where `% of State NACHOS` / `% of Elements` >= 1.5, add a callout noting it carries disproportionate complexity.

## Step 3 — Attendance Deep-Dive

```python
mask_domain = df_valid['Domain'].str.lower() == 'attendance'
mask_name   = df_valid['Entity Name'].str.contains('attendance', case=False, na=False)
att = df_valid[mask_domain | mask_name]
```

Note any entities caught by only one criterion — flag as potential misclassifications.

### Entity Profile Table

Same columns as the domain table but at entity level. For each entity compute: n, pct_ext, pct_high, nachos, avg_nachos, adj, avg_adj, pct_state_nachos, pct_state_adj. Include totals row.

### Low-Complexity Outlier Detection

Flag entities where `avg_nachos < 0.1` AND `n_high == 0`. Show them in the table with a visual flag but exclude them from Step 4 driver analysis. Never hardcode entity names — always derive dynamically. (Step 5 selects by extension flag rather than by entity, so it isn't affected by this exclusion.)

### Attendance Contribution Summary

- Attendance elements as % of state total
- Attendance NACHOS as % of state total NACHOS
- Disproportionality ratio (NACHOS% / elements%)
- Compression effect: if adjusted NACHOS% is meaningfully lower than base NACHOS%, note that extension burden is heavier outside attendance than within it

## Step 4 — High-Complexity Driver Analysis (All Domains)

Covers the **entire state**. Excludes zero-complexity outlier entities (derived dynamically).

```python
all_stats = df_valid.groupby('Entity Name').agg(
    avg_nachos=('NACHOS score', 'mean'),
    n_high=('NACHOS score', lambda x: (x >= 2).sum())
).reset_index()
outlier_entities = all_stats[
    (all_stats['avg_nachos'] < 0.1) & (all_stats['n_high'] == 0)
]['Entity Name'].tolist()

high = df_valid[
    (df_valid['NACHOS score'] >= 2) &
    (~df_valid['Entity Name'].isin(outlier_entities))
]
```

### Pattern Inference Rules (apply in order, stop at first match)

| Pattern | Detection |
|---------|-----------|
| COUNT aggregate | Formula contains `COUNT(` (case-insensitive) |
| SUM aggregate | Formula contains `SUM(` |
| Conditional threshold / cap | Formula or logic contains `>`, `<`, `exceed`, `cap`, `minimum`, `maximum`, `limit` |
| Unit conversion | Formula or logic contains `convert`, `conversion`, `minutes to`, `hours to`, `days to` |
| Multi-branch conditional | Word `if` appears 3+ times in logic text |
| Cross-entity lookup | Cross Entity = "Yes" AND no aggregate detected |
| Other / unclassified | None of the above |

### Output Structure Per Domain

For each domain with NACHOS >= 2 elements (sorted by NACHOS contribution, highest first):

1. **Domain header** — total high-complexity element count and NACHOS total
2. **Pattern summary table** — pattern, count, % of domain's high-complexity elements, cross-entity co-occurrence rate
3. **Raw evidence** — up to 3 examples per pattern showing: entity name, data element, Business Logic (Formula) verbatim, Business Logic (Redacted) verbatim, NACHOS score, Adjusted NACHOS score
4. **Pattern narrative** — 2–3 sentences explaining why the dominant pattern(s) reach NACHOS = 3 under the scoring rubric

## Step 5 — Granular Attendance Simulation *(optional — ask first)*

This step is opt-in. After presenting Steps 1–4, **always ask the user first** — do not
run it unless they say yes:

> "Would you like an attendance simulation? It removes only the *extension* (non-core)
> elements in the Attendance scope — keeping the core elements — and shows how the
> state's scores change: Part A across all attendance extensions, Part B broken down
> per entity."

If the user says no, skip this step entirely: go straight to Step 6 and omit the
simulation section from any PDF. Only proceed with the rest of Step 5 on a clear yes.

**Removal set.** The simulation isolates the burden the state *adds on top of* the core
standard within attendance, so it removes only the extension elements — not the core
ones. Scope attendance the same way as Step 3 (Domain is attendance OR entity name
contains "attendance"), then keep only rows flagged as extensions. Derive everything
dynamically; never hardcode.

```python
att_mask = (df_valid['Domain'].astype(str).str.lower() == 'attendance') | \
           (df_valid['Entity Name'].str.contains('attendance', case=False, na=False))
att_ext  = df_valid[att_mask & (df_valid['Is an extension'].astype(str).str.lower() == 'yes')]
```

If `att_ext` is empty, there are no attendance extensions to remove — report that as a
null result (state scores are unchanged) rather than printing empty tables, and move on.

### Part A — Remove All Attendance Extensions

Drop every attendance extension element at once; core attendance elements stay in.

```python
remaining      = df_valid.drop(index=att_ext.index)
rem_n          = len(remaining)
rem_nachos     = remaining['NACHOS score'].sum()
rem_adj        = remaining['Adjusted NACHOS Score'].sum()
rem_avg_nachos = rem_nachos / rem_n
rem_avg_adj    = rem_adj / rem_n
```

**Delta table:**

| Metric | With Att. Extensions | Without Att. Extensions | Delta | % Change |
|--------|----------------------|-------------------------|-------|----------|
| Total elements | | | | |
| Total NACHOS | | | | |
| Avg NACHOS / element | | | | |
| Total Adjusted NACHOS | | | | |
| Avg Adj. NACHOS / element | | | | |
| Elements NACHOS >= 0.5 | (%) | (%) | | Δ pct pts |
| Elements Adj. >= 0.5 | (%) | (%) | | Δ pct pts |

List the removed elements (entity + data element) explicitly so the user understands the
simulation scope, and note how many core attendance elements were retained.

### Part B — Per-Entity Breakdown

Break the removal down by entity — for each entity that has attendance extensions, remove
just that entity's extension elements. This is what makes the simulation *granular*: it
shows which entities actually carry the extension burden rather than treating attendance
as a monolith.

```python
per_entity_rows = []
for entity, g in att_ext.groupby('Entity Name'):
    d_nachos = g['NACHOS score'].sum()                # removing these rows drops exactly this
    d_adj    = g['Adjusted NACHOS Score'].sum()
    kept_n   = total_n - len(g)
    per_entity_rows.append({
        'entity':                   entity,
        'n_ext':                    int(len(g)),
        'd_nachos':                 d_nachos,
        'd_adj':                    d_adj,
        'new_avg_nachos':           (total_nachos - d_nachos) / kept_n,
        'new_avg_adj':              (total_adj    - d_adj)    / kept_n,
        'pct_state_nachos_removed': d_nachos / total_nachos * 100,
        'pct_state_adj_removed':    d_adj    / total_adj    * 100,
    })
per_entity_rows.sort(key=lambda x: x['d_nachos'], reverse=True)
```

**Per-entity table** (sorted by NACHOS removed, highest first; include a totals row):

| Entity | Ext. Elements | NACHOS Removed | Adj. Removed | New State Avg NACHOS | New State Avg Adj. | % of State NACHOS Removed | % of State Adj. Removed |
|--------|---------------|----------------|--------------|----------------------|--------------------|---------------------------|-------------------------|

The attendance extension elements are disjoint across entities, so the per-entity
`NACHOS Removed` values sum exactly to the Part A total — call this out as a sanity check
that the two parts agree.

**Interpretation:**
- Adjusted delta >> base delta → the attendance extension burden is mostly extension/cross-entity penalties layered on low base scores
- Sizable base delta too → the extensions add real calculation logic, not just an extension flag
- Concentrated in one entity → the extension burden is localized (a clean scoping target);
  spread across entities → it is systemic
- Compare against the retained core attendance elements: if core scores stay near zero, the
  attendance complexity that exists is entirely state-added

## Step 6 — PDF Offer

After the analysis (and the granular attendance simulation, if it was run), always close with:

> "Would you like a formatted PDF report with all of these results — the state baseline,
> domain breakdown, attendance deep-dive, and driver analysis?"

If the granular attendance simulation was run, add "and the granular attendance simulation"
to that offer.

If confirmed, use the **pdf skill**. PDF section order:
1. Cover page (state name, file version, date)
2. State Overall Baseline
3. Per-Domain Breakdown
4. Attendance Deep-Dive
5. High-Complexity Drivers by domain
6. Granular Attendance Simulation + key findings — include only if the simulation was run

## Rendering

Use `visualize:show_widget` for all tables when available. Color convention:

| Element | Color |
|---------|-------|
| Table headers | Navy #1B2A4A |
| Attendance highlights | Teal #0F6E56 |
| High-complexity callouts | Coral #993C1D |
| Disproportionality flags | Amber #BA7517 |
| Totals rows | Light blue #E6F1FB background |
| Alternating rows | Light gray #F1EFE8 |

Fall back to markdown tables only if the widget tool is unavailable.

---

## NACHOS Scoring Reference

| Score | Meaning | Typical pattern |
|-------|---------|----------------|
| 0 | Direct field map | Send value or descriptor as-is |
| 1 | Single conditional | One if/then |
| 2 | Simple calculation + up to 3 conditionals | A+B, A/B, up to 3 branches |
| 3 | Aggregate or transformation | SUM/COUNT across rows, unit conversion, 3+ branches |

**Adjusted NACHOS** = Base + 0.5 (necessary ext) or +1.0 (unnecessary ext) + 0.5 (cross-entity).
Penalties are additive. Theoretical max: 3 + 1.0 (unnecessary ext) + 0.5 (cross-entity) = 4.5.
