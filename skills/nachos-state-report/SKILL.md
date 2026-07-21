---
name: nachos-state-report
description: >
  Use this skill to produce a concise NACHOS state complexity REPORT from a scored spreadsheet
  (e.g., NACHOS_[State].xlsx or a state agency file), emphasizing AVERAGE NACHOS and AVERAGE
  adjusted scores rather than totals. Triggers: any request for a NACHOS state report, a
  summary or overview of a state's NACHOS complexity, an averages-focused breakdown, an
  extension-vs-core distribution, or a drivers conclusion delivered as a PDF. Also trigger when
  a NACHOS spreadsheet is uploaded and the user asks for a report or PDF summary. The report
  covers a state baseline, a per-domain breakdown, an extension-vs-core distribution with
  customization and high-complexity counts, and a conclusion naming the main base and adjusted
  complexity drivers and their patterns. Do NOT use for the full deep-dive with an attendance deep-dive or attendance
  simulation (that is the nachos-state-analysis skill), and do NOT use for multi-state comparisons.
---

# NACHOS State Report Skill

## Purpose

A concise, report-style NACHOS profile for a single state, built to be read at a glance —
**average** complexity is the headline, totals are secondary context. Four steps, then a PDF:

1. **State Baseline** — averages headlined (avg NACHOS, avg Adjusted shown big; totals small)
2. **Per-Domain Breakdown** — same averages-first emphasis, per domain
3. **Extension vs Core Distribution** — element split for extensions vs core, plus the count
   and share of Customization (Adjusted NACHOS > 0) and High Complexity (NACHOS ≥ 2), per
   domain and for the whole state
4. **Conclusion** — the main drivers of base and adjusted complexity and the patterns behind them

Then offer to produce a **PDF** of the report.

## Input

