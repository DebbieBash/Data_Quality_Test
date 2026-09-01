App Daily Metrics — Monitoring & Alerting
What this does

This notebook checks app_daily_metrics.csv for three kinds of problems — missing data, internally inconsistent data, and statistically unusual data — and produces a single alerts_df table listing anything that fired. Each row in alerts_df follows the same schema, so a downstream dashboard can filter, sort, and display alerts from any of the five rules without needing to know which one produced them.

Data
Source: app_daily_metrics.csv
Grain: one row per app_id + platform + date
Coverage confirmed before building anything: 20 apps × 2 platforms × 90 days, no missing app/platform/date combinations, no duplicate rows, no null dates.
alerts_df schema
Column	Meaning
alert_id	Unique ID, assigned after all rules are combined
detected_at	When the monitoring job ran
metric_date	The date the alert is about
app_id, platform	Which app and platform the alert applies to
alert_category	completeness, consistency, or anomaly
alert_type	The specific rule that fired
severity	critical or warning
metric_name	Which metric or comparison triggered the alert
observed_value	The actual value found
expected_value	What it should have looked like, where applicable
deviation_score	How far off (z-score, for the anomaly rule)
description	Human-readable explanation

observed_value, expected_value, and deviation_score are NULL where they don't apply to a given rule — for example, a missing row has nothing to observe, so forcing in a placeholder number would be misleading rather than helpful.

The five rules
1. Completeness — missing rows

Logic: built the full expected set of app_id × platform × date combinations (a cross join of every distinct app+platform pair against every distinct date), then left-joined the real data onto it. Any expected combination with no match is a missing row.

Result: zero missing combinations found. Also checked separately for duplicate rows on the same app+platform+date — zero found.

Why this rule stays live even though it found nothing today: this dataset happens to be complete, but the rule exists to catch a day going missing in a future run, not to describe today's file. A monitoring system that only checks what's already known to be clean isn't doing its job.

2. Consistency — duration reconciliation

Logic: session_mean_seconds × sessions_total should approximately equal duration_total_seconds, since a mean is definitionally total ÷ count.

Threshold: I didn't pick a tolerance upfront. I first calculated the percentage gap across every row in the dataset and looked at the actual distribution: minimum 0.0003%, maximum 3.09%, average 1.48%. An initial 2% cutoff flagged about a third of the dataset — all of it clustered tightly between 2–3%, which is the signature of a threshold sitting inside normal rounding noise rather than above it, not a real data problem. I moved the threshold to 4%, comfortably above the observed maximum, and the flag count dropped to zero.

Result: zero rows exceed 4% at the current threshold, confirming this reconciliation holds across the full dataset once the tolerance reflects the data's actual variance rather than an assumed round number.

3. Consistency — reach cannot exceed sessions

Logic: reach_users (distinct users) can never exceed sessions_total (total session events) — a user can have multiple sessions, but session count can't be smaller than the number of distinct users who generated them. This is a hard logical rule, not a tolerance-based one.

Result: zero violations. Confirmed the boundary case is handled correctly — several rows have reach_users == sessions_total exactly (every user had one session that day), which is valid and correctly not flagged.

4. Consistency — median/mean session length ratio

Logic: session_median_seconds and session_mean_seconds aren't expected to match — a skewed distribution of session lengths (a few very long sessions, most short) is normal user behaviour and will naturally pull the mean away from the median. What's checked here is whether the ratio between them stays within a plausible range.

Threshold: looked at the full distribution of median / mean across the dataset: range 0.233–1.556. I manually inspected the rows at both extremes — all had plausible, well-formed values (no zeros, no absurd numbers), consistent with genuine right- or left-skew rather than a data error. Set the flag range at <0.15 or >1.65, with headroom above the observed extremes on both sides.

Result: zero rows outside this range.

5. Anomaly — reach_users rolling z-score

Logic: for each app+platform, calculate a rolling mean and standard deviation of reach_users over the trailing 28 days (excluding the current day), then compute how many standard deviations today's value sits from that baseline.

Guard against thin-window artifacts: early in an app+platform's history, the rolling window has very few prior data points, which produces an unstable, unreliable baseline. An early version of this check without a guard produced z-scores as extreme as ±21 — implausible for real variance, and traced back to rows only 2–3 days into an app's reporting history. Added a minimum-history requirement: a z-score is only calculated once at least 14 prior real days exist for that app+platform; earlier rows get NULL and are excluded from anomaly alerting entirely.

Threshold: with the guard in place, z-scores across the dataset (excluding the excluded early rows) range from -4.58 to 12.18, averaging close to zero as expected. Set the alert threshold at ±3 standard deviations — the conventional boundary for "statistically unusual" under a roughly normal distribution, and comfortably below the genuine outliers found in the data. Severity escalates to critical at ±5 or beyond, warning between 3 and 5, so the most extreme spikes are distinguishable from milder ones on the alert list.

Result: 51 rows flagged (of 3,040 rows with a trustworthy baseline, ~1.7%). Spot-checked several — genuine, sustained spikes with an established baseline behind them, not artifacts of the window being too thin.

Known limitations
The anomaly check only covers reach_users. The same rolling z-score pattern could be applied to sessions_total and session_mean_seconds, which I'd extend given more time.
The 28-day window and 14-day minimum-history requirement are reasonable defaults, not values tuned against a longer history than this dataset provides. In production I'd revisit both once more historical data accumulates.
No load timestamp is available in this dataset, so I can't distinguish "row arrived late" from "row is missing entirely." The completeness check catches full absence but not late arrival.
All thresholds (4% duration tolerance, 0.15–1.65 median/mean range, ±3 z-score) were derived by inspecting this dataset's own distribution, not agreed with a business stakeholder. In a live setting I'd confirm each with whoever owns the source data before relying on them for alerting.
How to run
Place app_daily_metrics.csv in the same folder as the notebook.
Run all cells top to bottom (Kernel → Restart & Run All is the cleanest check — confirms nothing depends on stale in-memory state from earlier exploration).
The final alerts_df table contains all alerts from all five rules, ready for output or further filtering.
