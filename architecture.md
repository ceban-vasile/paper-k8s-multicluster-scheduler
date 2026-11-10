# Proposed architecture (Stage 2, design only)

This document describes the design. **Nothing here is implemented**; implementation is thesis work. The facts F1–F14 and decisions D1–D8 refer to [research_methodology.md](research_methodology.md).

## 1. Components

| # | Component | Status | Responsibility |
|---|---|---|---|
| 1 | karmada-apiserver, karmada-controller-manager, cluster-status controller / karmada-agent | Existing, unchanged | Store Cluster objects; fill `status.resourceSummary` every 10 s; apply health taints |
| 2 | Replica estimators (general; resource models; optional accurate estimator) | Existing, unchanged | Compute available replicas m_c(w) |
| 3 | **Extended karmada-scheduler** binary | Existing binary, rebuilt with out-of-tree plugins via `app.WithPlugin` (F2) | Runs the in-tree plugins plus three new ones (below) |
| 3a | `CapacityFit` (Filter) | **Proposed** | H2/H3: m_c(w) ≥ n_w using `GetReplicaEstimators()` (D2) |
| 3b | `NetLatency` (Filter) | **Proposed, optional** | H4: fail-closed latency bound; disabled unless configured |
| 3c | `NetResourceScore` (Score + NormalizeScore) | **Proposed** | Computes R_c, N_c and S_c; applies the freshness rules (§5.3) |
| 4 | **Latency probe controller** | **Proposed** | From the client vantage point, measures RTT to each member cluster's ingress endpoint every T_p seconds and patches Cluster annotations |
| 5 | Prometheus / Grafana | Existing tools, optional | Observability only (scheduler metrics, probe metrics). **Not** on the decision path (D6). |

## 2. Data flow at decision time

1. The user creates a workload (e.g. a Deployment with resource requests) and a PropagationPolicy with:
   - `schedulerName: netaware-scheduler`;
   - `replicaScheduling: {type: Divided, divisionPreference: Aggregated}`;
   - `spreadConstraints: [{spreadByField: cluster, minGroups: 1, maxGroups: 1}]` (D3).
2. Karmada's detector creates a ResourceBinding with `replicaRequirements` (per-replica requests) and `replicas`.
3. The extended scheduler takes the binding, because its schedulerName matches, and snapshots the Cluster objects from the informer cache (F13). The snapshot includes `status.resourceSummary` and the probe annotations.
4. **Filter.** The in-tree filters (H1) run, then `CapacityFit` (H2/H3), then optionally `NetLatency` (H4).
5. **Score.** `NetResourceScore.Score` computes R_c and reads ℓ_c from the annotations of each feasible cluster. The in-tree Score plugins return 0 at initial placement (F5).
6. **NormalizeScore.** `NetResourceScore` checks which candidates have valid measurements (the set V), applies the pessimistic or fallback rule, and writes the final S_c ∈ [0, 100].
7. **Select.** The `cluster` spread constraint with `maxGroups: 1` takes the top cluster in the order (overflow, score, available replicas, name) (F7). Because `CapacityFit` already ensured capacity, the swap in F8 does not fire.
8. **Assign.** Aggregated assigns all n_w replicas to the selected cluster, and the result is written to `ResourceBinding.spec.clusters`.
9. The binding controller creates the Work, and the execution controller or agent applies it to the member cluster. These steps are unchanged.

The probe controller runs **asynchronously** on its own period. It never blocks a decision.

## 3. Telemetry contract (Cluster annotations)

The prefix `netaware.lab/` is a placeholder; the real API group is chosen in the thesis.

| Annotation | Type | Example | Meaning |
|---|---|---|---|
| `netaware.lab/rtt-ms` | decimal string | `"24.7"` | ℓ_c: median of the last q samples (ms) |
| `netaware.lab/rtt-measured-at` | RFC 3339 UTC | `"2026-11-10T21:05:12Z"` | t_c: time of the newest sample |
| `netaware.lab/rtt-samples` | integer | `"5"` | q_c |
| `netaware.lab/rtt-vantage` | string | `"clients-chisinau"` | Which client population the value represents |
| `netaware.lab/probe-version` | string | `"v1"` | Schema version, for forward compatibility |