Expects a NACHOS scored spreadsheet (.xlsx), **Details** tab. Required (match semantically if
the state's column names differ — inspect names first):

| Column | Purpose |
|--------|---------|
| `Entity Name` | Ed-Fi entity / resource |
| `Domain` | Ed-Fi domain |
| `Data Element` | Element name |
| `NACHOS score` | Base score (0–3) |
| `Adjusted NACHOS Score` | Score with extension and cross-entity penalties |
| `Is an extension` | "Yes" / "No" (convert 1/0 if needed) |
| `Business Logic (Redacted)` | Plain-language description (for pattern inference) |
| `Business Logic (Formula)` | Formula representation, if present |
| `Cross Entity` | "Yes" / "No" (derive from a cross-entity penalty column if needed) |

## Step 0 — Load, Validate, Detect State, Capture Report Header

**Require a file first.** If no NACHOS spreadsheet is attached, ask the user to attach one
before doing anything.

```python
import pandas as pd, re, os, datetime

df = pd.read_excel("<uploaded_file>", sheet_name="Details", header=0)
df_valid = df.dropna(subset=['NACHOS score', 'Adjusted NACHOS Score'])
print(df_valid.columns.tolist(), len(df_valid))
```

Drop rows where both score columns are null; keep 0-scored rows.

**State name resolution** (in order): read a `State`/`State Name` column if present; else infer
from the attached filename — a state name/abbreviation anywhere in it, or an agency code mapped
to its state (`GADOE_` → Georgia, `TEA_` → Texas, `CDE_` → California, `NYSED_` → New York); if
ambiguous, ask the user.

**Report header — capture three items and show them at the top of the report and on the PDF cover:**

1. **Ed-Fi Data Standard version.** Ask the user directly:
   > "Which Ed-Fi Data Standard version is [state] using? (e.g., DS 6.1, DS 7.0)"
   If a version is inferable from the filename, offer it as a default to confirm. **If the user
   doesn't know, record it as "Not known"** and display that verbatim — do not guess.
2. **NACHOS file version** — derived from the file, not asked:
   ```python
   fname = os.path.basename("<uploaded_file>")
   m = (re.search(r'[Vv]\d{4}[_-]\d{2}[_-]\d{2}', fname) or   # e.g. V2025_06_20
        re.search(r'\d{4}[_-]\d{2}[_-]\d{2}', fname) or       # a bare date in the name
        re.search(r'[Vv]\d+(?:\.\d+)?', fname))               # e.g. V3 / v2.1
   nachos_file_version = m.group(0) if m else \
       datetime.date.fromtimestamp(os.path.getmtime("<uploaded_file>")).isoformat()  # fallback: file date
   ```
3. **Report creation date** — the date the report is generated:
   ```python
   report_created = datetime.date.today().strftime('%B %d, %Y')
   ```

Use the resolved state name in all headings.

## Step 1 — State Overall Baseline *(averages headlined)*

```python
total_n        = len(df_valid)
total_entities = df_valid['Entity Name'].nunique()
total_domains  = df_valid['Domain'].nunique()
total_nachos = df_valid['NACHOS score'].sum()
total_adj    = df_valid['Adjusted NACHOS Score'].sum()
avg_nachos   = total_nachos / total_n      # PRIMARY metric
avg_adj      = total_adj / total_n         # PRIMARY metric
uplift       = total_adj / total_nachos
n05 = (df_valid['NACHOS score'] >= 0.5).sum()
a05 = (df_valid['Adjusted NACHOS Score'] >= 0.5).sum()
n2  = (df_valid['NACHOS score'] >= 2).sum()
```

**Baseline table** — label the single data row with the **actual state name** (not "Per element"),
and give it these columns:

| [State name] | Total Elements Reviewed | Total Entities | Total Domains | **Avg NACHOS** | **Avg Adjusted NACHOS** |
|--------------|------------------------|----------------|---------------|----------------|-------------------------|

e.g. the row reads `Georgia | 223 | 21 | 6 | 0.23 | 0.50`. Keep the two **averages emphasized**
as the headline figures (largest text / leading emphasis). Show total NACHOS, total Adjusted,
the uplift factor, and the ≥0.5 / ≥2 counts as smaller secondary context beneath the table —
the averages are the story; the totals support it.

## Step 2 — Per-Domain Breakdown *(averages headlined)*

```python
rows = []
for dom, g in df_valid.groupby('Domain', sort=False):
    n = len(g)
    rows.append({
        'domain':     dom,
        'avg_nachos': g['NACHOS score'].sum() / n,          # PRIMARY
        'avg_adj':    g['Adjusted NACHOS Score'].sum() / n,  # PRIMARY
        'n':          n,
        'pct_ext':    (g['Is an extension'].str.lower() == 'yes').sum() / n * 100,
        'nachos':     g['NACHOS score'].sum(),
        'adj':        g['Adjusted NACHOS Score'].sum(),
        'pct_nachos': g['NACHOS score'].sum() / total_nachos * 100,
        'pct_adj':    g['Adjusted NACHOS Score'].sum() / total_adj * 100,
    })
rows.sort(key=lambda x: x['avg_nachos'], reverse=True)   # rank by average, not total
```

**Table, averages first:** Domain · **Avg NACHOS** · **Avg Adjusted** · Elements · % Ext ·
Total NACHOS · Total Adj. · % NACHOS Complexity · % Adjusted NACHOS Complexity (the last two
are each domain's share of the state's total NACHOS and total adjusted NACHOS). Lead with (and
visually emphasize) the two average columns; sort by Avg NACHOS descending. Include a
whole-state row.

## Step 3 — Extension vs Core Distribution, with Customization & High Complexity

For each domain **and** for the whole state, show the element split between extensions and core,
plus two complexity-incidence measures: how many elements require any customization at all, and
how many are highly complex. Do **not** report average NACHOS or average adjusted scores per
core vs extension here — this step is about counts and shares, not averages.

**Definitions (include this legend with the table):**
- **Customization** — elements with an **Adjusted NACHOS score > 0**: anything that needs more
  than a direct field map (a transformation, conditional, extension, or cross-entity handling).
- **High Complexity** — elements with a **NACHOS score ≥ 2** (base score): aggregations, multi-branch
  logic, or other elements whose base business logic is inherently demanding.

```python
def dist_profile(g):
    n = len(g)
    e = g[g['Is an extension'].str.lower() == 'yes']
    c = g[g['Is an extension'].str.lower() == 'no']
    cust = (g['Adjusted NACHOS Score'] > 0).sum()     # Customization
    high = (g['NACHOS score'] >= 2).sum()             # High Complexity (base NACHOS)
    return {
        'n': n,
        'n_core': len(c), 'pct_core': len(c) / n * 100,
        'n_ext':  len(e), 'pct_ext':  len(e) / n * 100,
        'n_cust': int(cust), 'pct_cust': cust / n * 100,   # Adjusted > 0
        'n_high': int(high), 'pct_high': high / n * 100,   # NACHOS >= 2
    }

dist = {dom: dist_profile(g) for dom, g in df_valid.groupby('Domain', sort=False)}
dist['ALL — STATE'] = dist_profile(df_valid)
```

**Table:** Domain · Core n (%) · Ext n (%) · **Customization n (%)** · **High Complexity n (%)**.
Put the whole-state row last and highlight it, and place the legend directly beneath. Note in
prose whether extensions are concentrated in particular domains, and how the customization and
high-complexity shares compare across domains (a domain can be heavily customized yet have few
high-complexity elements, or vice versa).

## Step 4 — Conclusion: Main Drivers & Patterns

Name the domains/entities that drive complexity and the pattern behind them. Report base and
adjusted separately, because extension and cross-entity penalties can shift which areas dominate.

```python
# top contributors by share of base and of adjusted complexity
base_by_dom = sorted(rows, key=lambda x: x['nachos'], reverse=True)
adj_by_dom  = sorted(rows, key=lambda x: x['adj'],    reverse=True)

# entity-level drivers among high-complexity elements (>= 2)
high = df_valid[df_valid['NACHOS score'] >= 2].copy()
```

### Pattern Inference (apply in order, stop at first match)

| Pattern | Detection (Business Logic Formula/Redacted, or calculation flag) |
|---------|------------------------------------------------------------------|
| COUNT aggregate | contains `COUNT(` |
| SUM aggregate | contains `SUM(` |
| Conditional threshold / cap | contains `>`, `<`, `exceed`, `cap`, `minimum`, `maximum`, `limit` |
| Unit conversion | contains `convert`, `conversion`, `minutes to`, `hours to`, `days to` |
| Multi-branch conditional | word `if` appears 3+ times |
| Cross-entity lookup | Cross Entity = "Yes" and no aggregate detected |
| Other / unclassified | none of the above |

If no formula column exists, infer from the plain-language logic + calculation flag and say so —
SUM/COUNT tagging is then approximate.

### Write the conclusion (prose, 1–2 short paragraphs) covering:

- **Base NACHOS drivers** — the top 1–3 domains/entities by base contribution and how much of the
  state's base NACHOS they account for.
- **Adjusted NACHOS drivers** — the top 1–3 by adjusted contribution; call out any area that
  climbs the ranking under adjustment (i.e., extension/cross-entity-driven rather than logic-driven).
- **Dominant pattern(s)** — the pattern(s) the driver elements share (e.g., record aggregation in
  one entity, cross-entity descriptor classification in another) and why they reach NACHOS 3.
- **One-line takeaway** — is this state's complexity primarily logic-driven or extension-driven?

## Produce the PDF *(ask first)*

Close by asking:

> "Would you like a PDF of this report?"

If yes, use the **pdf skill**. Section order:

1. **Cover page** — state name; **Ed-Fi Data Standard version** (or "Not known"); **NACHOS file
   version**; **report creation date**
2. State Baseline (averages headlined)
3. Per-Domain Breakdown (averages headlined)
4. Extension vs Core Distribution
5. Conclusion — main drivers & patterns

Note in a short methodology line that the report is AI-assisted, that pattern tags are inferred
(especially where no formula column exists), and that figures should be reviewed by a human
before informing decisions.

**PDF layout notes:**
- *State Baseline* — render the two headline figures (Average NACHOS, Average Adjusted NACHOS)
  as two separate cards with a clear gap between them (e.g., a spacer column or generous
  cell padding), not flush against each other. Inside each card, show the matching **total**
  beneath the average (Total NACHOS under Average NACHOS, Total Adjusted under Average Adjusted)
  so the totals are visible alongside the averages rather than crowded out.
- *Per-Domain Breakdown* — allow column headers to **wrap onto two lines** so long labels display
  in full; in particular force a line break in `% NACHOS Complexity` and
  `% Adjusted NACHOS Complexity` (e.g., a `<br/>` between the phrase and "Complexity") and give
  those columns enough width/row height that the full text shows.

## Rendering

Use `visualize:show_widget` for the metric cards and tables when available; otherwise fall back
to markdown. Emphasis rule for this skill: **average NACHOS and average adjusted are always the
largest / leading figures; totals are secondary.** Color convention:

| Element | Color |
|---------|-------|
| Table headers | Navy #1B2A4A |
| Average-score emphasis | Teal #0F6E56 |
| Driver / high-complexity callouts | Coral #993C1D |
| Extension highlights | Amber #BA7517 |
| Whole-state / totals row | Light blue #E6F1FB background |
| Alternating rows | Light gray #F1EFE8 |

## NACHOS Scoring Reference

| Score | Meaning | Typical pattern |
|-------|---------|----------------|
| 0 | Direct field map | Send value or descriptor as-is |
| 1 | Single conditional | One if/then |
| 2 | Simple calculation + up to 3 conditionals | A+B, A/B, up to 3 branches |
| 3 | Aggregate or transformation | SUM/COUNT across rows, unit conversion, 3+ branches |

**Adjusted NACHOS** = Base + 0.5 (necessary ext) or +1.0 (unnecessary ext) + 0.5 (cross-entity).
Penalties are additive. Theoretical max: 3 + 1.0 (unnecessary ext) + 0.5 (cross-entity) = 4.5.
