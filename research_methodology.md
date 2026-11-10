# Research methodology and scheduling design (Stage 2)

**Article title:** *Design of a Resource- and Network-Aware Cluster Selection Strategy for Karmada-Based Workload Placement*
Article type: **design-oriented**, with no implementation and no experiments. Implementation, the laboratory, benchmarking and validation belong to the master's thesis; see [future_validation_plan.md](future_validation_plan.md).
Verified against: Karmada **v1.19.1** (tag commit `c64ba38`). File references `path:Lnn` are relative to the Karmada repo root. Unless marked otherwise, every Karmada statement below was checked by hand in this source tree on 2026-11-10.

## 1. Research type and questions

The study is constructive (design-oriented). Its outputs are:
- a source-level analysis of the existing system (Karmada v1.19.1);
- a design derived from that analysis;
- an analytical, deterministic illustration of the design's decisions;
- a validation protocol for the thesis.

Nothing in the article is presented as measured, and no claim of improved latency or utilisation is made.

| RQ | Question | How the article answers it |
|---|---|---|
| RQ1 | How can resource and network information be incorporated into Karmada's cluster-selection process while keeping existing Karmada behaviour as the foundation? | Source analysis of the extension points and selection semantics (§2), then the architecture ([architecture.md](architecture.md)) |
| RQ2 | How should mandatory placement constraints be separated from optimisation objectives in this process? | Filter/Score separation and capacity constraint via Karmada's estimators (§4–§5) |
| RQ3 | How can stale or unavailable network measurements be handled safely, so that missing data never counts as favourable? | Validity definition and fallback rules (§5.3) |
| RQ4 | What trade-offs does the design introduce? | Decision regions over α, missing-data cases, complexity analysis, and design decisions with rejected alternatives (§6–§8) |

Comparing default Karmada, resource-only and resource+network selection empirically is the **future validation plan** for the thesis. The article only illustrates their differing decisions analytically (§7).

## 2. Karmada facts the design depends on (RQ1)

### 2.1 Selection pipeline

The pipeline is `Schedule` → `findClustersThatFit` (Filter) → `prioritizeClusters` (Score) → `selectClusters` → `assignReplicas` (`pkg/scheduler/core/generic_scheduler.go:L71–116`).

| # | Fact | Source | Consequence |
|---|---|---|---|
| F1 | Extension points are `Filter(ctx, *ResourceBindingSpec, *ResourceBindingStatus, *Cluster)`, `Score(ctx, *ResourceBindingSpec, *Cluster) (int64, *Result)` and optional `NormalizeScore(ctx, ClusterScoreList)`. There is no cycle state, PreScore, Reserve, Permit or extender. | `framework/interface.go:L85–89, L215–232` | Per-workload state must come from the binding spec. Comparisons across candidates are only possible in `NormalizeScore`. |
| F2 | Plugins are compiled into the scheduler binary (`app.WithPlugin(name, factory)`). The factory takes no arguments. | `cmd/scheduler/app/scheduler.go:L90–94`; `framework/runtime/registry.go:L31` | No framework handle. Clients or configuration must be captured in a closure. |
| F3 | Score plugins are summed **without weights**; `scorePluginsWeightMap` is never filled. | `framework/runtime/framework.go:L41, L156` | Weighting has to happen inside one plugin. |
| F4 | Any plugin error aborts the scheduling attempt. | `framework.go` `RunScorePlugins` | Missing telemetry must never become an error. |
| F5 | In-tree scores: ClusterAffinity always returns 0. ClusterLocality returns 100 if the cluster already hosts the binding, else 0, and 0 when `spec.Clusters` is empty. | `clusteraffinity/cluster_affinity.go:L97–99`; `clusterlocality/cluster_locality.go:L50–61` | At initial placement, every default score is 0. |
| F6 | With no spread constraint, every feasible cluster is selected regardless of score. | `spreadconstraint/select_clusters.go:L28–32` | Scores only matter with `spreadByField: cluster` and `maxGroups`. |
| F7 | Clusters are sorted by overflow order, then score (descending), then **available replicas** (descending), then name. | `spreadconstraint/util.go:L43–61`; `group_clusters.go:L372–377` | With equal scores, the default picks the cluster with the most available replicas. |
| F8 | For Divided placement, if the top-`maxGroups` clusters cannot hold `spec.replicas`, the scheduler swaps the lowest-scored one for the remaining cluster with the **most available replicas**, not the next-best score. | `select_clusters_by_cluster.go:L25–78` | Capacity must be enforced in Filter. Otherwise the swap discards the score ordering. |
| F9 | For Duplicated placement, available resources are ignored during selection. | `select_clusters.go:L71–80` | Use Divided + Aggregated with `maxGroups: 1`. |
| F10 | Available replicas are the minimum across the registered estimators. The general estimator is always present; the accurate gRPC scheduler-estimator is opt-in. Workloads without requests get `MaxInt32`, clamped to `spec.replicas`. | `core/util.go:L57–110`; `estimator/client/general.go` | Workloads must declare requests, or every cluster ties. |
| F11 | `estimatorclient.GetReplicaEstimators()` is exported. | `estimator/client/interface.go:L106` | A compiled-in Filter can reuse Karmada's own capacity estimate. |
| F12 | Placement is sticky. Changes to Cluster status or annotations do not requeue bindings. | `scheduler.go` `doScheduleBinding`; `event_handler.go:L326–354` | The article scope is the initial decision. |
| F13 | The scheduler snapshot lists Cluster objects from the informer at scheduling time and deep-copies them. | `cache/cache.go:L124–139` | Annotations written by a probe are visible at the next decision without requeueing. |
| F14 | Karmada does not filter unhealthy clusters itself. It relies on the `cluster.karmada.io/not-ready` and `unreachable` NoSchedule taints and the TaintToleration filter. | `generic_scheduler.go` (comment in `findClustersThatFit`); `cluster_controller.go:L573–595` | Health is already a hard constraint, the same for every strategy. |

