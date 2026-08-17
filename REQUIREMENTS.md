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
