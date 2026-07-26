# Horse Race Analyzer — Foundation & Model

**Version: 1.0.0 · July 2026**

This document is the reference for what the Analyzer is, how its model works, where its numbers come from, and how it is intended to evolve. It is the companion to README.md (setup and usage).

## 1. Provenance

The application is a functional tribute to the Mattel Electronics Horse Race Analyzer (1979, model #1670), a handheld "Handicapping Computer" built on a Texas Instruments TMS1100 microprocessor with 2 KB of ROM. Its handicapping program was developed by Dr. William L. Quirin, professor of mathematics and computer science at Adelphi University and author of *Winning at the Races: Computer Discoveries in Thoroughbred Handicapping* (1979), together with Evan Mandel (US Patent 4,382,280). After Mattel exited electronics, the rights passed to Advanced Handicapping Technologies, Inc., which sold the device into the late 1980s.

The exact ROM formula was never published. This rebuild reconstructs the model in the spirit of Quirin's published research — regression-derived impact factors with early speed as the dominant positive signal — using the same inputs the original device requested via its keypad (purse, distance, post position, days since last race, current record, lengths behind at points of call, finish positions, speed ratings). It is a Quirin-*inspired* formula, not a bit-for-bit ROM recreation.

## 2. Race eligibility (the 1979 rules)

Carried over from the original manual, a race qualifies only if all of the following hold: thoroughbreds; 3-year-olds and up (no 2-year-old-only races); not a maiden race of any kind (maiden special weight, maiden claiming, maiden optional claiming); distance of 6 furlongs or more. Non-qualifying races are shown greyed out with the reason. The manual's further guidance — prefer races where at least 90% of entrants show 5+ past races — is left to the user's judgment.

## 3. The rating model (v1 weights)

Each horse receives a rating that is the sum of seven components. A race is a **sprint** if under 8 furlongs, else a **route**; two components branch on this.

**3.1 Early speed (max 24; sprints ×1.25, max 30).** Quirin's strongest factor. For each of the last two races: `clamp(12 − 1.5 × lengths_behind_at_first_call, 0, 12)`. If only one race exists, it is doubled to the same scale.

**3.2 Recent form (max 20).** Last race only: finish-position points (1st=10, 2nd=8, 3rd=6, 4th=4, 5th=2; sprints de-weight these ×0.6, mirroring the original's sprint method that discounted late positions); plus `clamp(8 − lengths_behind_at_finish, 0, 8)`; plus 2 if the horse gained ground from the second call to the wire.

**3.3 Speed figures (max 24).** Average of the last two figures, normalized by scale: Beyer/BRIS ("modern"): `clamp((avg − 40)/60 × 24, 0, 24)`; 1979 DRF track-record ratings ("classic"): `clamp((avg − 60)/40 × 24, 0, 24)`.

**3.4 Class (max 12).** Earnings per start against today's purse: `clamp((earnings/starts) / (0.6 × purse) × 12, 0, 12)` — 0.6 approximating the winner's share.

**3.5 Consistency (max 12).** `clamp(win% × 8 + in-the-money% × 4, 0, 12)` from the current-year record (lifetime if no current-year starts).

**3.6 Freshness (max 8).** Days since last race: 7–30 → 8; 31–45 → 6; 4–6 → 5; 46–60 → 4; 61–90 → 2; under 4 → 3; over 90 → 0.

**3.7 Post position (−1 to +2).** Sprints: posts 1–4 → +2, 5–8 → +1, else 0. Routes: posts 1–7 → +1, 10+ → −1, else 0.

**Known blind spots (shared with the original):** no surface-switch awareness (turf form counts fully toward a dirt race), no trainer/jockey statistics, no workouts, no pace-matchup modeling, no track bias.

## 4. Win probabilities and the overlay calculation

Ratings are converted to approximate win probabilities with a softmax: `p_i ∝ exp(rating_i / 8)`, normalized across the field. The temperature (8) is a chosen constant, not a fitted parameter — **model % is an approximation, not a calibrated probability**, until real results say otherwise (see §7).

For each horse with odds `o` (decimal, x-to-1): board's implied win % = `1/(o+1)` (takeout not removed); fair odds = `(1−p)/p`; **edge** = `p × (o+1) − 1`, the expected profit per $1 if `p` were exact. Positive edge marks a candidate overlay; the largest positive edge gets the ★. The pari-mutuel takeout (~15–20%) means small positive edges are not true edges. Edges above roughly +40% more often indicate a data problem than a market gift — verify the horse's inputs first.

## 5. Data sources

**5.1 Brisnet Single Data File (primary).** Comma-delimited, ~1,435 fields per horse row, one row per entrant. Parsed locally and deterministically — identical file, identical output. Fields used (1-indexed): 1 track, 2 date, 3 race #, 4 post, 5 entry flag (S = scratched, skipped), 6 distance in yards (÷220 = furlongs), 7 surface, 9 race type (S/M/MO = maiden), 10 age/sex code ("AO…" = 2yo-only), 11 classification, 12 purse, 44 morning-line odds, 45 horse name, 86–90 current-year record, 97–101 lifetime record (fallback), 224 days since last race; per past race (10 slots): 256+ race date, 616+ finish position, 666+ lengths behind 1st call, 686+ lengths behind 2nd call, 746+ lengths behind at finish, 846+ BRIS Speed Rating. ZIP downloads are inflated in-browser (fflate). BRIS Speed Ratings ride the "modern" speed scale.

**5.2 DRF PDF / photos (fallback).** Read via Claude vision (model claude-sonnet-4-6, temperature 0 for run-to-run consistency), two-stage: a card scan listing races, then per-race extraction into a compact schema, with a salvage parser for truncated responses. DRF PDFs encode fractional-length superscripts as substitute glyphs, so vision extraction is good but not exact — the review screen exists to catch misreads. Requires the user's Anthropic API key (browser-only storage).

**5.3 Manual entry.** The full 1979 experience.

## 6. Architecture & persistence

Single-file static app (index.html) on GitHub Pages: React 18.3.1 (UMD), Babel standalone 7.26.4 (pinned — an unpinned Babel once broke the site), fflate 0.8.2. No build step, no server. Credentials (Anthropic API key, Supabase URL + anon key) live in browser localStorage behind the ⚙ SET drawer and are sent only to their respective services. The Claude-artifact variant (horse-race-analyzer.jsx) shares the same source minus keys/persistence.

**Supabase tables:**

- `hra_race_log` — one row per horse per logged race: track, race_date, race_no, distance_f, surface, purse, horse, post, rating, model_pct, odds (as entered at logging time), is_winner. Written when the user taps "Log result" and selects the winner. This is the training/evaluation dataset.
- `hra_cards` — one row per track+date (upsert): full parsed card JSON (race index, per-race horses including odds edits). Written on card load, on every tab switch, and after each logged result. Re-uploading the same file overwrites stored odds edits; reload via 📼 Saved cards instead.

Both tables use RLS policies open to the anon role — appropriate for hobby race data; do not co-locate with sensitive tables under the same exposure assumptions.

## 7. Model evolution roadmap

The 1979 weights are the starting point, not the destination.

1. **Accumulate** — log every played race (~10 seconds each). Optionally accelerate with Brisnet results files (~$0.50/track/day) for backtesting without waiting on live days.
2. **Calibrate** — with ~50+ races, compare model % to actual win rates by bucket (does the "25%" horse win 25%?), and check top-pick win rate and ROI at logged odds. This diagnoses *what* to fix.
3. **Refit** — with a few hundred races, fit a logistic regression of `is_winner` on the seven component scores (per-race normalized). The fitted coefficients replace the v1 weights; softmax temperature gets fitted rather than assumed. Ships as v2.x with the weights documented here.
4. **Extend** — only after refitting: candidate new features in priority order: surface-switch flag, BRIS Prime Power (field 251), pace pars vs. figures, trainer/jockey stats — each already present in the Brisnet file, each added only if it earns its coefficient.

## 8. Versioning

Semantic-ish: **major** = model change (new weights/features — anything that changes ratings), **minor** = feature additions, **patch** = fixes/cosmetics. The version appears on the device faceplate and in this doc. Model-affecting changes must be recorded in the changelog so logged predictions remain interpretable (a v1 rating and a v2 rating are different animals).

### Changelog

- **1.0.0 (2026-07-25)** — First versioned release. Quirin-inspired 7-component model (v1 weights, §3); Brisnet single-file parser with in-browser ZIP; DRF PDF vision fallback (temp 0); race tabs with 1979 eligibility filtering; one-line rows; overlay grid with editable odds and value sort; Supabase persistence (hra_cards) and result logging (hra_race_log); ⚙ SET credentials drawer.

## 9. Disclaimers

This is an analytical toy in the tradition of the original device — not betting advice, and not a calibrated probability model (yet). Even the best horse can lose on any given day; the 1979 manual said it first: *some unknowns just can't be predicted.*