Rules:
- **Annotations, not labels.** Label changes requeue every binding (F12, `event_handler.go:L346`), so writing latency every T_p seconds as labels would cause scheduling churn. Annotations do not requeue, and they are still read at the next decision (F13).
- **Validity** follows §5.3: the values parse, q_c ≥ q_min, and the age is ≤ τ = 3·T_p. A `rtt-vantage` that differs from the scheduler instance's configured vantage counts as invalid.
- **RBAC.** The probe needs `patch` on `clusters.cluster.karmada.io` in the Karmada control plane and nothing else.
- **Measurement method.** RTT is measured by TCP connect time, or a minimal HTTP request, to the member cluster's service ingress, so it reflects the path a client actually uses rather than ICMP. The probe must run close to the clients it represents; this is an explicit assumption.

## 4. Scheduler configuration (per strategy instance)

| Instance | Score plugins | Parameters |
|---|---|---|
| `default-scheduler` (strategy A) | In-tree only | — |
| `resource-scheduler` (strategy B) | In-tree + `NetResourceScore` | α = 1 |
| `netaware-scheduler` (strategy C) | In-tree + `NetResourceScore` | α ∈ (0, 1), ℓ_ref, τ, q_min, vantage |

- `CapacityFit` is enabled in B and C. It does not change the feasible set relative to A; it only enforces capacity earlier (§4, H2).
- Parameters are passed as flags to the custom binary and captured in the plugin factory closures (F2, D7).
- Each instance needs a distinct `--scheduler-name` and leader-election resource name (Karmada docs, "Customize the scheduler").

Feature gates and flags relevant to correctness:

| Setting | Recommended | Reason |
|---|---|---|
| `CustomizedClusterResourceModeling` | On (default) **with finer `resourceModels` for small clusters**, or the accurate estimator | Default grades under-estimate small nodes (§2.3) |
| `--enable-cluster-resource-modeling` | Keep `true` | `false` disables the whole resource summary (§2.3) |
| `--enable-scheduler-estimator` | Optional | Fragmentation-exact capacity at the cost of a gRPC call |
| `SchedulingOvercommitProtection` | Document; Alpha, off by default | Back-to-back placements within one status window can overcommit |

## 5. Pseudocode

```
Algorithm 1: NetResourceScore — Filter, Score and NormalizeScore for workload w
Input: binding spec (n_w, d_w), feasible snapshot clusters, parameters α, ℓ_ref, τ, q_min, vantage
Output: integer score S_c ∈ [0,100] for each c ∈ C_F(w)

procedure CapacityFit(w, c)                                  ▷ Filter (H2, H3)
    if c.status.resourceSummary = nil then return Unschedulable
    m ← min over e ∈ GetReplicaEstimators() of e.MaxAvailableReplicas(c, w)
    if m < n_w then return Unschedulable
    return Success

procedure Score(w, c)                                        ▷ per cluster, no I/O
    R ← clip( min_{k∈{cpu,mem}} (A_c^k − U_c^k − P_c^k − n_w·d_w^k) / A_c^k , 0, 1 )
    return round(100·R)                                      ▷ provisional; refined in NormalizeScore

procedure Valid(c, now)
    parse rtt-ms, rtt-measured-at, rtt-samples, rtt-vantage   ▷ any parse failure → false
    return q_c ≥ q_min ∧ now − t_c ≤ τ ∧ vantage_c = vantage

procedure NormalizeScore(list L of (c, s))                   ▷ sees all candidates
    V ← { c ∈ L : Valid(c, now) }
    for each (c, s) in L do
        R ← s / 100                                          ▷ recompute R exactly in an implementation
        if V = ∅ then
            S_c ← round(100·R)                               ▷ resource-only fallback; increment fallback counter
        else
            N ← clip(1 − ℓ_c / ℓ_ref, 0, 1) if c ∈ V else 0  ▷ missing never favourable
            S_c ← round(100·(α·R + (1 − α)·N))
    return Success                                           ▷ never an error (F4)
```

Note for the implementation: R is recomputed from the snapshot in NormalizeScore, rather than reconstructed from the rounded score, so that rounding is not applied twice.

## 6. Diagrams to produce in TikZ (article)

1. **Fig. 1 – Karmada scheduling pipeline**: Filter → Score → (NormalizeScore) → Select → Assign, with existing and proposed plugins marked differently, and the F8 swap path annotated.
2. **Fig. 2 – Proposed architecture**: control plane, extended scheduler with three plugins, probe controller, member clusters A/B/C, clients; data paths labelled (status sync, annotation patch, binding, Work).
3. **Fig. 3 – Decision regions**: S_c(α) for the illustrative scenario (pgfplots; analytic lines, no data).
