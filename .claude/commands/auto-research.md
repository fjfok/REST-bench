---
description: Karpathy-style AutoResearch ratchet loop on restaurant-manager.md — autonomous, never stops until interrupted
argument-hint: "[run-tag] [days] [seed] [cash]"
allowed-tools: Bash, Agent
---

# /auto-research — launch the ratchet loop on the restaurant-manager prompt

You are about to run an AutoResearch ratchet loop (Karpathy's pattern,
adapted per `autoresearch-build-your-own.md` in the repo root's parent) on
this repository. The **mutable artifact** is
`.claude/agents/restaurant-manager.md` — exactly one file, edited once per
iteration, exactly like `train.py` in the ML version of this loop. Everything
else is frozen. **Do not ask for confirmation between steps. NEVER STOP.**

## Arguments

`$ARGUMENTS`, positional, all optional:
- `$1` = run tag, kebab-case (default `run-<YYYYMMDD-HHMM>` using current local time)
- `$2` = days (default `30`)
- `$3` = seed (default `20260423`)
- `$4` = starting cash in EUR (default `20000`)

Fix these three (`$2`/`$3`/`$4`) for the *entire* run — every iteration must
play the identical month so `profit_eur` numbers are comparable. Do not vary
them between iterations under any circumstance; that would be equivalent to
editing the eval harness.

## Goal

Maximise **`total_net_profit_eur`** (end-of-month net profit) subject to two
hard constraints:

- `mean_reputation ≥ 4.0`
- `mean_satisfaction ≥ 0.75`

A run that violates either constraint does not advance the ratchet, no
matter how high its profit is — treat constraint violation like a crash for
ratchet purposes (see step 8). This is a hard-constraint + single-objective
collapse of a multi-metric goal, per the AutoResearch generalisation recipe:
never let profit trade off against reputation/satisfaction implicitly.

---

## Setup (once)

1. **Branch.** `git checkout -b autoresearch/<tag>`. If already on an
   `autoresearch/*` branch (resumed session), stay on it.

2. **Read in scope, in full.** `.claude/agents/restaurant-manager.md` (the
   only mutable file), `.claude/commands/play-month.md` (frozen — this is
   your harness spec, read it to know exactly how a month is played and
   scored), and the existing `results.tsv` if present (`tail -n 20`). Skim
   `README.md` once for context.

3. **Sanity-run the baseline** (see "Run one month" below) with the fixed
   `$2 $3 $4`. Confirm `scorecard` returns valid JSON with `bankrupt: false`
   fields present. If not, stop and surface the error — the harness is
   broken and fixing `rest_sim/*` is out of scope for this loop.

4. **Initialise `results.tsv`** if it does not exist. Exact header,
   tab-separated:

   ```
   commit	profit_eur	mean_reputation	mean_satisfaction	total_walkouts	bankrupt	status	description
   ```

   Append one row for the current HEAD with the baseline metrics. It is
   already gitignored — never `git add` it.

---

## Experimentation

**Ratchet metric.** `profit_eur` = `total_net_profit_eur` from
`python -m rest_sim scorecard`, computed fresh by you after the subagent
finishes — never from the subagent's self-reported post-mortem prose. Strict
inequality (`profit_eur > best_so_far` among *constraint-passing* rows)
advances the ratchet.

**Audit-only fields.** `score_eur` in the scorecard is the *engine's own*
composite score (different target thresholds: 0.78 satisfaction, 4.10
reputation — baked into `rest_sim/tuning.py`). Log it if you like, but it is
not the ratchet metric and must not drive keep/discard decisions — this loop
uses the user's thresholds (4.0 / 0.75), not the engine's.

**In scope.** `.claude/agents/restaurant-manager.md` only — the system
prompt, decision heuristics, examples, reporting style, and (if you choose)
the `model:` frontmatter field. Keep the YAML frontmatter valid and keep
`tools: Bash` intact — the harness invokes this file as a subagent
definition and it must stay loadable.

**Never weaken or remove** the file's existing "Rules of engagement"
anti-cheat clauses (never call `status --full`, never read/edit
`game/state.json` or `game/reservations.json` directly, never call `init`).
These exist so the agent can't peek at pre-generated future data. If an idea
needs to violate one of these to work, drop the idea.

**Out of scope — do NOT edit:**
- `rest_sim/*` (the simulation engine), `data/*`, `tests/*` — the frozen
  harness. Changing any of these invalidates every prior `results.tsv` row.
- `.claude/commands/play-month.md` — the harness spec.
- `pyproject.toml` — no new dependencies.
- `game/*` files — never read or write these directly, in the loop or in
  the prompt you're editing.
- The run parameters (`$2`/`$3`/`$4`) — fixed for the whole session.

**Hypothesis tracking.** Since this file has no `HYPOTHESIS` variable like
`train.py`, add/update a single HTML comment line right after the YAML
frontmatter closing `---`, e.g.:

```
<!-- HYPOTHESIS: proactive restocking 2 days ahead of supplier-news shocks -->
```

It's inert to the rendered prompt and gives you a diffable, grep-able
one-liner per iteration.

**Budget.** There is no engine-enforced wall-clock cap on a playthrough (no
`SIGALRM`, unlike the ML version's `TIME_BUDGET_S`). As a soft sanity
ceiling, a 30-day run should take roughly 2–5 CLI calls/day (~150 calls
total per Rules of engagement in the prompt). If the subagent appears
stuck/looping far past that, or `done` never reports true after $2 days'
worth of `advance` calls, treat it as a crash (step 6).

**Anti-gaming (non-negotiable).**
- Compute the ratchet metric only from your own fresh `scorecard` call — the
  subagent's self-reported numbers are for the human post-mortem, not for
  ratchet decisions.
- Never let the prompt instruct the subagent to read `game/*` files or call
  `status --full`.
- Every iteration plays the identical `$2 $3 $4` from a freshly reset game —
  no carrying state between iterations.

**What good iterations look like.**
- Sharper restocking heuristics (thresholds, lead time ahead of supplier
  news, avoiding substitution-penalty stockouts).
- Staffing that anticipates predicted peaks instead of reacting to them.
- Reservation-cap tuning to cut walkouts without leaving seats empty.
- Marketing/promo/happy-hour cadence and loyalty-toggle timing.
- Pricing tweaks on high-margin vs. low-margin items.
- Faster, more decisive responses to `news` alerts and `alerts` in `status`.
- Tightened reporting style so the subagent spends its turns deciding, not
  narrating.

---

## Run one month (the "harness call" — do this identically every iteration)

This replicates `/play-month $2 $3 $4` exactly; do not deviate from
`play-month.md`'s own steps.

```bash
python -m rest_sim init --days ${2:-30} --seed ${3:-20260423} --cash ${4:-20000} > run.log 2>&1
```

Then launch the **restaurant-manager** subagent (Agent tool) with the exact
brief from `play-month.md` step 2 (play day-by-day until `done`, then call
`scorecard`). Let it return its post-mortem — read it for colour only.

Then, yourself, independently pull the authoritative numbers:

```bash
python -m rest_sim scorecard > scorecard.json 2>> run.log
python -c "
import json
d = json.load(open('scorecard.json'))
print('profit_eur:', d['total_net_profit_eur'])
print('mean_reputation:', d['mean_reputation'])
print('mean_satisfaction:', d['mean_satisfaction'])
print('total_walkouts:', d['total_walkouts'])
print('bankrupt:', d['bankrupt'])
print('engine_score_eur:', d['score_eur'])
"
```

Never `cat run.log` in full — it can carry the reset command's stdout/stderr
plus scorecard errors; `tail -n 50 run.log` if you need to debug a crash.

---

## The Loop (9 steps — NEVER STOP)

1. **State check.**
   ```bash
   git status                    # must be clean
   git rev-parse --short HEAD
   tail -n 20 results.tsv
   ```
   Note the current best `profit_eur` among rows where
   `status == ok` (i.e. constraints passed).

2. **Edit `restaurant-manager.md`** with ONE new idea. Update the
   `HYPOTHESIS` comment line to a one-line, tab-free description.

3. **Commit.**
   ```bash
   git add .claude/agents/restaurant-manager.md
   git commit -m "<short hypothesis>"
   ```

4. **Run one month** exactly as specified above.

5. **Extract metrics** from the python one-liner's output.

6. **Handle crashes.** If `scorecard` errors, the game never reaches `done`,
   or the subagent stalls far past the soft budget:
   - `tail -n 50 run.log` to diagnose.
   - Record `status=crashed` in `results.tsv` (numeric fields blank/`nan` if
     unavailable), `git reset --hard HEAD~1`, go to step 1 with a different
     idea. Do not amend-and-retry — a full playthrough is too expensive to
     patch mid-run; just discard and try a cleaner idea next time.

7. **Append to `results.tsv`.** Tab-separated, one row:
   ```
   <short-sha>\t<profit_eur>\t<mean_reputation>\t<mean_satisfaction>\t<total_walkouts>\t<bankrupt>\t<ok|constraint_violated|crashed>\t<hypothesis>
   ```
   Set `status=constraint_violated` when `bankrupt` is true, or
   `mean_reputation < 4.0`, or `mean_satisfaction < 0.75` — even if
   `profit_eur` looks great.

8. **Ratchet decision.**
   - `bankrupt` true, or `mean_reputation < 4.0`, or
     `mean_satisfaction < 0.75` → constraint violated → discard:
     `git reset --hard HEAD~1`. The row stays in `results.tsv`; the commit
     vanishes from history.
   - Else, `profit_eur > best_so_far` (among prior `status=ok` rows) → keep.
     Ratchet advances.
   - **Simplicity tiebreak:** `|profit_eur − best_so_far| ≤ 1` (i.e.
     effectively tied) *and* the prompt edit strictly deleted lines →
     keep. Ratchet doesn't advance numerically, but the prompt gets
     simpler.
   - Otherwise (tie without simplification, or a profit regression that
     still passes constraints) → `git reset --hard HEAD~1`.

9. **Loop.** Go to step 1 with a fresh idea. Every ~5 iterations, scan
   `results.tsv` for hypothesis shapes you've already discarded (including
   constraint violations — e.g. "cut staffing to save payroll" that tanked
   satisfaction) and don't repeat them. Do not pause to ask whether to
   continue.

---

## NEVER STOP

Do not stop the loop unless one of the following is true:
- The user interrupts.
- An unrecoverable environment failure that can't be diagnosed without a
  human (e.g. `rest_sim` import error, `game/` unwritable).
- `profit_eur` on a constraint-passing run reaches ≥ €100,000 in a single
  month — implausible for a 22-table bistro; investigate for a scoring or
  engine exploit before celebrating.

"Should I continue?" is never a question you ask.

---

## Reporting to the user

After each iteration, emit **exactly one line**:

```
iter N | profit=€X (Δ ±€Y) | rep=A.BB sat=0.CC | KEPT|DISCARDED(reason) — <hypothesis>
```

`reason` for a DISCARDED row is one of `regression`, `constraint:reputation`,
`constraint:satisfaction`, `bankrupt`, or `crashed`. No tracebacks, no
per-day narration, no multi-paragraph summaries — the subagent's post-mortem
and `results.tsv` have the detail.

Every ~5 iterations (these runs are expensive — a full month is ~60–150
subagent tool calls), add one extra line:

```
best so far: profit=€X @ <short-sha> — rep=A.BB sat=0.CC — <hypothesis of best>
```

That's it.
