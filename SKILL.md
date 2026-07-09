---
name: adversarial-panel
description: >-
  Run a multi-model adversarial mutual review with facilitator synthesis — the
  "Fable 5-style" metacognition pattern, reproducible on Opus or any Claude
  model: 2+ panelists (different model families where possible) independently
  answer, adversarially critique each other across rounds, and the main
  session acts as facilitator, synthesizing agreements, live disagreements,
  and calibrated confidence into one answer. Use this whenever the user asks
  to debate or adversarially review a claim, hypothesis, design, decision, or
  answer with multiple models or perspectives; says things like "discuss with
  Opus and Codex", "get a second opinion", "red-team this", "devil's
  advocate", 敵対的レビュー, 相互レビュー, 多モデル討論, 合議, 議論させて;
  or wants a high-stakes answer with explicit uncertainty instead of a
  single-pass response. Also use proactively when the user challenges your
  confidence ("are you sure?") on something consequential.
---

# Adversarial Panel

Reproduce the metacognitive architecture explicitly: several models answer a
question independently, attack each other's answers, revise under fire, and a
facilitator (you, the main session) synthesizes the result. The benefit does
NOT come from "running multiple models" as such — it comes from two design
properties that you must engineer deliberately:

- **Decorrelation.** Reviewers only catch what the generator missed if their
  error modes differ. Same-model panels share blind spots, so confident
  convergence is weaker evidence than it looks. Maximize heterogeneity;
  where you can't, say so in the synthesis.
- **Verification advantage.** Refuting a given answer is often easier than
  producing one. Point critics at *verifiable* claims (run the code,
  recompute the numbers, check the source) — "refute by reproduction, not by
  assertion."

Protect three invariants throughout: **independence** (Round-1 answers are
generated blind — a panelist who has seen another answer anchors on it),
**adversariality** (agreement without a new argument is a failed round), and
**synthesis without averaging** (preserve and adjudicate disagreements;
splitting the difference destroys the signal).

Conduct everything in the user's language.

## Roles

- **Facilitator** — you, the main session. You frame, dispatch, validate, and
  synthesize. Keep your own arguments out of the panel's mouth; if you add a
  view, label it as the facilitator's.
- **Panelists** — 2–4 reviewers. Maximize heterogeneity in this order:
  1. Different model families — an external CLI via Bash (check availability
     first, e.g. `codex --version`), if the user has one installed.
  2. Different Claude models via the Agent tool `model` parameter.
  3. Same model with forced-divergent *methods* (not just tones): one argues
     from first principles, one from base rates / outside view, one hunts
     only for disconfirming evidence, one re-executes verifiable claims.

Default: 2 panelists × 3 rounds. Scale to 3–4 panelists only when the user
asks for thoroughness; cost grows as panelists × rounds.

## Protocol

### Round 0 — Frame and triage (facilitator, no agents)

First decide whether the panel is worth it: if the question is low-stakes or
you are justifiably confident, answer directly and offer the panel as an
option. Invoke the full panel for questions that are consequential AND
contested or verifiable — that is where critique pays for its cost.

Beware the triage trap: this gate is run by the same model whose blind
spots the panel exists to catch, so the tasks where you're confidently
wrong are exactly the ones that won't trigger it. Therefore: if the user
explicitly asked for a panel, run it regardless of your confidence; and for
consequential questions, let stakes (not your felt certainty) make the call.

Then write a self-contained brief: the exact question, needed context,
constraints, deliverable format, and "separate facts from speculation;
attach a confidence and a falsification condition (what observation would
prove this wrong) to each key claim." Falsification conditions must be
concrete and checkable ("fails on input X", "contradicted by source Y") —
generic ones ("if evidence emerges to the contrary") are calibration
theater and count as missing. Subagents see none of your conversation — the
brief must stand alone.

### Round 1 — Independent answers (parallel)

Launch all panelists in parallel with the brief plus: "your final message is
the debate record; no preamble." No panelist sees another's output.

