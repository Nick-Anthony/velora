# Account Prioritization Agent

[Video Link](https://drive.google.com/file/d/110EnBOkZnCaP91zTwLrsUSvXSzbNlIrB/view?usp=sharing)

Ranks CRM accounts for weekly SDR outreach. Takes an account export and an
engagement export, and produces a prioritized call list where every row carries
the reason for its position and a confidence rating.

## Running it

**[Open in Google Colab](https://colab.research.google.com/drive/1C0FHm0dgtItg7EXvkHnspiQhqEvZw1jM?usp=sharing)** → **Runtime → Run all** (`Ctrl+F9`).

Takes about 30 seconds. Tries to load files from github, if it cannot will request you to upload the two files.

The ranked list prints at the bottom of the notebook, and
`prioritized_accounts.csv` and `review_queue.csv` download automatically when the
run finishes. If your browser blocks the second download as a popup, both files
are also in the Colab file browser (folder icon in the left sidebar).

Both output files are committed to this repo as well, if you'd rather just read
the result without running anything.

## What's in this repo

| file | what it is |
|---|---|
| `Velora_Takehome.ipynb` | the MVP — annotated, runs top to bottom |
| `REQUIREMENTS.md` | one-page requirements document: approach, assumptions, validation, non-goals |
| `accounts.csv` | supplied account export (300 rows) |
| `engagement_signals.json` | supplied engagement export (360 rows) |
| `prioritized_accounts.csv` | example output — the ranked call list |
| `review_queue.csv` | example output — records needing human correction |

## Output

`prioritized_accounts.csv`, one row per account, highest priority first:

| column | meaning |
|---|---|
| `rank`, `priority` | position and score |
| `confidence` | High / Medium / Low — how much of the score rests on real data |
| `why` | plain-English reason for the position |
| `flags` | what was imputed or merged, and why |
| `fit`, `intent`, `neglect` | the three score components, so the rank is inspectable |
| `source_names`, `source_arr` | audit trail for merged duplicate records |

`review_queue.csv` holds the records a human should correct in the CRM —
low-confidence accounts, invalid values, and merged pairs with conflicting
revenue figures.

## How the ranking works

```
priority = fit × (0.7·intent + 0.3·neglect)
```

- **Fit** — would we care if they said yes. ARR percentile blended with tier weight.
- **Intent** — are they showing interest now. Each engagement event scored as
  `type_weight × log1p(count) × 0.5^(days_ago / 30)`, summed per account.
- **Neglect** — days since a rep last made contact, capped at 180.

Fit multiplies rather than adds: a large account showing no activity is a
nurture, not a call. Full reasoning, including how the weights were chosen and
how the ranking was tested, is in [`REQUIREMENTS.md`](REQUIREMENTS.md).

## Data handling

The supplied export contains a number of defects. All are handled and reported
rather than silently dropped:

- 14 organisations recorded twice under different names (`AFSC` /
  `American Friends Service Committee`) — merged on website domain
- 52 engagement records whose account name doesn't match the CSV exactly
  (`Teach For Amer.`, `AMERICAN CIVIL LIBERTIES UNION`) — resolved by
  normalisation; all 360 event rows matched
- a non-numeric value and a negative value in the ARR column
- 38 missing ARR values, 23 missing contact dates, 4 contact dates in the future
- 3 accounts with an email address in the website field

Missing values are imputed **neutral, never zero** — zero-filling would penalise
an account for a CRM data-entry gap rather than for anything real. Every
imputation is flagged on the row and lowers that account's confidence rating.
Low-confidence accounts are ranked and shown, never suppressed.

## Scope

Not built: live CRM sync, scheduling, authentication, a UI. Re-running against a
fresh export regenerates the list.

Weights are informed priors, not fitted values — the export contains no
conversion outcomes, so nothing can be trained against. Perturbing every weight
±20% across 200 runs holds Spearman rank correlation at 0.99, so the ordering is
driven by the data rather than by those choices. See `REQUIREMENTS.md` for how
this would be validated properly.