### 2.2 How Karmada knows a cluster's resources (Allocatable, Allocated, Allocating)

The cluster-status controller (Push mode) or the karmada-agent (Pull mode) computes these values on each status sync. The sync runs every `--cluster-status-update-frequency` (default 10 s), and the status is written only when it changed (`pkg/controllers/status/cluster_status_controller.go:L263–310, L558–613`).

| Field | Definition in code | Nature |
|---|---|---|
| `Allocatable` | Sum of `node.Status.Allocatable` over **every listed node** (`labels.Everything()`), including NotReady and cordoned nodes (`L527–539, L571–586`) | Node capacity minus kubelet reservations; aggregated |
| `Allocated` | Sum of pod requests for pods **with** `spec.nodeName` whose phase is not Succeeded or Failed. All namespaces, kube-system included. Plus the pod count. (`L601–613`) | Requested (reserved) resources of bound pods |
| `Allocating` | Sum of pod requests for pods **without** `spec.nodeName` (pending), plus the pod count (`L588–599`) | Requested resources of unbound pods |
| `AllocatableModelings` | Per-grade count of nodes, from each node's free resources (`L642–680`). Only populated when the `CustomizedClusterResourceModeling` gate is on (Beta, **default on**) | Grade histogram |

Pod requests = sum of container requests, raised to the largest single init-container request if that is higher, plus `spec.overhead` (`pkg/util/resource.go:L290–303`).

**Requested versus real utilisation.** All three fields are request-based. None reflects measured CPU or memory use. Consequences:
- Pods without requests (BestEffort) contribute nothing, so headroom is over-estimated on clusters with unrequested load.
- Pending pods that will never fit still count in `Allocating`, which is conservative.
- NotReady and cordoned nodes still count in `Allocatable`, which is optimistic.
- The data can be up to one sync period old, plus informer delay.

This is the same information Kubernetes admission works with, which is why the design uses it (decision D1 below). Measured utilisation (Prometheus) is a thesis extension.

**In-flight placements.** Until a scheduled workload's pods appear in a member cluster, its demand is missing from `Allocated` and `Allocating`. The `SchedulingOvercommitProtection` gate subtracts assumed in-flight workloads, but it is **Alpha and off by default** (`pkg/features/features.go:L184`; `core/util.go:L75–86`). Back-to-back placements within one status window can therefore overcommit a cluster. This affects every strategy equally and is listed as a limitation.

### 2.3 Is aggregate capacity enough? Fragmentation

