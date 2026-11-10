# Research plan: conference article

**Title:** *Design of a Resource- and Network-Aware Cluster Selection Strategy for Karmada-Based Workload Placement*
**Type:** design-oriented paper; **no implementation and no experiments** (scope decision of 2026-11-10).
**Venue:** CSMD-UTM, section C2/C3; at least 10 pages (target 11–12) in CSMD formatting. See [conference_requirements.md](conference_requirements.md), including the paper-type check in §7.

Evidence and design documents:
- [literature_review.md](literature_review.md), [literature_matrix.csv](literature_matrix.csv)
- [notes/karmada_v1.19.1_verification.md](notes/karmada_v1.19.1_verification.md)
- [research_methodology.md](research_methodology.md): RQs, model, constraints, freshness rules, worked example
- [architecture.md](architecture.md): components, telemetry contract, pseudocode
- [future_validation_plan.md](future_validation_plan.md): thesis experiments

## 1. Research questions

| RQ | Question |
|---|---|
| RQ1 | How can resource and network information be incorporated into Karmada's cluster-selection process while keeping existing Karmada behaviour as the foundation? |
| RQ2 | How should mandatory placement constraints be separated from optimisation objectives? |
| RQ3 | How can stale or unavailable network measurements be handled safely? |
| RQ4 | What trade-offs does the design introduce? |

## 2. Contributions (as they will be stated)

1. A source-verified analysis of Karmada v1.19.1 scheduling semantics that matter for custom cluster selection:
   - unweighted score sum;
   - scores take effect only under spread constraints;
   - the capacity swap in selection;
   - request-based `resourceSummary`;
   - resource-model granularity;
   - sticky placement.
2. A Karmada-integrated design that:
   - separates hard constraints (Filter) from soft objectives (Score);
   - reuses Karmada's own capacity estimators;
   - adds client-to-cluster latency as a soft signal through Cluster annotations;
   - defines explicit validity and fallback rules for stale or missing measurements;
   - preserves default Karmada behaviour.
3. An analytical illustration of the design's decisions (worked example, decision regions over α, missing-data cases), and a validation protocol for future empirical work.

**Not claimed:**
- that combining resource and network signals is novel (prior work: L01–L03, L05);
- any improvement in latency or utilisation, since no evidence exists yet;
- optimality.

## 3. Article structure and page budget

CSMD format: TNR 12 pt, 1.5 spacing, A4, margins 2/2/2 cm and 2.5 cm left, roughly 450–500 words per full text page.

| § | Section | Content | ≈ pages |
|---|---|---|---|
| — | Title block, abstract (150–200 words), keywords (4–6, not repeating title terms) | | 0.7 |
| 1 | Introduction | Problem, motivation, RQs, contributions, paper outline | 1.0 |
| 2 | Background and Related Work | Multi-cluster orchestration; Karmada; node-level network-aware scheduling; cluster-level resource+network placement; comparison table vs L01–L05 | 2.2 |
| 3 | Problem Formulation and Research Objectives | Notation table, single-cluster selection problem, constraint vs objective separation, objectives | 1.0 |
| 4 | Proposed Architecture | Fig. 2, components, telemetry contract, existing vs proposed table | 1.2 |
| 5 | Cluster Selection Methodology | Karmada capacity information (Allocatable/Allocated/Allocating, request vs utilisation, fragmentation); H1–H4; Eq. (R), Eq. (N), Eq. (S); freshness rules | 1.8 |
| 6 | Algorithm Design and Integration with Karmada | Fig. 1 pipeline; Algorithm 1; integration facts (F1–F14 subset, with source locations); configuration | 1.3 |
| 7 | Design Analysis and Discussion | Worked example (Tab.), decision regions (Fig. 3), missing-data cases, complexity, decisions and rejected alternatives, advantages | 1.5 |
| 8 | Proposed Experimental Validation for Future Research | Strategies A/B/C/N, scenarios S1–S5, metrics, statistics, threats | 0.8 |
| 9 | Limitations | Version dependence, request-based view, aggregation, single vantage point, no empirical evidence | 0.4 |
| 10 | Conclusions and Future Work | Thesis extensions | 0.4 |
| — | References | about 28 entries | 1.4 |
| | **Total** | | **≈ 13.7**, trimmed to 11–12 when writing |

## 4. Workflow from here

| Stage | Status |
|---|---|
| Stage 1: research, requirements, literature | Done |
| Stage 2: research and architectural design | Done (this set of documents) |
| Stage 3: implementation and experiments | **Moved to the master's thesis** |
| Stage 4: LaTeX article | Next. Write and compile; LaTeX is not installed (see below). |
| Stage 5: critical academic review | After Stage 4 |

LaTeX is not installed on this machine. Compiling needs MiKTeX, TeX Live or Tectonic. This is a document-preparation tool, not experimental infrastructure, but installing it needs the user's approval.

## 5. Risks

| Risk | Mitigation |
|---|---|
| The CSMD template gives a 4-page maximum | User decision: ≥ 10 pages; confirm with the supervisor |
| Reviewers expect empirical results | Original findings come from the source-verified analysis and the design; the analytical example is clearly labelled; the validation plan is explicit; confirm with the supervisor |
| Karmada internals change after v1.19.1 | Pin the version and commit, and cite file locations |