If panelists run as background agents, you may be paused and re-invoked as
each completes: treat every resume as "collect → validate → continue", and
never declare the task finished while a round is incomplete. If a panelist's
result seems to have gone missing, check its output file rather than waiting
indefinitely.

**Validation gate (do not skip):** when a panelist returns, check that the
text is a substantive answer. A status line ("task started in background",
tool notifications, an error dump) is NOT a contribution — if it enters the
record, later rounds critique a ghost and the panel silently degrades.
Re-run failed panelists, and for external CLIs require foreground execution:
the wrapper must wait for completion (generous timeout, e.g. 10 min), never
background the call and return early. If a panelist fails twice, drop to the
next heterogeneity tier and tell the user which panel actually ran.

### Round 2 — Cross-critique (parallel)

Give each panelist the OTHER panelists' Round-1 answers:

- Quote specific claims and attack them: factual errors, weak evidence,
  logical leaps, missing alternatives, unstated assumptions.
- For verifiable claims, refute by reproduction: run the code, recompute,
  check the cited source — not "I doubt this" but "this fails on input X".
- No agreement padding, no summaries, no praise. Concessions need reasons;
  so do attacks.
- End with at most a short list of genuine agreements.

### Round 3 — Final positions (parallel)

Give each panelist the critiques of their own answer: concede what was
rightly attacked (rightly — not socially), defend what survives with
reasons, state residual uncertainty, and give calibrated confidence with a
falsification condition. An unexplained full reversal is a sycophancy flag —
ask that panelist for the grounds of its new agreement before accepting it.

For a cheap question with 2 panelists, merge Rounds 2+3 into one
"critique, then restate your final position" round.

### Synthesis (facilitator)

Produce the user-facing output in this shape:

```
## 合意点 (Agreements)
What all panelists converged on, with the strongest single argument for it —
and whether convergence is strong evidence (heterogeneous panel) or weak
(same-family panel, shared blind spots possible).

## 対立点 (Live disagreements)
Each unresolved disagreement as: who claims what / on what evidence / your
adjudication with reasons — or an explicit "unresolvable with available
evidence" plus what evidence would resolve it. Distinguish real
disagreements from taxonomy differences (two panelists slicing the same
probability mass into different buckets is not a conflict). Weigh by
evidence strength: when one side has reproducible evidence and the other
has intuition, adjudicate — presenting them as symmetric "both views" is
false balance, not neutrality.

## 結論 (Calibrated conclusion)
The answer, confidence, what would change it, and any minority opinion worth
preserving. Note which critiques you accepted/rejected and why (audit
trail), and which heterogeneity tier the panel actually ran at — verified,
not intended: report the model you actually launched (e.g. the model
parameter you actually set), not the one you meant to.
```

Never present the synthesis as unanimous when it wasn't, and never cite the
panel as authority for a view that is actually your own.

## Degradation modes

- **No subagent tool**: run panelists as sequential, strictly separated
  inline sections; write each Round-1 section without re-reading the others.
  Weaker (shared weights AND shared context) — say so in the synthesis.
- **Only one model family**: force method-level divergence (tier 3 above)
  and downgrade the evidential weight of convergence in your synthesis.
- **External CLI fails twice**: drop the panelist, continue, disclose.

## Anti-patterns (each observed in practice)

- **Ghost panelist** — a status/error string treated as a contribution;
  every later round then debates thin air. The validation gate exists
  because of this.
- **Sycophantic convergence** — panelists agreeing by Round 2 without new
  arguments. Add "you disagree with at least one central claim; find it."
- **Facilitator capture** — laundering your prior opinion through the panel.
- **Confidence theater** — numeric confidence without a falsification
  condition attached.
- **Diversity illusion** — same-model personas treated as independent
  reviewers. A role name ("Red Team") does not decorrelate errors; only
  different models, different methods, or actual reproduction of claims do.
  Report convergence from same-family panels as weak evidence.
