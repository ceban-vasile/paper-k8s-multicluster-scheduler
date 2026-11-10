# Karmada scheduler: verified facts (v1.19.1)

Checked against Karmada **v1.19.1** (released 2026-10-31, tag commit `c64ba38cf7083c99bedaeed6acbdd1a8597fded1`) and karmada.io docs v1.19, accessed 2026-11-10.
Source paths are relative to the repo root: `https://github.com/karmada-io/karmada/blob/v1.19.1/<path>`.
✔ = re-checked by hand in the source after the research agent's report.

## Findings that shape the design

| # | Fact | Source |
|---|---|---|
| K1 ✔ | Score plugins **cannot be weighted**. `scorePluginsWeightMap` is declared and read but never set, so the total cluster score is an unweighted sum of plugin scores. | `pkg/scheduler/framework/runtime/framework.go` L41, L156; `core/generic_scheduler.go` `prioritizeClusters` |
| K2 ✔ | With **no SpreadConstraints** (or static weights), `SelectBestClusters` returns **all feasible clusters**, so scores do not change selection. Scores matter only with a `cluster`/`region` spread constraint with `maxGroups` (top-N by score, then swap for capacity). | `core/spreadconstraint/select_clusters.go` L28–32; `select_clusters_by_cluster.go` |
| K3 | Divided replica assignment does **not** use scores. StaticWeight uses only the weights. DynamicWeight and Aggregated order clusters by available replicas, and Aggregated also prefers previously used clusters. | `core/assignment.go`, `core/division_algorithm.go` |
| K4 ✔ | Placement is **sticky**. Rescheduling happens only on placement change, replica change, `rescheduleTriggeredAt`, or target-cluster deletion. Changes to Cluster status do not requeue bindings; only deletion, label changes or Generation changes do. | `scheduler.go` `doScheduleBinding`; `event_handler.go` `updateCluster` L326–354 |
| K5 ✔ | In-tree Score plugins: **ClusterAffinity** always returns 0. **ClusterLocality** returns 100 if the cluster already hosts the binding, else 0. "Locality" here means stickiness, not network locality. | `plugins/clusteraffinity/cluster_affinity.go` L97–99; `plugins/clusterlocality/cluster_locality.go` L50–61 |
| K6 | Extension points are **Filter** and **Score (+NormalizeScore)** only. There is no PreFilter, PreScore, Reserve, Permit or Bind, and no extender or webhook. | `framework/interface.go` |
| K7 | Plugins are added **at compile time** via `app.WithPlugin(name, factory)`. The factory takes **no arguments** (no framework handle or clients), and plugins are toggled with `--plugins` (e.g. `*,-ClusterLocality`). This is documented officially. | `cmd/scheduler/app/scheduler.go` L86–95; `framework/runtime/registry.go` L31; docs "Customize the scheduler" |
| K8 | **Multiple schedulers** are supported: `--scheduler-name` on the scheduler and `spec.schedulerName` in the PropagationPolicy. Each scheduler handles only its own bindings. | `options.go` L165; `propagation_types.go` L134–139; `event_handler.go` L380–386 |
| K9 | Score range is 0–100 (`MinClusterScore`/`MaxClusterScore`) but is **not enforced**. Plugins run serially per cluster, and **any plugin error fails the scheduling attempt**. | `framework/interface.go` L39–42; `framework.go` L126–170 |
| K10 | `Score` receives the full `*Cluster` object (labels, annotations, spec, status) as a DeepCopy from the informer snapshot, plus the `ResourceBindingSpec` (`ReplicaRequirements`, `Replicas`, `Placement`, current `Clusters`). | `pkg/scheduler/cache/cache.go` `Snapshot` |
| K11 | There is **no existing network, latency or cost awareness**: no plugin, field or proposal. Topology is limited to `provider`, `region`, `zones[]` and labels. | grep of `pkg/scheduler`, `docs/proposals` |

