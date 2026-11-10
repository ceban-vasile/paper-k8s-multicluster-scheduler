# Literature review: resource- and network-aware workload placement in multi-cluster Kubernetes

Stage 1 deliverable, written 2026-11-10. The source IDs (L01–L41) refer to [literature_matrix.csv](literature_matrix.csv). The BibTeX lives in [notes/track_network_aware.bib](notes/track_network_aware.bib) and [notes/track_multicluster_and_docs.bib](notes/track_multicluster_and_docs.bib). The verified Karmada internals are in [notes/karmada_v1.19.1_verification.md](notes/karmada_v1.19.1_verification.md).

## 1. Method

**Search.** I ran two parallel searches, one for node-level network/latency/resource-aware Kubernetes scheduling and one for multi-cluster/federated placement. They used IEEE Xplore, ACM DL, SpringerLink, ScienceDirect, MDPI, USENIX and arXiv, queried through Crossref, Semantic Scholar and the publishers' own pages. A third search checked the primary technical sources: Karmada v1.19.1 source and docs, Kubernetes, KubeFed and Kueue.

**Inclusion.** I prioritised peer-reviewed work from 2023–2026. Older works are included only when they are foundational (Borg, Tail at Scale, Santos 2019) or methodological (Jain, Papadopoulos).

**Verification levels**, shown in the `Verification` column of the matrix:

| Level | Meaning | Sources |
|---|---|---|
| Full text checked by hand | Key claims re-read in the full text | L01 Tabatabaei, L02 CosmoSpan (key quotes), L36 Karmada source |
| Abstract verified | Metadata via Crossref/DOI and the abstract read. Claims are limited to what the abstract states. | Most sources |
| Metadata only | Bibliographic data verified. Do not cite specific claims or numbers until the paper is read. | L13, L15, L22, L32 |

**Excluded or held back:**
- Kinitos (JNCA 2025), Qiao et al. (Cluster Computing 2025) and Marchese & Tomarchio (ICSOC 2022): metadata verified, content not. Their numbers came only from search snippets.
- A NOMS 2024 RL multi-cluster paper whose author list could not be verified.

**Source mix.** 35 scholarly sources (32 peer-reviewed, including 5 surveys; 1 arXiv preprint; 1 practitioner magazine article; 1 book) and 6 primary technical sources. The article should cite about 25–30 of them.

## 2. Thematic analysis

### 2.1 Node-level network-aware scheduling inside one cluster (L17–L23, L26)

This is a mature line of work:
- Santos et al. began with RTT-aware fog provisioning [L18]. Their Diktyo framework [L17] made dependency-graph and latency/bandwidth-aware placement part of the upstream `scheduler-plugins` (`TopologicalSort`, `NetworkOverhead`) [L39].
- NetMARKS [L20] and NAS [L19] replace static topology descriptions with live service-mesh telemetry (Istio, Cilium/eBPF).
- Centofanti et al. [L21, L22] argue for application-layer, **user-perceived** latency instead of node-to-node latency.
- Marchese & Tomarchio add network-aware descheduling [L23].

**Critical observations:**
- All of these work inside **one control plane**. The network model is node-to-node within a cluster.
- Resource and network objectives are usually traded off, not combined. Diktyo's README recommends disabling `NodeResourcesFit` when `NetworkOverhead` is used. The Marchese & Tomarchio group treats load [L24] and network [L23] in separate papers.
- Evaluations often report mean latency or response-time reductions on small testbeds or single-host kind clusters [L26]. Few report tail percentiles with confidence intervals.

**Consequence for this article:** network-aware scheduling as such cannot be claimed as a contribution. The relevant question is how such signals behave at **cluster-selection** granularity, which differs from node selection:
- Cluster resource data is aggregated and request-based.
- Telemetry is coarser and potentially stale (Karmada updates cluster status every 10 s by default).
- Inter-cluster paths cross gateways rather than a flat pod network.

### 2.2 Multi-cluster orchestration substrates (L04, L07, L08, L15, L36, L40, L41)

