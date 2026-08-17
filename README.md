# Account Prioritization Agent

[Video Link](https://drive.google.com/file/d/110EnBOkZnCaP91zTwLrsUSvXSzbNlIrB/view?usp=sharing)

Ranks CRM accounts for weekly SDR outreach. Takes an account export and an
engagement export, and produces a prioritized call list where every row carries
the reason for its position and a confidence rating.

## Running it

**[Open in Google Colab](https://colab.research.google.com/drive/1C0FHm0dgtItg7EXvkHnspiQhqEvZw1jM?usp=sharing)** → **Runtime → Run all** (`Ctrl+F9`).

Takes about 30 seconds. Tries to load files from github, if it cannot wil lrequest you to upload the two files.

The ranked list prints at the bottom of the notebook, and
`prioritized_accounts.csv` and `review_queue.csv` download automatically when the
run finishes. If your browser blocks the second download as a popup, both files
are also in the Colab file browser (folder icon in the left sidebar).

Both output files are committed to this repo as well, if you'd rather just read
the result without running anything.

## What's in this repo

| file | what it is |
|---|---|
| `account_prioritizer.ipynb` | the MVP — annotated, runs top to bottom |
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

# Requirements Doc

## Interpretation

The VP asked for a ranked call list. The underlying problem is that reps choose
targets ad hoc, so high-value accounts showing active interest get the same
attention as accounts that will never convert. "Priority" is therefore defined as:
**an account worth winning that is showing interest now and is not already being
worked.**

## Scope and assumptions

In scope: ingest both exports, deduplicate, score, rank, and emit a prioritized
CSV plus a review queue, with a per-row explanation and confidence rating.

Assumptions:

- **Missing or invalid ARR is imputed to the median** (40 accounts). This covers
  38 blanks, one non-numeric string, and one negative value. Zero-filling would
  penalise an account for a CRM data-entry gap rather than for anything real, so
  an unknown value lands mid-pack. Every imputed row is flagged and its
  confidence rating drops.

- **ARR of exactly 0.00 is left as-is** (18 accounts). A free-tier or lapsed
  account is plausible, and the export gives no way to distinguish that from a
  null written as zero. The ambiguity is noted rather than resolved.

- **Missing or placeholder tier gets a weight of 0.4** (8 accounts: 5 blank, 3
  `TBD`). Same reasoning as ARR — unlabelled means unknown-sized, not small. The
  three `TBD` accounts sit at a median ARR percentile of 0.44, which supports 0.4
  as a neutral value.

- **An unknown last-contact date sets neglect to 0.5** (27 accounts: 23 blank, 4
  future-dated). Treating a blank as "never contacted" would give it maximum
  neglect and actively promote it, so a missing field would raise an account's
  rank.

- **Where merged duplicates disagree on ARR, the higher value is taken** (14
  pairs). The export has no modification timestamp, so there is no way to
  identify which figure is current. Both values are retained in the output for
  verification.

- **Engagement recency decays against the latest event date in the export**
  (2026-07-28), not the wall clock, so the same export produces the same ranking
  on any run date.

- **`industry` and `region` are excluded from scoring.** With no conversion
  history, any weighting would encode assumption rather than evidence. Both are
  retained as filter columns.

## Approach

**Priority = fit × (0.7·intent + 0.3·neglect)**

Fit multiplies rather than adds, because both conditions must hold. A $120k
account with no engagement scores exactly zero on intent; under a weighted
average it would still rank near the top on revenue alone, which is the failure
the ranking exists to prevent. That account is a nurture, not a call.

- **Fit** (static — would we care if they said yes): 0.6 × ARR percentile +
  0.4 × tier weight. Tier weights are set to each tier's median ARR percentile
  (Enterprise 0.89, Strategic 0.85, Mid-Market 0.58, SMB 0.22), so tier acts as
  a revenue proxy for the 40 accounts where ARR is unusable.
- **Intent** (behavioural — are they showing interest now): each event scored as
  `type_weight × log1p(count) × 0.5^(days_ago/30)`, summed per account and
  normalised. Log damping stops one inflated count dominating; exponential decay
  makes last week matter more than March. Email opens are weighted lowest —
  automated inbox scanning makes open counts unreliable as a human signal,
  visible here as a median of 22.5 opens per record against 2 for demo requests.
- **Neglect** (are we already working it): days since last rep contact, capped
  at 180. This is what surfaces the VP's actual complaint — valuable accounts
  showing interest that nobody has called.

Intent and neglect are kept separate rather than combined into a single recency
measure. An account that stopped engaging is a risk; an account nobody has
called is an opportunity. Collapsing them loses that distinction.

**Deduplication is on website domain, not name.** 14 organisations appear twice
under an initialism and a full name (`AFSC` / `American Friends Service
Committee`); no string-similarity method connects those. Merged pairs carry
conflicting ARR and the export has no modification timestamp, so the merge takes
the maximum, retains both source values, and flags the record.

**Engagement joins to accounts on a normalised name key** — case folded, accents
stripped, legal suffixes and common abbreviations resolved. 52 of 241 event
names did not match exactly (`Teach For Amer.`, `AMERICAN CIVIL LIBERTIES
UNION`). Fuzzy matching was considered but proved unnecessary: all 360 event
rows matched deterministically, with no key collisions across the 300 source
accounts. A deterministic match is preferable where it works — no threshold to
defend, and the result is reproducible.

**Every imputation is recorded as a flag with its reason**, and the flags drive a
confidence rating (High 178 / Medium 101 / Low 7). Low-confidence records are
ranked and shown, never suppressed — suppressing them would recreate the problem
the tool exists to fix. Records needing human correction, such as the two invalid
ARR values and four future-dated contacts, go to a separate review queue
alongside the ranked list.

## Judging whether it works

The export has no outcome data, so ranking accuracy cannot be measured and no
accuracy figure is reported. Three things can be checked without it.

**Stability.** All weights, the five event-type weights, the fit blend, the
intent/neglect blend, and the decay half-life were perturbed ±20% across 200
runs. Spearman correlation against the baseline ranking held at 0.990 (worst run
0.967), and 83% of the top 20 was retained on average. The ordering is driven by
the data rather than by the chosen weights. The churn that does occur sits at the
boundary of the top 20, where scores are separated by hundredths of a point.

**Face validity.** Every row states its reason, so a rep can disagree specifically.
A disagreement points at a missing factor rather than at a bug.

**Coverage.** 178 of 286 accounts scored High confidence, 101 Medium, 7 Low, with
no Low-confidence account in the top 20, the head of the list is not built on
imputed values.

To really test it we would have to test it live like an A/B test. Half the team works the ranked list, half works as
today, compared on meetings booked per 100 calls over three to four weeks. That
also produces the first outcome labels, which is what would let the hand-set
weights be replaced with fitted ones.

## Non-goals and next steps

Not built: live CRM sync, scheduling, authentication, a UI, entity resolution
beyond domain and name normalisation. Learned weights are deliberately deferred —
supervised scoring needs conversion outcomes, which this export does not contain.
The scoring config is isolated so weights can be replaced with fitted coefficients
once that data exists.
