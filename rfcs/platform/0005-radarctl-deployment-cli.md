---
RFC: 0005
Title: radarctl — Deployment CLI for RADAR-Kubernetes
Author(s): Yatharth Ranjan (@yatharthranjan)
Status: Draft
Created: 2026-05-18
Updated: 2026-05-18
Discussion: N/A
---

Summary
-------
This RFC proposes `radarctl`, a Go CLI tool that significantly improves the deployment experience for the RADAR-Kubernetes stack. It provides an interactive setup wizard, a deployment command with live progress and health checks, a status dashboard, and structured JSON output for agentic/CI workflows — all by shelling out to existing tools (kubectl, helm, helmfile) rather than reimplementing their logic.

Motivation
----------
Deploying RADAR-Kubernetes currently requires:

- Manually editing 3+ YAML files (`production.yaml`, `secrets.yaml`, `environments.yaml`) with no validation
- Understanding which `mods/` to compose for a given deployment profile (dev, staging, production)
- Running raw `helmfile sync` with no progress feedback or post-deploy health checks
- Diagnosing failures by manually `kubectl describe`-ing pods across ~30 releases
- A steep entry barrier for researchers and first-time operators who are not Kubernetes experts

`radarctl` addresses these by encoding institutional knowledge about the stack into an interactive CLI that guides users through setup, validates configuration before deployment, and provides a unified status and diagnostic view after deployment.

Non-Goals
---------
- Replacing helmfile or kubectl — `radarctl` shells out to these tools, it does not reimplement them.
- Secret manager integration (Vault, AWS Secrets Manager) — deferred to a future version.
- Upgrade orchestration — a future `radarctl upgrade` command is out of scope for v1.
- Windows support — macOS and Linux only for v1.
- Direct Kubernetes API calls — all cluster interaction goes through kubectl for v1.

Guide-level explanation
-----------------------
`radarctl` is a single binary distributed alongside RADAR-Kubernetes (at `cli/` in the repository). Users interact with five commands:

### radarctl init

The entry point for new deployments. Offers three modes:

- **Wizard** (recommended for first-time users): asks high-level questions ("Enable Fitbit data source?") and expands them into all required config values, only prompting for values it cannot infer (API keys, secrets).
- **Interactive**: field-by-field prompts for every config option with current values pre-filled.
- **Expert**: skips prompts, runs prerequisites check and config validation only.

Before any prompts, a prerequisites check verifies that all required tools are installed at the correct versions, the cluster is reachable, and resources are sufficient:

```
Checking prerequisites...

  ✓ kubectl        v1.30.2    (context: my-cluster, nodes: 3 Ready)
  ✓ helm           v3.15.1
  ✓ helmfile       v0.169.1
  ✓ helm-diff      v3.9.12
  ✓ yq             v4.44.3
  ✗ java           not found  (required for keystore generation)

1 prerequisite missing. Show install instructions?
```

The wizard flow:

```
1. Cluster basics        (hostname, email, kube context)
2. Deployment profile    (production / staging / dev — auto-applies relevant mods)
3. Kafka                 (local or Confluent Cloud)
4. Data sources          (Fitbit, Garmin, REDCap, upload portal — yes/no per source)
5. Storage               (local Minio or external S3)
6. Authentication        (Ory Hydra/Kratos)
7. Monitoring & logging  (Prometheus/Grafana, Graylog/Elasticsearch)
8. Review & confirm      (summary of choices, files to be written)
```

Wizard progress is saved to `.radarctl-state.yaml` (gitignored) and can be resumed if interrupted.

### radarctl deploy

Wraps `helmfile sync` with pre-deploy validation, a change preview, live progress, and post-deploy health checks:

```
radarctl deploy            # full sync
radarctl deploy --diff     # preview changes only
radarctl deploy --dry-run  # render templates, no apply
radarctl deploy --yes      # skip confirmation
radarctl deploy -o json    # structured output for CI/agents
```

Live progress display during sync:

```
Deploying RADAR stack...

  ✓ cert-manager          installed  (12s)
  ✓ kube-prometheus-stack installed  (45s)
  ⠸ mongodb               syncing...
  ○ kafka                 waiting
```

### radarctl status

A health dashboard across all deployed releases:

```
RADAR Stack Status — my-cluster (production)

INFRASTRUCTURE
  ✓ cert-manager            healthy    1/1 pods
  ✓ nginx-ingress           healthy    2/2 pods

KAFKA
  ✓ zookeeper               healthy    3/3 pods
  ✗ ksql-server             degraded   0/1 pods    CrashLoopBackOff
    └─ radar-ksql-0: OOMKilled — last 3 restarts in 10m

Summary: 19 healthy  1 degraded  1 warning
```

Supports `--watch`, `--component <name>`, `--show-urls`, and `-o json`.

### radarctl diagnose

Collects a full diagnostic snapshot (config validation, pod states, events, log tails) in a single JSON blob — designed for agentic loops to consume and act on.

### radarctl validate

Validates `production.yaml` and `secrets.yaml` standalone: required fields, no placeholder values, feature dependency consistency, mod compatibility.

Reference-level design
----------------------
### Repository location

`radarctl` lives at `cli/` inside the RADAR-Kubernetes repository, colocated with the helmfiles it manages.

### Architecture