| System | Model | Status / relevance |
|---|---|---|
| KubeFed [L40] | Central federation | Archived 2023-04-25 |
| Karmada [L36] | Central control plane with Push/Pull member clusters and its own scheduler | Our substrate |
| Liqo [L07] | Peer-to-peer resource sharing through virtual nodes | Alternative model |
| Oakestra [L08] | Hierarchical, non-Kubernetes. The root picks a cluster coarsely, then the cluster places pods. | Supports a two-level design: cluster first, then node |
| Kueue MultiKueue [L41] | Dispatches batch jobs by quota | Not latency-sensitive services, so out of scope |
| Decentralised federation [L15] | Position paper challenging centralised control planes | Context only |

**Karmada's actual scheduling behaviour** (verified in source, v1.19.1):
1. Extension points are only Filter and Score/NormalizeScore. Plugins are compiled in and toggled with `--plugins`.
2. The scheduler sums plugin scores without weights, despite a weight map that exists but is never filled.
3. The two in-tree Score plugins carry no information about resources or the network. ClusterAffinity always returns 0. ClusterLocality returns 100 for clusters that already host the workload.
4. Scores influence cluster selection only when a `cluster`/`region` spread constraint with `maxGroups` is set. Otherwise every feasible cluster is selected.
5. Replica division (StaticWeight, DynamicWeight, Aggregated) ignores scores. DynamicWeight and Aggregated use available replicas instead, estimated from `resourceSummary`, which counts requests rather than measured utilisation.
6. With equal scores, clusters are ranked by available replicas and then by name. For a new workload with `maxGroups: 1`, the default scheduler therefore already picks the cluster with the most capacity.
7. Placement is sticky. Changes in cluster status do not trigger rescheduling. Rebalancing needs a WorkloadRebalancer, a descheduler (which only handles dynamically divided Deployments) or failover (feature gate off by default).
8. There is no latency, network or cost awareness anywhere in the code or the design proposals.

CosmoSpan's authors describe the same limitation independently: Karmada "first distributes workload replicas across clusters according to user-defined policies without intelligent placement decisions" [L02].

### 2.3 Cluster-level placement using resource and/or network signals (L01–L06, L09, L10, L14)

This is the decisive category. **Combining resource and latency signals for cluster selection is not new.**