A pod must fit on a single node. An aggregate free amount F ≥ request does not guarantee a fit. For example, three nodes with 1 CPU free each give an aggregate of 3 CPU, yet a 1.5-CPU pod fits nowhere.

Karmada offers three capacity views:

| View | Mechanism | Fragmentation-aware? | Notes |
|---|---|---|---|
| Summary path | ⌊(Allocatable − Allocated − Allocating) / request⌋ per resource, taking the minimum over resources and the pod limit (`general.go:L445–505`) | **No** (optimistic) | Used when `AllocatableModelings` is empty |
| Resource models | Each node goes to the grade given by the **minimum** of its per-resource grade indices (`modeling/modeling.go:L112–121`). Each model node gets that grade's **minimum** values as capacity (`general.go:L296–362`), and the result feeds a scheduling simulation. | Approximately (conservative) | **On by default.** Default grades are attached when a Cluster is created (`registry/cluster/strategy.go:L91–103`). Grade 0 is CPU [0, 1), memory [0, 4 GiB) with minimum 0, so a node with under 1 free CPU **or** under 4 GiB free memory contributes no capacity. Small nodes are heavily under-estimated. |
| Accurate scheduler-estimator | A per-member-cluster gRPC service simulating node-level fit (`MaxAvailableReplicas`) | Yes | Opt-in (`--enable-scheduler-estimator`); adds a network call per decision |

**Design decision D2.** The capacity constraint uses **Karmada's own estimator result**: call `GetReplicaEstimators()` and take the minimum (F11). This gives:
1. The plugin and Karmada's selection step always agree, so the F8 swap never fires on a candidate the plugin judged feasible.
2. Fragmentation handling is whatever the operator configured: resource models by default, or the accurate estimator if enabled. The plugin does not re-implement node-level fitting.
3. The plugin's limitations are those of the configured estimator, and the article states them.

Two notes on configuration:
- The default resource-model grades are too coarse for small clusters. Deployments with small nodes should either define finer `resourceModels` or enable the accurate estimator.
- `--enable-cluster-resource-modeling=false` is **not** a way to disable the models. It turns off the whole node and pod collection, so `ResourceSummary` is never filled and estimates fall to 0 (`cluster_status_controller.go:L263`). The right switch is the feature gate `CustomizedClusterResourceModeling=false`, accepted by both the controller-manager and the scheduler (`options.go:L226`, `scheduler options.go:L166`).

## 3. Notation

| Symbol | Meaning | Unit / domain |
|---|---|---|
| C | Set of member clusters registered with Karmada | — |
| C_F(w) ⊆ C | Clusters passing all hard constraints for workload w | — |
| w | Workload (one ResourceBinding) | — |
| n_w | Desired replicas (`spec.replicas`) | integer ≥ 1 |
| d_w^k | Per-replica request of resource k (`spec.replicaRequirements.resourceRequest`) | k = cpu: millicores; k = mem: bytes |
| K | Resources considered in scoring | {cpu, mem} |
| A_c^k, U_c^k, P_c^k | Allocatable, Allocated, Allocating of cluster c | as for d_w^k |
| F_c^k = A_c^k − U_c^k − P_c^k | Free requestable amount (aggregate) | as for d_w^k |
| m_c(w) | Available replicas, the minimum over Karmada's estimators | integer ≥ 0 |
| ℓ_c | Client-to-cluster round-trip time (median of the last q probe samples) | ms |
| t_c | Timestamp of the newest sample behind ℓ_c | s (UTC) |
| q_c | Number of samples behind ℓ_c | integer |
| τ | Freshness horizon | s |
| q_min | Minimum number of samples | integer |
| ℓ_ref | Reference RTT at which network suitability reaches 0 | ms |
| α | Resource weight; the network weight is 1 − α | [0, 1] |
| R_c(w), N_c | Resource and network suitability | [0, 1] |
| S_c(w) | Final plugin score | integer 0–100 |

## 4. Hard eligibility constraints (Filter)

A cluster is eligible, c ∈ C_F(w), only if **all** of the following hold. Constraints are evaluated before any score and are never traded against objectives.

