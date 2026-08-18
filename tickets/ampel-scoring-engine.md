# Ticket: Ampel-Engine — rule-based daily factor rating (traffic light system)

**Status:** open — do NOT implement yet. Scheduled for the implementation of the final redesign mockups (v28+ line).
**Created:** 2026-08-18
**Depends on:** final mockup sign-off; `daily_logs` schema (see `Symptom Tracker/schema-design.md`)
**Replaces:** the placeholder logic in `mockups/nav-redesign-v28.html` (`SERIES`/`AMP_LEVEL`/`ampStep()`), which fakes the rating from a single synthetic per-factor value.

---

## Problem

The Heute tab communicates each factor (Ernährung, Schlaf, Stress, Bewegung, Wohlbefinden, Umwelt) as **one traffic light**. That light is a compressed verdict over many underlying variables — e.g. a day's Ernährung light summarizes fiber, omega-3, alcohol, processed food, sugar, etc. We need a deterministic backend system that turns the day's raw variables into that verdict, weighted by **how much each variable actually matters for psoriasis**:

- Day A: fiber slightly low, everything else fine → **green** (fiber alone is low-severity)
- Day B: omega-3 low **and** 2× processed food → **yellow** (two medium/high-severity deviations)

## Requirements

1. **No AI at scoring time.** Pure rules/parameters, deterministic and reproducible. (AI may still *extract* variables from meal text upstream — that's a separate, existing concern.)
2. **Scientifically grounded.** Severity weights derive from published evidence tiers, not gut feeling.
3. **Not personalized yet.** No use of the user's own correlations (`analysis_results`) in v1 — explicitly a later extension.
4. **Transparent and tunable.** All parameters live in database tables, not code. Every daily verdict stores *why* (“Warum gelb?” must be answerable in the UI).
5. **Honest about missing data.** Unknown ≠ good. Insufficient coverage → grey “nicht erfasst”, never a default green.

## Proposed design: weighted demerit points with dominance rules

Per factor, per day, three steps:

### 1. Variable → deviation score (0–3)

Each raw variable maps to a deviation degree via banded/piecewise-linear thresholds stored in the DB:

| deviation | meaning | example (Omega-3, target ≥ 2.5 g) |
|---|---|---|
| 0 | in target | ≥ 2.5 g |
| 1 | mildly off | 1.5–2.5 g |
| 2 | clearly off | 0.5–1.5 g |
| 3 | strongly off | < 0.5 g |

Count-type variables (e.g. “2× Verarbeitetes” from `meal_ingredients` categories) map counts to deviations the same way. Protective items (oily fish, vegetables, olive oil) may map to **negative** deviations (credits) — capped so they can soften small demerits but never cancel a hard rule.

### 2. Deviation × evidence weight = demerit points

Each variable carries a severity weight from its evidence tier for psoriasis:

| tier | weight | criteria | examples |
|---|---|---|---|
| A | 3 | guideline/RCT-level | alcohol, smoking, weight-relevant energy surplus, stress, sleep deprivation |
| B | 2 | consistent observational | processed food / pro-inflammatory pattern, added sugar, omega-3, Mediterranean adherence, exercise (protective), fiber/microbiome |
| C | 1 | anecdotal/mechanistic only | nightshades, gluten (default*) |

\* gluten is per-user configurable: with diagnosed celiac/sensitivity the user profile promotes it to tier A. This is a profile flag, not learned correlation.

`points(variable) = deviation × weight`, `total = Σ points − credits`.

### 3. Banding with dominance rules

Plain summation would let many harmless positives dilute one severe item, so the band check is twofold:

- **green** — `total ≤ T1` **and** no single item with `points ≥ D` (dominance threshold)
- **red** — `total ≥ T2` **or** a hard rule fired (e.g. `alcohol_units ≥ 3` on the day)
- **yellow** — otherwise

Worked example (T1 = 3, T2 = 8, D = 4):
- Day A: fiber deviation 1 × weight 2 = 2 points → total 2 ≤ T1, no dominant item → **green** ✓
- Day B: omega-3 (2 × 2 = 4) + processed 2× (2 × 2 = 4) → total 8 → **yellow/red boundary** ✓
- Any day with 3 alcohol units → **red** regardless of an otherwise perfect diet ✓

The continuous `total` also maps onto the mockup's −2…+2 display scale, so `tone()` (5-step color) and the 3-step wording stay consistent, exactly as v28 already separates them. Display-level hysteresis (`ampStep`) can stay a frontend concern applied on top of the engine's raw band.

### Missing data

Each factor defines a minimal coverage set (e.g. Ernährung: at least 2 logged meals). Below it → level `null` → grey ring in the UI (v28 already renders this state for Umwelt). Partially covered days score only on present variables and record the coverage ratio.

## Database schema (parameters + results)

```sql
-- versioned parameter sets: scores are reproducible and recomputable
create table scoring_param_sets (
  id          serial primary key,
  version     text not null,          -- e.g. '2026.1'
  notes       text,
  active      boolean default false,
  created_at  timestamptz default now()
);

-- one row per scored variable
create table scoring_variables (
  id             serial primary key,
  param_set_id   int references scoring_param_sets(id),
  factor         text not null,       -- 'ernaehrung' | 'schlaf' | 'stress' | ...
  key            text not null,       -- 'omega3_g', 'fiber_g', 'processed_count', 'alcohol_units', ...
  label          text not null,       -- UI wording, German
  direction      text check (direction in ('higher_better','lower_better','window')),
  bands          jsonb not null,      -- threshold edges → deviation 0..3 (or credit)
  evidence_tier  text check (evidence_tier in ('A','B','C')),
  weight         numeric not null,    -- default from tier, individually overridable
  hard_rule      jsonb,               -- e.g. {"gte": 3, "level": "red"}
  source_ref     text                 -- citation key, see Evidence section
);

-- band thresholds per factor
create table scoring_thresholds (
  param_set_id  int references scoring_param_sets(id),
  factor        text not null,
  t_yellow      numeric not null,     -- T1
  t_red         numeric not null,     -- T2
  dominance     numeric not null,     -- D
  min_coverage  jsonb                 -- what must be logged for a verdict
);

-- computed results incl. full explanation
create table daily_factor_scores (
  user_id        uuid,
  date           date,
  factor         text,
  points_total   numeric,
  level          text check (level in ('gut','mittel','ungut')),  -- null = not covered
  contributions  jsonb,   -- [{key, label, value, deviation, weight, points}, ...]
  coverage       numeric,
  param_set_id   int references scoring_param_sets(id),
  computed_at    timestamptz default now(),
  primary key (user_id, date, factor)
);
```

`contributions` is what powers the transparency requirement: the UI can render “Gelb, weil: Omega-3 niedrig (4 P.), 2× Verarbeitetes (4 P.)” without recomputing anything.

## Evidence base (to be verified before parameterization)

Initial tier assignments rest on, among others:

- **Ford AR et al., JAMA Dermatol 2018** — National Psoriasis Foundation dietary recommendations: weight reduction, Mediterranean pattern; gluten-free only with confirmed sensitivity.
- **Phan C et al., JAMA Dermatol 2018 (NutriNet-Santé)** — Mediterranean diet adherence inversely associated with psoriasis severity.
- **Brenaut E et al., JEADV 2013** — alcohol as trigger/severity factor (consistent association).
- **Armstrong AW et al., Br J Dermatol 2014** — smoking meta-analysis.
- **Snast I et al., Am J Clin Dermatol 2018** — psychological stress and psoriasis, systematic review.
- **Irwin MR et al., Biol Psychiatry 2016** — sleep disturbance → inflammatory markers, meta-analysis.
- **Frankel HC et al., Arch Dermatol 2012** — physical activity protective.
- **Afifi L et al., Dermatol Ther 2017** — patient-reported triggers incl. nightshades (anecdotal tier).
- Omega-3 RCTs are mixed but supportive as adjunct (e.g. Bittiner 1988) → tier B, not A.

**Editorial caveat (same spirit as the mockup's ENTWURFSVORBEHALT):** band edges, weights and thresholds in this ticket are working placeholders. Before launch they must be reviewed by dermatology/nutrition expertise, and each `scoring_variables` row must carry its `source_ref`.

## Explicitly out of scope (future versions)

- Personalized weights from the user's own lag correlations (`analysis_results`) — the schema anticipates this: a later param set can carry per-user weight overrides.
- Nutrient quantification from photos; the engine consumes whatever the extraction layer provides (ingredient categories at minimum, grams when available).
- Cross-factor interactions (e.g. alcohol × sleep).

## Acceptance criteria

1. Given a day's `daily_logs` rows, the engine returns per factor: `level`, `points_total`, `contributions`, `coverage` — deterministically (same input + same param set → same output).
2. The two scenario days from *Problem* above produce green and yellow respectively with the shipped default param set.
3. A hard rule (alcohol ≥ 3 units) forces red regardless of other variables.
4. A day below minimal coverage yields `level = null`, never green.
5. Changing a parameter in the DB (new param set version) changes subsequent scores without a code deploy, and old scores remain attributable to their param set.
6. Every score can be explained in the UI from `contributions` alone.
