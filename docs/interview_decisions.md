# Setup Interview — Decisions, Challenges and Open Questions

> Running log of the setup interview for the registry-analysis agent study. Started 2026-10-16.
> Related files: `agent_system_prompt.md` (Condition 3 prompt), `study_protocol.md` (protocol draft).

---

## 1. Decisions made

| # | Topic | Decision | Notes |
|---|-------|----------|-------|
| D1 | Data for development | **Synthetic dataset first** (Phase 1) | Real pseudonymised data only in Phase 2. |
| D2 | Programming language | **Python** | Protocol says "Python or R"; Python chosen. |
| D3 | "Do not alter hypothesis" rule (Objective 9) | User asked to **remove** it | ⚠️ **Conflicts with protocol** — Condition 3 explicitly lists "avoid post-hoc alteration" and "label exploratory analyses". Recommendation: keep it in the Condition 3 prompt. **Needs re-confirmation.** |
| D4 | SEFAS MID | User will supply | Value and source not yet given. |
| D5 | Unit of analysis | User will supply | Feet vs patients not yet specified. |
| D6 | Loss to follow-up | Includes **death, revision, contralateral surgery and non-response** | See challenge C2. |
| D7 | Baseline HV severity | **Clinical grading** (not radiographic) | Number of levels not yet given; to be modelled as categorical. |

---

## 2. Challenges raised (to resolve)

**C1 — Data governance (Phase 2).** Pseudonymised data is still personal data under GDPR. The ethics approval and registry data-release agreement must cover processing by the model provider. Alternative: agent works on synthetic data only, and a human runs the final script locally on real data. Long-term, local execution (e.g. agent harness on own machine/server) keeps data on-premises.

**C2 — Reasons for loss to follow-up are statistically different.**
- *Death*: one-year SEFAS does not exist → exclude or handle separately; must **not** be imputed.
- *Revision*: SEFAS reflects the revision, not the index operation → censor, treat as outcome, or exclude.
- *Contralateral surgery*: score may be contaminated by the other foot.
- *Non-response*: the only category where multiple imputation is the natural tool.
→ Dataset needs a **reason-for-loss variable**; build it into the synthetic data.

**C3 — Pre-specification is unenforceable by prompt alone.** Proposed compromise (single pass): agent writes `analysis_plan.md` (model, covariates, handling of missing data) **before loading data**; anything beyond the plan is labelled exploratory. Makes Objective 9 scorable.

**C4 — Unit of analysis / clustering.** If rows are feet and patients can contribute two feet, the agent needs clustered SEs or mixed models.

**C5 — SEFAS MID must be fixed by the investigators**, not chosen by the agent. Consider a sensitivity range if the published value is uncertain.

### Challenges to the protocol itself

**P1 — Scoring rubric must be frozen before any agent runs.** Define "correct" per task (cohort N, estimate within tolerance of truth, CI coverage, handling of planted traps). Decide who scores and how to blind them to condition (output style reveals condition).

**P2 — "Reproduce human results" needs a tolerance.** Reference analyses may be in R/SPSS/Stata; Python defaults (MI, robust SEs, contrasts) differ. Pre-define a match (e.g. same direction, overlapping CI, estimate within X% of reference).

**P3 — Condition 2 needs its own frozen, neutral system prompt.** Otherwise the Condition 3 comparison is against an arbitrary baseline. Decide: build Condition 2 on the same model (required for "same underlying model") vs using Finch itself (would break that).

**P4 — Condition 1 cannot produce estimates.** Needs a separate scoring scheme (correctness of the described plan), or the comparison is uninformative.

**P5 — Keep the answer key out of the agent's workspace.** Generator script, true coefficients and trap list never mounted in the run environment.

**P6 — Model versioning.** Pin the exact model version; log it in every run; check availability across Phase 1 → Phase 2.

**P7 — Repeated runs.** Define number of repeats per task per condition and sampling settings (consistency is a secondary outcome → this is part of the sample size).

**P8 — Protocol is truncated.** Missing: outcomes/scoring, evaluation methods, human time measurement, analysis plan for the study itself.

---

## 3. Open questions

1. SEFAS MID — value and literature source?
2. Unit of analysis — feet or patients? Can one patient contribute two feet?
3. Severity grading — how many levels, and labels?
4. Does the (real) dataset have a reason-for-loss-to-follow-up variable?
5. Confirm D3 — keep or drop the hypothesis/exploratory rule in the Condition 3 prompt?
6. Who reviews/scores outputs? Blinding approach?
7. Condition 2: own build on the same model, or Finch?
8. Number of repeated runs per task per condition?
9. Rest of the protocol document (scoring/evaluation sections)?
10. Synthetic data: user has one, or should it be generated (with planted traps)?

---

## 4. Proposed environment (draft, for critique)

- **Harness:** Python script calling a **pinned model version** via API / Agent SDK.
- **Execution:** fresh, isolated **Jupyter kernel per run**, containing only dataset + data dictionary + task.
- **Conditions:** set by a **frozen system-prompt file** per condition (C1 no tools / C2 neutral / C3 registry-specific).
- **Logging per run:** notebook, full transcript, model ID, timestamp, seed.
- **Separation:** synthetic-data generator and answer key in a separate folder never mounted into runs.
- **Scoring:** performed afterwards against the **frozen rubric**.

### Proposed synthetic dataset traps (if generated)

- Known true effect of operative technique (and true null effects, e.g. Akin)
- Missingness in one-year SEFAS dependent on baseline SEFAS (MAR) — and possibly MNAR component
- Reason-for-loss variable incl. deaths (must not be imputed)
- Bilateral patients (clustering)
- Ceiling effect in SEFAS
- A post-operative variable the agent must **not** adjust for
- Severity as ordinal grading (must not be treated as continuous)