| ID | Constraint | Implemented by | Same for all strategies? |
|---|---|---|---|
| H1 | API enablement, cluster affinity, tolerations of taints (including the automatic `not-ready`/`unreachable` NoSchedule taints, F14), eviction state | In-tree Filter plugins | Yes |
| H2 | Capacity: m_c(w) ≥ n_w (the whole workload fits, as required for single-cluster placement) | New `CapacityFit` Filter, which uses Karmada's estimators (D2) | Default Karmada enforces the same condition later, in selection (F8). The feasible set is identical, but the plugin enforces it **before** ranking. |
| H3 | Resource data present: `status.resourceSummary ≠ nil` | `CapacityFit` (m_c = 0 if it is missing) | Yes (the estimator returns 0) |
| H4 | *(optional, off by default)* Latency SLO: if the scheduler instance is configured with a hard bound ℓ_max, then ℓ_c must be **valid** (§5.3) and ℓ_c ≤ ℓ_max. Missing or stale data makes the cluster **ineligible** (fail-closed). | `NetLatency` Filter | No. Used only in a separately named configuration, never in the three-strategy comparison. |

Why H4 fails closed while the soft objective falls back: a hard constraint is a guarantee, and a guarantee cannot be checked without data. A soft objective is a preference, so it can degrade to a resource-only decision (§5.3).

## 5. Soft objectives (Score + NormalizeScore)

### 5.1 Resource suitability

R_c(w) = clip( min over k ∈ K of (F_c^k − n_w·d_w^k) / A_c^k , 0, 1 )   — Eq. (R)

- **Meaning.** The fraction of the bottleneck resource still free after placing w. It is relative to cluster size, so equal headroom fractions score equally on small and large clusters. This is the deliberate difference from Karmada's default tie-break, which uses the **absolute** count m_c(w) and therefore favours large clusters (F7).
- **Bottleneck (min) rather than mean.** A cluster with free CPU but no free memory should not score well. The minimum follows the binding constraint.
- **Data source (D1).** `status.resourceSummary`, i.e. request-based (§2.2), so the objective is consistent with Kubernetes admission. Aggregate F is acceptable **here** because R is a relative preference, and node-level fit is already guaranteed by H2.
- **Edge cases.**
  - A_c^k = 0: the cluster cannot be eligible under H2.
  - d_w^k = 0 for every k: the workload is BestEffort and outside the model; such workloads must declare requests (F10).

### 5.2 Network suitability

N_c = clip( 1 − ℓ_c / ℓ_ref , 0, 1 )   — Eq. (N)

- **Which latency.** The article covers single-tier, client-facing services, so ℓ_c is **client-to-cluster RTT** measured from a vantage point that represents the client population. **Service-to-service** latency applies to multi-tier workloads whose components could be split across clusters. Single-cluster placement (`maxGroups: 1`) keeps all components together, so inter-cluster service latency does not arise in the article's model. It is thesis scope (dependency graphs, split placement).
- **Absolute, not candidate-relative, normalisation.** With min-max over two or three candidates, the best cluster always gets 1 and the worst 0, whether they differ by 1 ms or 100 ms. Eq. (N) keeps scores comparable across decisions and makes α interpretable. ℓ_ref is set from the latency budget, e.g. the RTT share of the service SLO; it is a stated parameter, not a fitted one.
- **Estimator.** The median of the last q samples (q ≥ q_min), which is robust to single outliers. Using the mean or a high percentile is a sensitivity question for the thesis.

### 5.3 Missing or stale measurements

A measurement is **valid** iff the annotations exist and parse, q_c ≥ q_min, and t_now − t_c ≤ τ, with τ = 3 × probe period. Let V ⊆ C_F(w) be the eligible clusters with valid latency.

| Case | Rule | Rationale |
|---|---|---|
| V = C_F(w) | S_c = round(100·(α·R_c + (1 − α)·N_c)) | Normal operation |
| ∅ ⊂ V ⊂ C_F(w) | For c ∉ V, set N_c = 0 (**pessimistic**). The cluster can still win on resource headroom. | Missing data is never favourable. A neutral value (0.5) was rejected because an unmeasured cluster would beat one known to be slow. Exclusion was rejected because it turns a soft objective into a hard one. |
| V = ∅ | **Resource-only fallback** for all candidates: S_c = round(100·R_c), i.e. strategy B. A log line and a counter metric record the fallback. | No cluster is unfairly penalised relative to another; the decision degrades to a well-defined baseline. |
| Any parsing error | Treated as invalid. **Never returned as an error** (F4). | One bad annotation must not block scheduling. |