| Work | What it combines | How | Evaluation | Gap relative to us |
|---|---|---|---|---|
| **Tabatabaei et al. 2025** [L01] | Peak CPU load + latency, weighted per service; latency also a per-service bound | Karmada **Filter** plugin calls an external CPLEX ILP. Scoring "not yet included". | Python/CPLEX simulation with "fixed RTT values … without running actual service traffic". Testbed measures deployment time only. Baselines: load balancer, First Fit. | No measured application response time; no comparison to Karmada's default; external solver; initial placement only (stated by the authors) |
| **CosmoSpan 2026** [L02] | CPU, memory, uptime, bandwidth, latency, fault tolerance | Greedy MCDM scheduler picks **cluster and node jointly**, binds through the Karmada API (bypassing Karmada's scheduler), plus deschedulers | KWOK simulation scored on a composite placement score; physical multi-region deployment for overhead | "Karmada scheduler was not included in the comparison because no simulation framework currently exists for it"; latency is a hard filter |
| **LAIS 2024** [L03] | Latency intent (hard/soft); resources inside clusters | Latency-driven cluster choice plus rescheduler | Real multi-cluster deployment vs default K8s scheduler | Latency is the main objective; resources not jointly scored at cluster level; short paper |
| **mck8s 2021** [L04] | Resource policies + proximity | Controllers over KubeFed | Grid'5000 | Retired substrate; how the two are combined is not verified |
| **CODECO 2026** [L05] (preprint) | CPU, memory, bandwidth, latency, service class | Cluster ranking, then per-cluster optimisation, over OCM | No quantitative placement evaluation | Not peer-reviewed |
| Faticanti 2023 [L09], Khan & Khan 2025 [L14] | Resource cost + locality; inter-service latency | ILP / graph partitioning | Numerical / lightweight K8s | No Karmada; one-shot placement |
| Caspian 2024 [L10] | Carbon + deadline | Optimisation over MCAD | — | No network objective |
| Patiño-Martínez 2025 [L06] | — | Empirical study of Karmada reconfiguration | Karmada testbed | No placement policy |

### 2.4 Learning-based multi-cluster schedulers (L11–L13, L25)

DRL and multi-agent RL schedulers (KMPS [L11], DRKC [L12], RLSK [L13]) report gains in throughput and load balance. Their policies are opaque, they need training infrastructure, and in several cases "latency" means scheduling latency, not user-perceived latency. For a first study, this supports the brief's instruction: **establish a transparent, analysable heuristic baseline first**. Learned policies are thesis-level future work.

### 2.5 Measurement methodology (L33–L35)

- The tail-latency argument of Dean & Barroso [L33] justifies p95/p99 over means.
- Papadopoulos et al. [L34] list eight principles for reproducible cloud evaluation (repeated runs, reporting variability, describing the environment, etc.). They also found that few published studies follow them.
- Jain [L35] covers experimental design and confidence intervals.

The reviewed scheduling papers mostly report single-run or mean results. A rigorous protocol is therefore a legitimate, checkable strength of this article.

## 3. Research gap

Comparing the works above, the following combination is **not covered** by any verified source. Each point is supported by evidence:

| # | Gap | Evidence |
|---|---|---|
| G1 | **No measured tail-latency evaluation against Karmada's actual default behaviour.** Neither L01 nor L02 compares against Karmada's scheduler. L01's latency is modelled from fixed RTTs without service traffic; L02 reports a composite score in KWOK simulation. No verified study measures p50/p95/p99 response time of real traffic under controlled, emulated inter-cluster and client-to-cluster latency for Karmada's default vs resource-aware vs resource+network-aware placement. | L01 full text; L02 full text; L06 has no placement policy |
| G2 | **No transparent scoring heuristic in Karmada's native Score extension point.** L01 uses Filter plus an external ILP and explicitly leaves scoring out; L02 replaces Karmada's scheduler. Karmada itself has no such plugin and no weights, and scores only matter with spread constraints. An interpretable in-framework design that works with these constraints has not been documented or evaluated. | L01 quote; Karmada source K1–K5, K11 |
| G3 | **Telemetry staleness and missing data are unaddressed.** None of the cluster-level works specifies how stale or missing latency and resource measurements affect decisions. L01 uses fixed RTTs. | L01, L02, L03 abstracts/full text |
| G4 | **Decision overhead of in-scheduler, telemetry-driven scoring is unmeasured** at the plugin level. L02 reports platform overhead; L01 reports solver time for an external ILP. | L01, L02 |

**Gaps deliberately left to the thesis:** placement stability and rebalancing (oscillation, hysteresis, WorkloadRebalancer, deschedulers), service-to-service (dependency-graph) latency across clusters, bandwidth as an objective, eBPF/Cilium telemetry, and learned policies.

## 4. Positioning: claims we can and cannot make

**Cannot claim:**
- That combining resource availability and network latency for multi-cluster placement is novel. L01, L02, L03 and L05 already do it.
- That network-aware Kubernetes scheduling is novel (L17–L23).
- That any heuristic is "optimal". It is a heuristic; optimality would need comparison with an ILP such as L01's.

**Can claim** (article scope after the 2026-11-10 decision: design only, no experiments):
1. A characterisation of Karmada's scheduling semantics from the v1.19.1 source as they affect custom placement:
   - unweighted score sum;
   - scores only effective under spread constraints;
   - the capacity swap in selection;
   - request-based available replicas and resource-model granularity;
   - sticky placement.

   These are concrete, checkable facts, and several are undocumented.
2. A Karmada-integrated design that:
   - separates hard constraints (Filter) from soft objectives (Score);
   - reuses Karmada's capacity estimators;
   - adds client-to-cluster latency as a soft signal;
   - defines explicit freshness and fallback rules.
3. An analytical illustration of the design's decisions, and a validation protocol. **The empirical comparison (G1, G4) is planned for the thesis and not claimed in the article.**

Article title: *Design of a Resource- and Network-Aware Cluster Selection Strategy for Karmada-Based Workload Placement*. In the article's terms, gaps G2 and G3 are addressed by design, and G1 and G4 are named as open and planned.

## 5. Open verification items

| Item | Action |
|---|---|
| LAIS [L03] internals (Karmada-based? resource handling?) | Read the full text through UTM library access if available; otherwise cite only abstract-level claims |
| mck8s [L04] latency mechanism | Cite only "proximity-aware placement" as stated |
| Alhakimi & Latip 2026 [L30] | Check whether its "scheduling scopes" dimension names multi-cluster as an open area (OA, MDPI) |
| Jain 1991 | Confirm the subtitle against a physical copy before citing it |
| CSUR article numbers (L27, L28) | Fill in from the ACM DL before final submission, or omit |
