---
RFC: 0006
Title: K8sGPT LLM-Powered Kubernetes Cluster Monitoring
Author(s): Mani Thumu
Status: Draft
Created: 2026-05-29
Updated: 2026-05-29
Discussion: NA
---

Summary
-------
This RFC proposes adopting the [k8sgpt-operator](https://github.com/k8sgpt-ai/k8sgpt-operator) to integrate LLM-powered analysis into our Kubernetes cluster monitoring workflow. The operator runs as a native Kubernetes workload, continuously inspects cluster state, and emits human-readable diagnostic `Result` objects (and optional sink notifications) that explain anomalies and suggest remediation steps — augmenting existing Prometheus/Grafana observability with contextual AI reasoning.

Motivation
----------
Current cluster monitoring surfaces raw metrics and alerts but leaves root-cause analysis to engineers who must manually correlate events across pods, nodes, and workload-level resources — often without dedicated capacity for cluster operations. We run on AWS EKS where the Kubernetes control plane is fully managed by AWS; our operational scope covers workloads, nodes, namespaces, and AWS-integrated resources (ALB, EBS, IAM). This creates:

- **High mean-time-to-diagnose (MTTD):** Engineers context-switching from feature work spend significant time interpreting `CrashLoopBackOff`, OOM kills, pending pods, and misconfigured services before acting.
- **Alert fatigue:** Metric thresholds fire without actionable context, and with no dedicated ops rotation, alerts are easily missed or deprioritised.

k8sgpt continuously analyzes Kubernetes resources using configurable analyzers (Pods, Deployments, Services, HPAs, NetworkPolicies, etc.) and passes findings to a configured LLM backend. The LLM returns plain-language explanations with suggested fixes, which are stored as `Result` CRD objects and optionally forwarded to Slack or other sinks.

**Measurable goals:**
- Reduce MTTD for common cluster failures by surfacing LLM-generated root-cause summaries alongside raw metrics.
- Decrease escalation rate to engineers for routine anomalies.
- Provide a self-documenting audit trail of cluster health findings via `Result` objects queryable with `kubectl`.

Non-Goals
---------
- This RFC does not replace existing Prometheus, Grafana, or alerting infrastructure. k8sgpt augments, not replaces, those systems.
- This RFC does not propose automated remediation or any automated cluster mutations triggered by LLM output.
- Monitoring across multiple EKS clusters simultaneously is out of scope for this rollout; this RFC covers a single EKS cluster deployment only.

Guide-level explanation
-----------------------
Once deployed, the operator introduces two new Kubernetes custom resources:

**K8sGPT** – the configuration resource. A cluster admin creates one K8sGPT object that declares the AI backend, analysis interval, which analyzers to run, and where to send results. Example:

```yaml
apiVersion: core.k8sgpt.ai/v1alpha1
kind: K8sGPT
metadata:
  name: k8sgpt
  namespace: k8sgpt-operator-system
spec:
  ai:
    enabled: true
    backend: openai                          # OpenAI-compatible API (LiteLLM gateway)
    baseUrl: https://<litellm-endpoint>/v1   # LiteLLM proxy on CREATE
    model: <model-served-by-litellm>         # self-hosted vLLM model, or a Bedrock route
    secret:                                  # LiteLLM virtual key
      name: litellm-api-key
      key: api-key
  noCache: false
  filters:
    - Pod
    - Deployment
    - Service
    - HorizontalPodAutoscaler
    - NetworkPolicy
  sink:
    type: slack
    webhook: https://hooks.slack.com/services/...  # stored in a secret in practice
  interval: 5m
```

> **Auth note:** k8sgpt authenticates to the LiteLLM gateway with a **virtual API key** stored as a Kubernetes Secret (referenced via `spec.ai.secret`). LiteLLM holds all downstream credentials — it routes to the self-hosted vLLM model on CREATE, and can route to Amazon Bedrock (via IRSA on the LiteLLM side) if a Claude model is needed. This centralizes provider credentials in the gateway rather than in k8sgpt.

> **Prerequisites (LiteLLM gateway):** This integration is opt-in and requires setup before the `K8sGPT` CR will function:
> 1. **Reachable endpoint:** The LiteLLM proxy on CREATE must be reachable from the EKS cluster over HTTPS (it is currently internet-exposed). Confirm the cluster's egress can reach it.
> 2. **Model availability:** The `model` named in the CR must be served by LiteLLM — either the self-hosted vLLM model, or a configured Bedrock route.
> 3. **Virtual key:** Create a LiteLLM virtual key scoped to k8sgpt and store it as a Kubernetes Secret referenced by `spec.ai.secret`. Rate-limit the key at the gateway.
> 4. **Bedrock (optional):** If LiteLLM routes to Bedrock, wire IRSA/credentials scoped to `bedrock:InvokeModel` on the **LiteLLM** side — not on k8sgpt.

> **How `interval` works:** `interval: 5m` means k8sgpt scans the cluster every 5 minutes. A sink notification (e.g. Slack) is only sent when a `Result` is **first created** or when its content **changes** — not on every scan. A persistently broken pod will generate one alert, not one every 5 minutes.

**Result** – a read-only resource written by the operator. Each finding becomes a `Result` object:

```yaml
apiVersion: core.k8sgpt.ai/v1alpha1
kind: Result
metadata:
  name: default-nginx-pod-crashloopbackoff
  namespace: k8sgpt-operator-system
spec:
  kind: Pod
  name: nginx
  namespace: default
  error:
    - text: "Back-off restarting failed container"
      sensitive: []
  details: |
    The nginx Pod is in CrashLoopBackOff because the container exits immediately after
    start. Likely causes: missing environment variable, failed readiness probe, or image
    entrypoint misconfiguration. Check `kubectl logs nginx` and verify the container
    image entrypoint. If a ConfigMap is referenced, confirm the key exists.
```

Engineers query findings with `kubectl get results -n k8sgpt-operator-system` or receive them directly in Slack. No new dashboard is required to get started.

Reference-level design
----------------------

### Component layout

```
k8sgpt-operator-system namespace
├── Deployment: k8sgpt-operator          # operator controller
├── Deployment: k8sgpt                   # managed k8sgpt workload (created by operator)
├── CRD: k8sgpts.core.k8sgpt.ai          # configuration
└── CRD: results.core.k8sgpt.ai          # analysis findings

ClusterRole / ClusterRoleBinding
└── k8sgpt-operator-role                 # read access to all cluster resources under analysis
```

### Control flow

1. Admin applies a `K8sGPT` CR.
2. The operator controller reconciles the CR: creates the k8sgpt `Deployment`, `ServiceAccount`, and secret mounts.
3. The k8sgpt workload runs its analyzer loop at the configured interval (default `30s`, recommended `5m` for production).
4. For each anomaly detected, k8sgpt sends the raw diagnostic over HTTPS to the LiteLLM gateway, which routes to the configured model (self-hosted vLLM, or Bedrock if configured).
5. The LLM response is stored as a `Result` CR and (if configured) forwarded to the sink.
6. Stale `Result` objects are garbage-collected when the underlying issue resolves.

### AI backends

| Backend | Auth mechanism | Notes |
|---------|---------------|-------|
| LiteLLM gateway (OpenAI-compatible) | Virtual API key (K8s Secret) | **Primary.** Single gateway on CREATE; routes to the self-hosted vLLM model and centralizes keys, model routing, rate limits, and cost tracking |
| Amazon Bedrock (via LiteLLM) | IRSA on the LiteLLM side | **Optional** downstream route for Claude models (e.g. Sonnet) when higher capability is needed |

### Analyzers (configurable)

Core (enabled by default): `Pod`, `Deployment`, `ReplicaSet`, `StatefulSet`, `Service`, `Ingress`, `PersistentVolumeClaim`, `Node`

Advanced (opt-in): `HorizontalPodAutoscaler`, `NetworkPolicy`, `Log` (streams container logs to LLM — higher token cost)

### Caching

k8sgpt deduplicates analysis results using a local cache keyed on resource identity + error hash. For persistent cache needs aligned with our AWS infrastructure, the operator supports:
- Amazon S3 bucket (preferred — fits existing AWS setup)
- Interplex (built-in k8sgpt cache format, no external dependency)

### Sinks

Findings can be forwarded to:
- **Slack** via incoming webhook (primary)
- **CloudEvents** endpoint (for custom routing into existing alerting pipelines)

Sink credentials must be stored in a Kubernetes Secret and referenced in the `K8sGPT` spec.

Compatibility and migration
---------------------------
- This is a net-new addition with no impact on existing resources or APIs.
- The operator and k8sgpt workload run in a dedicated namespace (`k8sgpt-operator-system`) isolated from application workloads.
- Removal is clean: `helm uninstall` removes the operator, the managed deployment, and both CRDs (and therefore all `Result` objects). No cluster state is mutated by the operator beyond its own namespace.
- Scaling to more clusters: Each EKS cluster deploys its own independent k8sgpt operator (the standard pattern) — there is no central multi-cluster controller or remote kubeconfig injection. Adding a cluster means installing the operator there, not extending this deployment.

Alternatives considered
-----------------------
**Prometheus + manual investigation:** Already in place. Surfaces metrics but provides no language-level diagnostic context. Retained as the primary metrics layer; k8sgpt is complementary.

**Komodor / other commercial AIOps platforms:** Vendor lock-in, licensing cost, and data-egress concerns for a platform that sees all cluster events. k8sgpt is open-source, and routing through our own LiteLLM/vLLM gateway keeps diagnostic data on infrastructure we control rather than sending it to a third-party AIOps vendor.

**Custom in-house LLM integration:** Building a bespoke solution around the Kubernetes API + an LLM would replicate what k8sgpt already provides (analyzer logic, deduplication, CRD schema, sink integrations) at significant engineering cost with no differentiation.

**Deploying k8sgpt CLI without the operator:** The CLI is suitable for ad-hoc debugging but not continuous cluster monitoring. The operator brings the GitOps-friendly CRD model and lifecycle management needed for production use.

Operational considerations
--------------------------
**Rollout plan:**
1. Deploy on Stage (non-production) cluster first. Validate `Result` quality and gateway/model behavior (latency, request volume) over one week.
2. Tune the analyzer filter list and interval to match observed noise levels.
3. Enable Slack sink in `#cluster-alerts` (or a dedicated `#k8sgpt-findings`) channel.
4. Promote to production with conservative interval (`5m`) and no `Log` analyzer initially.

**Resource footprint:**
- Operator: ~50m CPU / 128Mi memory
- k8sgpt workload: ~100m CPU / 256Mi memory (higher briefly during analysis runs)
- Set resource requests and limits in the `K8sGPT` spec to enforce these.

**Observability of k8sgpt itself:**
- Monitor the k8sgpt workload pod for restarts (`kube_pod_container_status_restarts_total`).
- Alert if the `Result` CR count goes stale (no new objects over several intervals despite known issues) — indicates k8sgpt or LLM backend is down.
- Monitor LLM API error rates via operator logs (`kubectl logs -n k8sgpt-operator-system`).

**Rollback:** Delete the `K8sGPT` CR to stop the workload. The operator is removed via `helm uninstall` with no residual impact on cluster workloads.

**Feature flags:** Analysis scope is controlled declaratively via the `filters` list in the `K8sGPT` CR — no code changes required to enable/disable analyzers.

**LLM cost model:**
k8sgpt only calls the model when a `Result` is first created or its content changes — not on every scan. A persistently broken resource is queried once, not repeatedly. This keeps request volume low.

Because the primary model is **self-hosted on vLLM (CREATE)**, per-request inference has no marginal token cost — the cost is the fixed GPU-node capacity (shared infrastructure, not attributable per-scan). The only AWS-side cost is **public-internet egress** for the request/response payloads (resource metadata + error text, a few KB per finding, edge-triggered) — on the order of **cents/month** at expected volume.

If LiteLLM is configured to route to **Amazon Bedrock** for higher-capability analysis, per-token pricing applies for that path:

| Model (via Bedrock route) | Stable cluster (~5K tokens/day) | Active cluster (~50K tokens/day) |
|-------|----------------------------------|----------------------------------|
| Claude Haiku | ~$0.005/day (~$0.15/month) | ~$0.05/day (~$1.50/month) |
| Claude Sonnet | ~$0.03/day (~$1/month) | ~$0.33/day (~$10/month) |
| Log analyzer enabled | Unpredictable — significantly higher; not recommended initially | |

_Bedrock pricing based on on-demand rates as of 2026. Actual cost depends on anomaly volume and output length._

For a typical cluster, spend is negligible on the self-hosted path (egress only) and in the order of cents/day if routed to Bedrock. Costs spike only when there are many simultaneous new issues or if the Log analyzer is enabled. Model selection (self-hosted vLLM vs. Bedrock Claude Sonnet) and any Bedrock spend approval require manager and team sign-off before production rollout.

Security and privacy
--------------------
**Data sent to the model:**
k8sgpt sends resource metadata and error messages (e.g., pod names, container statuses, event messages) to the configured model via the LiteLLM gateway. It does **not** send pod environment variables, secret values, or volume contents. Sensitive field scrubbing is enabled by default and configurable.

**Threat model:**
- *Credential exposure:* k8sgpt holds only a LiteLLM **virtual key** (a Kubernetes Secret), not provider credentials. Downstream credentials (e.g. Bedrock IAM) live on the LiteLLM gateway. Rotate and rate-limit the virtual key at the gateway, and scope it to k8sgpt only.
- *RBAC scope:* The k8sgpt service account requires cluster-wide read access to analyzed resources. Apply least-privilege: grant only `get`, `list`, `watch` on required resource groups; no write permissions.
- *Network egress:* The LiteLLM/vLLM endpoint on CREATE is **internet-exposed**, so k8sgpt makes outbound HTTPS calls over the **public internet** (not intra-AWS). This traffic must be TLS-encrypted and authenticated with the virtual key, and the gateway should be **IP-allowlisted** to the EKS cluster's egress addresses (e.g. NAT gateway EIPs). Control cluster egress via VPC Security Groups / AWS Network Firewall.
- *Result data:* `Result` CRs are stored in the cluster and contain LLM-generated text. Treat them as internal operational data subject to the same access controls as other cluster resources.

**Compliance:** Diagnostic data is sent to our **own self-hosted model** (vLLM on CREATE), not a third-party LLM provider — so no cluster data is shared with an external AI vendor. Note, however, that the data **leaves AWS and traverses the public internet** to reach CREATE; it does not stay within the AWS network. Confirm this cross-environment path (AWS EKS → KCL CREATE) meets data-residency requirements, and keep the channel TLS-encrypted and access-controlled. If a Bedrock route is enabled, that subset of traffic stays within AWS.

Testing strategy
----------------
**Pre-production validation:**
- Deploy on stage cluster. Deliberately introduce known failure conditions (e.g., image pull error, OOM, missing ConfigMap key) and verify that `Result` objects are created with accurate, actionable LLM explanations within one analysis interval.
- Verify that `Result` objects are garbage-collected after the failure is resolved.

**Analyzer coverage test:**
- Run through each enabled analyzer type with a crafted broken manifest; assert a corresponding `Result` is produced.

**Sink integration test:**
- Enable Slack sink with a test webhook; confirm findings appear in the designated channel with correct formatting.

**Security test:**
- Confirm the k8sgpt service account cannot write to any cluster resource.
- Confirm Secrets are not included in LLM request payloads by inspecting operator debug logs.

**Success metrics:**
- Mean time from anomaly introduction to `Result` creation < 1 analysis interval.
- LLM explanation rated "actionable" by the reviewing engineer in >80% of cases (qualitative review over 30 days post-rollout).
- No increase in LLM API errors or missed analyses over a 7-day baseline window.

Open questions
--------------
- **Model selection and spend approval:** The primary backend is the LiteLLM gateway routing to the self-hosted vLLM model on CREATE. The team has flagged an expectation of **at least Sonnet-level capability** — note that Claude Sonnet itself is not self-hostable on vLLM, so we need to either confirm the vLLM-served model meets that capability bar, or configure LiteLLM to route k8sgpt to **Bedrock Claude Sonnet** (per-token cost applies; see cost model). Requires manager and team sign-off before production rollout; if Bedrock is used, a spend budget alert in AWS Cost Explorer is recommended.
- **Analysis interval tuning:** `5m` is a starting point; production load and token usage may require adjustment after the staging validation week.
- **Slack channel strategy:** Dedicated `#k8sgpt-findings` channel vs. routing to existing `#cluster-alerts`? High-volume findings may warrant a separate channel with digest/summarization.

References
----------
- k8sgpt-operator GitHub: https://github.com/k8sgpt-ai/k8sgpt-operator
- k8sgpt CLI GitHub: https://github.com/k8sgpt-ai/k8sgpt
- k8sgpt Helm chart (GitHub): https://github.com/k8sgpt-ai/k8sgpt-operator/tree/main/chart
- k8sgpt Helm repo: `helm repo add k8sgpt https://charts.k8sgpt.ai/ && helm repo update`
- k8sgpt documentation: https://docs.k8sgpt.ai
- LiteLLM proxy documentation: https://docs.litellm.ai
- vLLM documentation: https://docs.vllm.ai
- Kubernetes Operator pattern: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/