Implementation consequence: rules 2–3 need the whole candidate set, so `Score` returns per-cluster values and the case analysis runs in `NormalizeScore` (F1). This uses `ClusterScore.Cluster` to re-read each candidate's annotations.

**Clock skew.** Freshness compares the probe's timestamp with the scheduler's clock. Both must use synchronised clocks (NTP); τ ≫ expected skew.

### 5.4 Weight selection

- α is a configuration parameter **per scheduler instance**. The Score signature exposes only `ResourceBindingSpec` (F1), not the binding's annotations, so per-workload weights cannot be read without extra machinery. Workloads choose a configuration through the PropagationPolicy field `spec.schedulerName`. Each scheduler instance handles only bindings with its own name (`propagation_types.go:L134–139`; `event_handler.go:L380–386`).
- The article does not prescribe one α. It derives the **decision regions** of α for a given cluster state (§7.2) and leaves the empirical choice to the thesis.

### 5.5 Integer scores and ties

Scores are int64 in [0, 100] (`interface.go:L39–42`, not enforced at runtime). Rounding merges differences below 0.5 points. Ties are resolved by Karmada's deterministic order: available replicas, then name (F7). The selection is therefore a deterministic function of the snapshot.

### 5.6 Interaction with in-tree plugins

The default plugin set stays enabled in every strategy. At initial placement, ClusterLocality and ClusterAffinity both score 0 (F5), so the final score equals S_c. On a later reschedule (replica change), ClusterLocality would add 100 to the current cluster. That stickiness is outside the article's scope, and it is identical across strategies.

## 6. Complexity and overhead (analytical)

- **Filter.** One estimator call per candidate, so O(|C|) general-estimator evaluations. With the accurate estimator, add one gRPC round trip per decision (default timeout 3 s).
- **Score.** O(|K|) per candidate, no I/O. Annotations come from the in-memory snapshot (F13).
- **NormalizeScore.** O(|C|).
- **Per decision:** O(|C|·|K|) arithmetic plus the estimator cost, the same order as Karmada's existing pipeline, which already calls the estimators during selection.
- **Telemetry cost** sits outside the scheduling path: |C| probes every probe period, plus one Cluster annotation patch per cluster per period.
- **Measurement** (thesis): `karmada_scheduler_plugin_execution_duration_seconds{plugin=...}` and Go micro-benchmarks. The histogram's 1 ms minimum bucket is too coarse for per-plugin timing, so benchmarks are needed.

## 7. Analytical illustration (calculation, not measurement)

### 7.1 Scenario: latency and capacity in conflict

The values below are **chosen inputs** for a worked example. They are not measurements.

Workload: n_w = 4 replicas, d = 250 m CPU and 512 MiB per replica, so the total demand is 1000 m and 2 GiB. Parameters: ℓ_ref = 100 ms, α = 0.5, all latency data valid.

| Cluster | A_cpu (m) | A_mem (GiB) | F_cpu (m) | F_mem (GiB) | m_c | ℓ_c (ms) |
|---|---|---|---|---|---|---|
| edge | 4000 | 8 | 1200 | 3 | min(4, 6) = 4 | 5 |
| regional | 8000 | 16 | 4000 | 8 | 16 | 25 |
| central | 16000 | 32 | 12000 | 24 | 48 | 60 |

Here m_c uses the summary-path formula, assuming single-node clusters so that there is no fragmentation. All three clusters satisfy H2 (m_c ≥ 4).

| Quantity | edge | regional | central |
|---|---|---|---|
| R_c = min((F − n·d)/A) | min(0.050, 0.125) = **0.050** | min(0.375, 0.375) = **0.375** | min(0.6875, 0.6875) = **0.6875** |
| N_c = 1 − ℓ/100 | 0.95 | 0.75 | 0.40 |
| S_c (α = 0.5) | 50 | 56 | 54 |

| Strategy | Selected | Why |
|---|---|---|
| A: Karmada default | **central** | All scores 0, so the tie-break picks the most available replicas (48) (F5, F7) |
| B: resource-only (α = 1) | **central** | Highest R |
| C: resource+network (α = 0.5) | **regional** | Best combined score |
| Latency-only ablation (α = 0) | **edge** | Lowest RTT, but leaves only 5 % CPU headroom after placement |

