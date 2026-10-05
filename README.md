# Bayesian Curl Analysis

An analysis of ~13 months of personal bayesian curl training data, looking at how I progress and what factors (if any) actually drive that progress.

## Background

This project analyzes approximately 13 months of my bayesian curl progress at the gym.

### What does this project aim to do?

**Track my progress (rate of becoming stronger)**
- How many reps do I do per set and per session before I easily progress to the next weight setting?
- How does the difference in weight setting jumps (2kg or 5kg; see below) influence this?

**Discover trends that point to accelerated progress**
- Do rest-pause reps or drop sets have an influence?
- Did taking one month off from bayesian curls (doing basic curls instead, to let my rhomboid heal) have an effect?
- What are the effects of factors such as bodyweight, time on a weight, and previous performance?

### What is a bayesian curl?

A bayesian curl is a bicep curl exercise performed on a cable machine. The lifter stands and faces away from the machine. At the starting position, the shoulders are extended behind the body, causing the biceps to be in a more stretched position. The lifter then flexes the arm at the elbow, bringing the wrists toward the shoulders. The elbow is then extended and the arms return to the starting position.

### About the gym setup

All recorded bayesian curls were performed on a cable machine with two independent cable stacks (one per arm). Plates increase by 2kg up to the 27kg setting; after that, they increase by 5kg, so the setting following 27kg is 32kg. A standard handle attachment was used for all curls.


### Limitations

- The data records whether bayesian curls were the first or second bicep exercise of the day, but not where within the full routine they fell.
- Time of day isn't recorded. Most sessions were early morning, but not all.
- Rest time between sessions isn't tracked; it's generally consistent but not logged.
- Bodyweight wasn't tracked during this period. I weighed about 70kg at the start of data collection and roughly 76kg by the end; an estimated bodyweight column approximates this with a conservative 0.5kg/month gain.

## Repository structure

```
bayesian_curl_analysis/
├── README.md
├── bayesian_curl_analysis.ipynb
├── bayesian_curls.csv
```

## Variable Dictionary (raw data)

| Variable | Description |
|---|---|
| `date` | Date the session occurred, in `YYYY-MM-DD` format. |
| `set` | Set number within that session (resets to 1 for each new date). |
| `weight` | Weight (kg) used for the main portion of the set. Each arm's stack is independent; each arm moved this weight, not a combined total. |
| `reps` | Reps completed at `weight`, before any rest-pause or drop. |
| `rest_pause_reps` | Additional reps performed after a brief (~15 sec) rest-pause, at the same `weight`. `0` if no rest-pause was used. |
| `drop_weight` | Weight (kg) dropped mid-set for a drop set. `0` if no drop occurred. |
| `drop_reps` | Reps completed at `drop_weight`. `0` if no drop occurred. |
| `arm` | Which arm(s) performed the set: `both`, `left`, or `right`. A missing arm for a given set/date means that arm wasn't trained that set (not a 0-rep attempt). |
| `exercise_order` | Whether bayesian curl was the `first` or `second` bicep exercise performed that day. |
| `note` | Free-text context (e.g. equipment issues, injuries, supersets, extra rest). Blank when nothing notable. |

**Notes:**
- Total reps for a set = `reps + rest_pause_reps` (main portion) plus `drop_reps` at `drop_weight` if a drop occurred.
- A small number of sets included a second drop stage beyond what `drop_weight`/`drop_reps` can capture; those are recorded only in `note` (flagged: 2025-12-20 set 3, 2026-07-11 set 3).

## Model Variable Dictionary (engineered)

| Variable | Description |
|---|---|
| `total_reps` | Set 1 reps + rest_pause_reps for that session (the model's outcome variable). |
| `exercise_order_num` | Numeric encoding of `exercise_order`: 0 = first, 1 = second. |
| `rest_pause_flag` | 1 if any set that session used a rest-pause, 0 otherwise. |
| `drop_set_flag` | 1 if any set that session used a drop set, 0 otherwise. |
| `days_since_last` | Days elapsed since the previous bayesian curl session. |
| `weight_jump` | Change in weight vs. the previous session. Positive = increase, negative = deload. |
| `deload_flag` | 1 if `weight_jump` is negative, 0 otherwise. |
| `bodyweight` | Estimated bodyweight (kg) for that date, stepping up 0.5kg per calendar month from 70kg to a 76kg cap. |
| `days_since_break` | Days since returning from the one-month basic-curls break (2026-06-17), clipped at 0 for any date before/at the break. |
| `total_reps_lag1` | `total_reps` from the previous session (captures session-to-session momentum). |
| `change_in_weight` | 1 if this session's weight differs from the immediately preceding session's weight, 0 otherwise. Flags any weight change, including a one-off light day or a reverted attempt, not just a true first-ever arrival at a weight. |
| `block_id` | Sequential ID of a sustained weight block (0 = 11kg, 1 = 16kg, ... 7 = 32kg). |
| `is_block_primary` | 1 if this session's weight matches its block's assigned weight, 0 if it's a deviation (light day, tired-day dip, or a reverted attempt at the next tier). |
| `session_num_in_block` | Count of primary sessions since arriving at the current weight block; 0 for non-primary sessions. |

## Methodology

The analysis has two parts:

1. **Exploratory visualizations** — reps over time by set, colored by weight; rest-pause/drop-set usage overlaid on performance; total reps per session stacked by set, annotated with weight-progression events.
2. **Inferential modeling** — an iterative series of OLS regressions (`total_reps` as the outcome), with HAC-robust standard errors to account for session-to-session autocorrelation. Each model iteration was motivated by inspecting the previous model's residuals and comparing them against my own training notes, adding: a lagged rep-count term (momentum), a flag for sessions where the weight changed, and a count of sessions spent at the current weight tier.

## Summary of Findings

Working weight and depth into the current weight block are the two dominant, consistently significant predictors of reps across every model iteration. Session-to-session momentum (how well the previous session went) also matters a great deal. Both the size of a weight jump and simply changing weight at all carry a measurable rep cost, separate from each other.

Rest-pause reps, drop sets, and exercise order never showed a reliable effect in any model at this sample size. An early apparent bodyweight effect mostly disappeared once accumulated time at a given weight was accounted for, suggesting it was partly standing in for practice time rather than reflecting added mass on its own.

On the 32kg question explored along the way: the model estimates the pure load penalty of 32kg vs. 27kg, at matched depth into the block, at around 1.6 reps, well below the ~4-rep gap it felt like in the moment. Current performance is tracking close to what the model expects at this stage, suggesting the discouraging early 6-rep sets reflected an expected new-tier/early-block cost rather than a sign the weight wasn't achievable.

## Caveats

This is a single-subject, exploratory inference exercise on a modest dataset, not a validated predictive model. Bodyweight and time-in-block remain somewhat entangled, since both trend upward together over the data's span. Several plausible factors (rest time between sessions, sleep, general fatigue) aren't tracked at all, so some of the unexplained variance is very likely real, unmeasured, session-to-session variation.

## Requirements

Built with Python 3, pandas, numpy, matplotlib, and statsmodels.


## Author
Anthony DiSorbo — Data Analyst, Greater Tokyo
[LinkedIn](https://www.linkedin.com/in/adisorbo/) · [GitHub](https://github.com/adisorbo)