## Resource model and cluster health

- `Cluster.status.resourceSummary`: `allocatable`, `allocating`, `allocated`, `allocatableModelings[]` (`pkg/apis/cluster/v1alpha1/types.go` L346–378).
- Available replicas = **minimum** across estimators. The general estimator, always on, works from resourceSummary (allocatable − allocated − allocating, plus a pod limit) or from resource-model grades. The gRPC scheduler-estimator (`MaxAvailableReplicas`) is used only with `--enable-scheduler-estimator` (`core/util.go` `calAvailableReplicas`; `estimator/client/general.go`).
- Cluster status updates every 10 s by default (`--cluster-status-update-frequency`). Monitor grace period is 40 s.
- Automatic taints: `cluster.karmada.io/not-ready:NoSchedule` when Ready=False and `cluster.karmada.io/unreachable:NoSchedule` when Ready=Unknown. NoExecute taints come only from a user `ClusterTaintPolicy`.

## Rebalancing and failover (thesis scope; mention only)

- The descheduler covers Deployments with dynamic division only. It shrinks unschedulable replicas; it does not optimise placement.
- WorkloadRebalancer (since v1.10) sets `rescheduleTriggeredAt`, which triggers a Fresh reschedule.
- The Failover feature gate is Beta and **off by default**. `ClusterAffinities` groups (v1.5) and `overflowAffinities` (v1.18) also exist.

## Scheduler metrics (`pkg/scheduler/metrics/metrics.go`)

| Metric | Labels |
|---|---|
| `karmada_scheduler_e2e_scheduling_duration_seconds` | `result`, `schedule_type` |
| `karmada_scheduler_scheduling_algorithm_duration_seconds` | `schedule_step` = Filter/Score/Select/AssignReplicas |
| `karmada_scheduler_framework_extension_point_duration_seconds` | `extension_point`, `result` |
| `karmada_scheduler_plugin_execution_duration_seconds` | `plugin`, `extension_point`, `result` |
| `karmada_scheduler_schedule_attempts_total` | `result`, `schedule_type` |

Histogram buckets are 1 ms × 2^0..14, which is coarse; precise plugin overhead needs Go benchmarks. The docs' label values differ from the code (code: `scheduled`/`error`, `ReconcileSchedule`/`ScaleSchedule`). `scheduler_pending_bindings` sits in the legacy registry and may not be exported (unconfirmed).

## Extension options

| Option | Status | Use in the article |
|---|---|---|
| (a) Out-of-tree Filter/Score plugin in a rebuilt karmada-scheduler | Documented, supported | **Primary.** Scores take effect only with `spreadByField: cluster` + `maxGroups`. Disable ClusterLocality so stickiness does not add +100. |
| (b) Second scheduler via `schedulerName` | Documented, supported | **Evaluation harness.** Baselines A/B/C each run as a separate scheduler or plugin set, on separate test runs. |
| (c) External controller writing `staticWeightList`/ClusterAffinity | Public API | Thesis: damped rebalancing. Churn risk. |
| (d) Writing `ResourceBinding.spec.clusters` directly | Unsupported | Rejected. |

**Telemetry path for a plugin:** an external probe controller writes measured latency onto Cluster objects (labels or annotations, or a CRD), and the plugin reads them from the `*Cluster` argument without I/O. Labels requeue bindings; annotations do not. Network I/O inside Score is risky (K9).

## Other systems

- **kube-scheduler** has a full set of extension points (PreEnqueue … PostBind) and **weighted** score plugins (https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/).
- **KubeFed** was archived on 2023-04-25 (https://github.com/kubernetes-retired/kubefed).
- **Kueue MultiKueue** (Beta since v0.9; v0.20.5 checked) dispatches jobs to worker clusters by quota and admission (AllAtOnce/Incremental/external); it has no network awareness (https://kueue.sigs.k8s.io/docs/concepts/multikueue/).