### 7.2 Decision regions over α

With S_c(α) = α·R_c + (1 − α)·N_c:
- edge: 0.95 − 0.900·α
- regional: 0.75 − 0.375·α
- central: 0.40 + 0.2875·α

Setting these equal gives the boundaries:
- α < 0.381: **edge** wins;
- 0.381 < α < 0.528: **regional** wins;
- α > 0.528: **central** wins.

Integer rounding can move a boundary by up to one score point. The figure in the article will plot these lines. For the thesis, this predicts *where* the empirical latency and capacity trade-off should change, so the predictions can be tested.

### 7.3 Missing-data cases (same scenario)

| Condition | Effect | Selected |
|---|---|---|
| Only regional's latency is stale | N_regional = 0, so S = 19, 50, 54 | **central**; the stale cluster gains nothing from missing data |
| All latencies stale | Resource-only fallback, S = 5, 38, 69 | **central** (identical to B) |
| H4 enabled with ℓ_max = 30 ms | central is ineligible; regional and edge remain | **regional** (α = 0.5) |

## 8. Design decisions

| ID | Decision | Alternatives considered | Reason |
|---|---|---|---|
| D1 | R from request-based `resourceSummary` | Prometheus utilisation | Consistent with Kubernetes admission; available to the plugin without I/O; accepted by the user |
| D2 | Capacity constraint through Karmada's estimators | Re-implement the summary or model formula | Consistency with the selection step (F8); inherits the configured fragmentation handling |
| D3 | Single-cluster selection (Divided + Aggregated, `maxGroups: 1`) | Duplicated; multi-cluster splits | F6, F8 and F9: scores only matter with `maxGroups`; Duplicated ignores capacity; score-based splitting needs changes to Karmada core |
| D4 | Absolute normalisation (Eq. N, Eq. R) | Min-max across candidates | Stable, interpretable α |
| D5 | Pessimistic per-cluster plus global resource-only fallback | Neutral value; exclusion; error | §5.3 |
| D6 | Latency on Cluster annotations, written by an external probe | Plugin queries Prometheus inside Score | F2, F4, F13: no I/O or errors on the scheduling path |
| D7 | Parameters per scheduler instance | Per-workload annotations | F1: not visible to Score |
| D8 | No learned components | RL / ML | First establish an interpretable, analysable baseline |

## 9. Threats to validity of the analysis

- **Version dependence.** Every fact is pinned to v1.19.1. Several are internal behaviours (F3, F7, F8) that may change, and some are flagged in the code as TODOs.
- **Aggregation.** R uses aggregate data. If the estimator is optimistic (summary path, multi-node clusters), H2 may admit clusters where pods then stay Pending.
- **Request-based view.** Clusters with unrequested load look emptier than they are.
- **Single client vantage point.** ℓ_c represents one client population. Multiple regions need per-region measurements and a per-workload region selector; see D7.
- **Worked example.** It illustrates the decision logic only. It says nothing about response time; that needs the thesis experiments.

## 10. Existing functionality versus proposed extension

| Element | Existing in Karmada v1.19.1 | Proposed by this design |
|---|---|---|
| Filter/Score framework, `WithPlugin`, `--plugins`, `schedulerName` | ✔ | Used unchanged |
| Health via automatic NoSchedule taints and TaintToleration (H1) | ✔ | Used unchanged |
| `resourceSummary` (Allocatable/Allocated/Allocating) | ✔ | Read by the `NetResourceScore` plugin for R |
| Replica estimators (general, resource models, accurate) | ✔ (used in selection) | **Also** used before ranking, in the `CapacityFit` Filter (H2) |
| Spread constraint `maxGroups`, tie-break by available replicas | ✔ | Used unchanged |
| Client-to-cluster latency measurement | ✘ | Probe controller + Cluster annotations |
| Resource- and network-aware score with freshness rules | ✘ | `NetResourceScore` plugin (Score + NormalizeScore) |
| Optional hard latency bound (H4) | ✘ | `NetLatency` Filter (off by default) |
| Rebalancing, failover, descheduling | ✔ (separate mechanisms) | Not modified; thesis scope |
