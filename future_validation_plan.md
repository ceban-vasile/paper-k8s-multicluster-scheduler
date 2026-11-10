# Proposed experimental validation (master's thesis scope)

This is a plan only. **No part of it is carried out for the article.** The article summarises it in Section 8, "Proposed Experimental Validation for Future Research".

## 1. Objectives

| ID | Question to test empirically | Analytical prediction from the article |
|---|---|---|
| V1 | Does the network term change p50/p95/p99 response time relative to resource-only and default selection, when capacity is sufficient everywhere? | C chooses lower-RTT clusters (§7.1) |
| V2 | When the lowest-latency cluster has limited headroom, does the combined score avoid the saturation that latency-only selection produces, and at what latency cost? | The decision changes at the α boundaries computed in §7.2 |
| V3 | How does selection behave with stale or missing latency data? | The rules in §7.3 |
| V4 | What overhead does the plugin add per decision, and how does it scale with \|C\|? | O(\|C\|·\|K\|) plus the estimator cost (§6) |

## 2. Strategies compared (fair configuration)

| ID | Strategy | Configuration |
|---|---|---|
| A | Default Karmada | In-tree plugins |
| B | Resource-only | + `CapacityFit` + `NetResourceScore`, α = 1 |
| C | Resource + network | + `CapacityFit` + `NetResourceScore`, α ∈ {0.25, 0.5, 0.75} |
| N | Latency-only (ablation) | α = 0 |

All strategies share the same:
- workload and PropagationPolicy (Divided, Aggregated, `maxGroups: 1`);
- estimator configuration;
- clusters and scenario injection;
- load profile.

## 3. Environment (indicative)

- One Karmada control plane and three heterogeneous member clusters (kind or k3d; local VMs if `sch_netem` is unavailable).
- Heterogeneous allocatable resources set through kubelet reservations.
- Client-to-cluster delay injected with `tc netem` on each cluster's ingress path.
- k6 as the load generator on the client side; Prometheus and Grafana for observation.
- Workloads must declare requests and be CPU-bound with limits, so that saturation affects response time.

**Threat to validity:** with emulated delay, part of any latency gain exists by construction. The analysis must focus on the conflict scenarios (V2) and report gains relative to the emulated differences.

## 4. Scenarios

| ID | Scenario |
|---|---|
| S1 | Heterogeneous capacity, no pressure |
| S2 | CPU/memory pressure on one cluster (filler pods with requests) |
| S3 | Latency degradation on one cluster |
| S4 | **Conflict:** the lowest-latency cluster has limited headroom and a slower cluster has more capacity |
| S5 | Stale or missing telemetry (probe stopped for one cluster, then for all) |

## 5. Metrics

| Metric | Source |
|---|---|
| Response time p50/p95/p99 | k6 |
| SLO-violation rate | Share of requests over the SLO |
| Selected cluster per run | `ResourceBinding.spec.clusters` |
| Requested-resource share per cluster | `resourceSummary` |
| Measured CPU and memory utilisation | Prometheus (observation only) |
| Plugin duration | `karmada_scheduler_plugin_execution_duration_seconds` |
| End-to-end scheduling duration | `karmada_scheduler_e2e_scheduling_duration_seconds` |
| Scaling with \|C\| | Go micro-benchmarks for 3–100 synthetic clusters |

## 6. Protocol and statistics

- At least 10 repetitions per (scenario, strategy), in randomised order.
- A fixed warm-up period, then a fixed measurement window. The environment is reset between runs.
- Percentiles are computed per run. Report the median across runs with a 95% bootstrap confidence interval.
- Pairwise comparisons use non-parametric tests with effect sizes.
- Report null results.
- Use Papadopoulos et al. (IEEE TSE 2021) as the reporting checklist.

## 7. Further thesis extensions (beyond validation)

- Dynamic rebalancing: WorkloadRebalancer or a damped controller, with oscillation and hysteresis analysis.
- Service-to-service latency for multi-cluster splits.
- Measured-utilisation resource signals.
- eBPF/Cilium/Hubble telemetry.
- Multiple client regions.
- Comparison with an ILP optimum.