```
cli/
├── main.go
├── cmd/
│   ├── root.go        # root command, global flags (--output, --context, --yes)
│   ├── init.go        # radarctl init
│   ├── deploy.go      # radarctl deploy
│   ├── status.go      # radarctl status
│   ├── diagnose.go    # radarctl diagnose
│   └── validate.go    # radarctl validate
├── pkg/
│   ├── config/
│   │   ├── loader.go      # read/write base.yaml, production.yaml, secrets.yaml
│   │   ├── validator.go   # validate completeness and consistency
│   │   └── features.go    # feature flag to config expansion
│   ├── wizard/
│   │   ├── wizard.go      # orchestrates wizard flow and mode selection
│   │   ├── questions.go   # question definitions and branching logic
│   │   └── writer.go      # writes collected answers to config files
│   ├── helmfile/
│   │   └── runner.go      # shells out to helmfile
│   └── kubectl/
│       └── runner.go      # shells out to kubectl
└── go.mod
```

### Key dependencies

| Dependency | Purpose |
|------------|---------|
| Cobra | Command structure and flags |
| Huh (charmbracelet) | Interactive terminal prompts and wizard forms |
| Viper | Config file reading/writing |
| go-yaml | YAML manipulation for config generation |
| pterm | Progress bars, spinners, status tables |

### Design principles

- **Shell out, do not reimplement** — use kubectl, helm, helmfile for all cluster operations.
- **Structured output everywhere** — every command supports `-o json` for agent/CI consumption.
- **Fail loudly with context** — errors include release name, pod name, and relevant log lines.
- **Progressive disclosure** — wizard mode for beginners, expert mode for power users.
- **Resumable** — wizard state saved to `.radarctl-state.yaml` (gitignored).

### Feature expansion

The wizard encodes knowledge about which config values each feature requires. Example:

- "Enable Fitbit?" sets `enable_fitbit: true` in `production.yaml`, prompts for `fitbit_client_id` and `fitbit_client_secret` in `secrets.yaml`, and adds the fitbit connector release to the enabled set.
- "Deployment profile: dev" auto-applies `mods/minimal + mods/localdev + mods/disable_tls + mods/fast_deploy`.

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Deployment failed (fixable) |
| 2 | Config invalid (needs human input) |
| 3 | Prerequisites missing |
| 4 | Cluster unreachable |

### Agent-friendliness

All commands support `-o json`. The intended agentic loop:

```
radarctl deploy -o json
  if failed: radarctl diagnose -o json
  agent proposes and applies fix
  radarctl deploy --yes -o json
  repeat until healthy or escalate
```

JSON schema for deploy result:

```json
{
  "status": "degraded",
  "releases": [
    { "name": "mongodb", "status": "healthy", "duration_s": 34 },
    { "name": "radar-appserver", "status": "failed", "error": "CrashLoopBackOff",
      "pod": "radar-appserver-6d4f9b-xkp2q", "logs": "..." }
  ],
  "summary": { "healthy": 21, "failed": 1, "pending": 0 }
}
```

Compatibility and migration
---------------------------
`radarctl` is purely additive — it does not change any existing files, scripts, or helmfile structure. Existing workflows (`bin/init`, `helmfile sync`, etc.) continue to work unchanged. The CLI is an optional layer on top.

Alternatives considered
-----------------------
- **Extend existing `bin/` shell scripts** — shell scripts do not scale well for validation logic, interactive prompts, YAML manipulation, and structured output across platforms. Rejected.
- **Python CLI** — richer library ecosystem but introduces a runtime dependency (venv, Python version) that operators must manage. A Go binary is self-contained. Rejected.
- **Separate repository** — cleaner release versioning but breaks the tight coupling between CLI logic and helmfile/config structure. Config changes and CLI logic must stay in sync. Rejected for v1.
- **Kubernetes operator** — a controller that manages the stack state declaratively. Powerful but a major architectural shift. Out of scope.

Operational considerations
--------------------------
- `radarctl` is distributed as a compiled binary built locally with `go build ./cli` or via a release workflow.
- No changes to helmfile, Kubernetes manifests, or existing scripts.
- The `--atomic` flag on deploy (on by default) ensures failed releases are rolled back automatically.
- CI pipelines can use `-o json` and exit codes to branch on failure.

Security and privacy
--------------------
- `radarctl` does not store or transmit secrets. It writes `etc/secrets.yaml` locally, identical to the existing `bin/generate-secrets` behaviour.
- The wizard prompts for secrets with terminal masking (no echo).
- `.radarctl-state.yaml` (wizard resume file) must not contain secret values — only non-sensitive config choices.
- The `--skip-prereqs` flag should be documented as for advanced use only.

Testing strategy
----------------
- Unit tests for `pkg/config/validator.go` and `pkg/config/features.go` — core validation and feature expansion logic.
- Integration tests for `pkg/helmfile/runner.go` and `pkg/kubectl/runner.go` using mock binaries.
- End-to-end test: run `radarctl init` in wizard mode against a k3d cluster using `mods/e2e.yaml`, then `radarctl deploy`, then `radarctl status` and assert all releases healthy.
- The existing `test/features/` BDD suite can be extended with a `radarctl_init.feature`.

Open questions
--------------
- Should `radarctl` be installable via `brew install` / `go install` as a standalone tool, or is it always used from within the cloned repository?
- Should wizard question definitions (`pkg/wizard/questions.go`) be data-driven (YAML/JSON config) to allow community contributions without Go knowledge?
- What is the recommended upgrade path when new config fields are added to `base.yaml` — should `radarctl init` detect and prompt for missing fields on re-run?
- Should `radarctl diagnose` optionally redact secret values before outputting JSON (for safe sharing in bug reports)?

References
----------
- RADAR-Kubernetes repository: https://github.com/RADAR-base/RADAR-Kubernetes
- radar-helm-charts: https://github.com/RADAR-base/radar-helm-charts
- Helmfile documentation: https://helmfile.readthedocs.io
- Cobra CLI framework: https://github.com/spf13/cobra
- Huh interactive forms: https://github.com/charmbracelet/huh
