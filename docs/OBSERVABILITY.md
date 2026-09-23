# RHOAI Observability — Deployment Modes (3.6 draft)

> **Status:** Draft for docs team handoff.
>
> **Extends:** [RHOAI 3.5 — Manage observability](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/managing_openshift_ai/managing-observability_managing-rhoai) (dashboard details, workload scraping, tracing procedures).

---

## 1. Introduction

Red Hat OpenShift AI observability collects metrics, traces, and logs for platform and workload monitoring. Observability is configured on **Data Science Cluster Initialization** (`DSCInitialization`, abbreviated **DSCI**) at `spec.monitoring`. There is no single “mode” field — choose one of **four deployment modes** below based on which capabilities you need and whether telemetry stays in-cluster or goes to external systems.

**Install path:** [rhai-on-openshift-chart](https://github.com/opendatahub-io/odh-gitops/tree/main/charts/rhai-on-openshift-chart) as an OCI Helm chart. The chart installs the RHOAI operator, dependency operators (COO, OpenTelemetry, Tempo), and projects monitoring settings into the `DSCInitialization` CR via `services.monitoring.*` values. **Not in chart:** OpenShift Logging (CLO) and Loki — install from OperatorHub.

| Stage | OCI registry | Example version |
|-------|--------------|-----------------|
| **Pre-release / EA** | `oci://quay.io/rhoai/rhai-on-openshift-chart` | `v3.6.0-ea.2` — [Quay tags](https://quay.io/repository/rhoai/rhai-on-openshift-chart?tab=tags) |
| **GA** | `oci://registry.redhat.io/rhai/rhai-on-openshift-chart` | e.g. `v3.6` when published — [Red Hat catalog](https://catalog.redhat.com/en/software/containers/rhai/rhai-on-openshift-chart/69d5326b4f12a69777fa181d) (currently **3.5** only) |

---

## 2. Choose your deployment mode

| Mode | Intent | Required operators |
|------|--------|-------------------|
| **1 — All built-in** | Full stack: metrics, traces, logs, dashboards | COO, OpenTelemetry, Tempo, Loki, OpenShift Logging (CLO) |
| **2 — External only** | Forward telemetry to existing backends; no built-in storage/UI | OpenTelemetry, CLO |
| **3 — None** | No observability module | None |
| **4 — Custom** | Mix built-in and external per capability | See [§6](#6-mode-4--custom) |

**Mode 1** — Use when you want Prometheus, Tempo, Perses dashboards, and log aggregation inside the platform. Install all five operators before enabling monitoring.

**Mode 2** — Use when you already have enterprise observability (metrics/traces via OTLP, logs via your logging pipeline). RHOAI deploys collectors/forwarders only; no built-in Prometheus, Tempo, Loki, or Perses dashboards.

**Mode 3** — Use when observability is not required. No observability operator prerequisites.

**Mode 4** — Enable only the pillars you need (metrics, tracing, logs, dashboards). Install operators matching the capability matrix in [§6](#6-mode-4--custom).

---

## 2.1 Install with Helm (all modes)

Chart source: [odh-gitops](https://github.com/opendatahub-io/odh-gitops) — `charts/rhai-on-openshift-chart`.

### Chart location — OCI vs git clone

| Source | When to use | Registry login? |
|--------|-------------|-----------------|
| **OCI — Quay (pre-release / EA)** | `oci://quay.io/rhoai/rhai-on-openshift-chart` | **Yes** — `helm registry login quay.io` (Quay credentials with access to `rhoai/rhai-on-openshift-chart`) |
| **OCI — Red Hat (GA)** | `oci://registry.redhat.io/rhai/rhai-on-openshift-chart` | **Yes** — `helm registry login registry.redhat.io` (Red Hat Customer Portal). **3.6 tag not published yet** — catalog currently shows **3.5** only. |
| **Git clone** | [opendatahub-io/odh-gitops](https://github.com/opendatahub-io/odh-gitops) — branch **`main`** | **No** chart login — install from local path (chart `Chart.yaml` may lag OCI semver) |

```bash
# OCI — pre-release / EA
export CHART_VERSION=v3.6.0-ea.2
helm registry login quay.io
helm upgrade --install rhoai oci://quay.io/rhoai/rhai-on-openshift-chart \
  --version "${CHART_VERSION}" -n rhoai-gitops --create-namespace \
  --set operator.type=rhoai -f values-mode<N>.yaml

# OCI — GA (when v3.6 is published on registry.redhat.io)
export CHART_VERSION=v3.6
helm registry login registry.redhat.io
helm upgrade --install rhoai oci://registry.redhat.io/rhai/rhai-on-openshift-chart \
  --version "${CHART_VERSION}" -n rhoai-gitops --create-namespace \
  --set operator.type=rhoai -f values-mode<N>.yaml

# Git clone (upstream source mirror — main branch)
git clone https://github.com/opendatahub-io/odh-gitops.git
cd odh-gitops
helm upgrade --install rhoai ./charts/rhai-on-openshift-chart -n rhoai-gitops --create-namespace \
  --set operator.type=rhoai -f values-mode<N>.yaml
```

> **Note:** Versioned chart **tags** for 3.6 EA live on **Quay** (`v3.6.0-ea.2`, etc.). Red Hat Container Catalog publishes chart tags **with the product release** (currently **3.5**). The git repo has only `main` (no `rhoai-3.6` branch). For GA, use `registry.redhat.io/rhai` (not `quay.io/rhoai`).
>
> The chart installs `rhods-operator` from OperatorHub (`redhat-operators`, channel `beta` by default). Use a chart and OperatorHub channel that match your target RHOAI version.

Install CLO and Loki from OperatorHub when Mode 1 or Mode 4 logs pillar is required (see [§7](#7-operator-reference)).

### How dependencies work (same model as components)

Official chart reference: [How Component Dependencies Work](https://github.com/opendatahub-io/odh-gitops/tree/main/charts/rhai-on-openshift-chart#how-component-dependencies-work) — the **same resolution flow applies to `services`** (e.g. monitoring), not only `components`.

| Components (e.g. KServe) | Services (monitoring) |
|----------------------------|------------------------|
| `components.kserve.dsc.managementState` | `services.monitoring.dsci.managementState` |
| `components.kserve.dependencies.*` (boolean) | `services.monitoring.dependencies.*` (boolean) |
| Configures `DataScienceCluster` | Configures `DSCInitialization` (`spec.monitoring`) |

**Service-level dependency toggles** (`services.monitoring.dependencies.*`) — boolean only:

| Value | Meaning |
|-------|---------|
| `true` | Monitoring service requests this operator when active |
| `false` | Do not install this operator for monitoring |

**Top-level tri-state** (`dependencies.<name>.enabled`) — global override:

| Value | Behavior |
|-------|----------|
| **`false`** | **Never** installed — even if monitoring is `Managed` and `services.monitoring.dependencies.<name>=true` |
| **`true`** | **Always** installed — even if monitoring is `Removed` |
| **`auto`** (default) | Installed when the [resolution flow](#dependency-resolution-flow) below is satisfied |

#### Dependency resolution flow

1. Service is active (`services.monitoring.dsci.managementState` = `Managed` or `Unmanaged`)
2. `services.monitoring.dependencies.<name>` is `true` (or not set — defaults in chart values are `true` for COO, OTel, Tempo)
3. Top-level `dependencies.<name>.enabled` is `auto` or `true`
4. → Operator is installed via OLM (transitive deps apply, e.g. COO → OpenTelemetry)

**Recommended:** set `services.monitoring.dependencies.*` per mode; leave top-level `dependencies.*.enabled` at **`auto`**. The chart manages both operators and DSCI (`services.monitoring.dsci.*`).

### Two Helm value layers

| Layer | Values | Effect |
|-------|--------|--------|
| **Operators (OLM)** | `services.monitoring.dependencies.*` + `dependencies.*.enabled` | Resolution flow above |
| **Monitoring config (DSCI)** | `services.monitoring.dsci.*` | Writes `spec.monitoring` on `default-dsci` |

### Two-phase install

The first `helm upgrade --install` may create OLM Subscriptions only; `DSCInitialization` and other CRs are skipped until CRDs exist. **Re-run the same command** after operator CSVs show `Succeeded`. Do not use `helm install --wait` (can time out). See [Red Hat GitOps article](https://developers.redhat.com/articles/2026/08/26/automating-red-hat-openshift-ai-installations-with-helm-and-gitops).

### Base command

```bash
# OCI (GA registry — use Quay + CHART_VERSION for pre-release)
helm upgrade --install rhoai \
  oci://registry.redhat.io/rhai/rhai-on-openshift-chart \
  --version ${CHART_VERSION} \
  -n rhoai-gitops --create-namespace \
  --set operator.type=rhoai \
  -f <mode-values-file>.yaml

# Or local chart (no registry login)
helm upgrade --install rhoai ./charts/rhai-on-openshift-chart \
  -n rhoai-gitops --create-namespace \
  --set operator.type=rhoai \
  -f <mode-values-file>.yaml
```

Prefer a **values file** per mode (`-f`) for nested `metrics` / `traces` / `exporters` keys. Re-run after CRDs are ready. With `auto` (default), do **not** set top-level `dependencies.*.enabled=true` unless you intentionally want operators installed without `Managed` monitoring.

### Mode 3 — No observability

`values-mode3-none.yaml`:

```yaml
services:
  monitoring:
    dependencies:
      clusterObservability: false
      opentelemetry: false
      tempo: false
    dsci:
      managementState: Removed
```

### Mode 1 — All built-in

`values-mode1-builtin.yaml`:

```yaml
services:
  monitoring:
    dependencies:
      clusterObservability: true
      opentelemetry: true
      tempo: true
    dsci:
      managementState: Managed
      metrics:
        storage:
          size: 5Gi
          retention: 90d
      traces:
        sampleRatio: "0.1"
        storage:
          backend: pv
          size: 5Gi
          retention: 2160h
```

Also install **CLO** and **Loki** from OperatorHub before or after Helm (not in chart).

### Mode 2 — External observability only

`values-mode2-external.yaml`:

```yaml
services:
  monitoring:
    dependencies:
      clusterObservability: false
      opentelemetry: true
      tempo: false
    dsci:
      managementState: Managed
      metrics:
        exporters:
          debug:
            verbosity: detailed
          otlp/external:
            endpoint: https://metrics.example.com:4317
            tls:
              insecure: false
      traces:
        exporters:
          debug:
            verbosity: detailed
          otlp/external:
            endpoint: https://traces.example.com:4317
```

Also install **CLO** from OperatorHub for log forwarding. Do **not** enable COO or Tempo.

> **API note:** `traces.storage` is optional when using exporters only (external backends). CollectorReplicas may be set when metrics has storage **or** exporters, or when traces is configured.

### Mode 4 — Custom (examples)

**Metrics only** — `values-mode4-metrics.yaml`:

```yaml
services:
  monitoring:
    dependencies:
      clusterObservability: true
      opentelemetry: true
      tempo: false
    dsci:
      managementState: Managed
      metrics:
        storage:
          size: 10Gi
          retention: 30d
```

**Traces only (built-in PV)** — `values-mode4-traces.yaml`:

```yaml
services:
  monitoring:
    dependencies:
      clusterObservability: false
      opentelemetry: true
      tempo: true
    dsci:
      managementState: Managed
      traces:
        storage:
          backend: pv
          size: 5Gi
```

**Metrics + external remote write** — add under `dsci.metrics`:

```yaml
exporters:
  prometheusremotewrite/external:
    endpoint: https://prometheus.example.com/api/v1/write
```

### Verification (all modes)

| Check | Command |
|-------|---------|
| RHOAI operator | `oc get csv -n redhat-ods-operator` |
| COO / OTel / Tempo | `oc get csv -n openshift-cluster-observability-operator` (and OTel, Tempo namespaces) |
| DSCI monitoring | `oc get dsci default-dsci -o jsonpath='{.spec.monitoring.managementState}{"\n"}'` |
| Monitoring operands | `oc get pods -n redhat-ods-monitoring` |
| Mode 2 collector proof | `oc logs -n redhat-ods-monitoring -l app.kubernetes.io/component=opentelemetry-collector` (when DSCI applies and collector runs) |

---

## 3. Mode 1 — All built-in observability enabled

### Prerequisites

Use Helm per [§2.1](#21-install-with-helm-all-modes) (`values-mode1-builtin.yaml`). Additionally install from OperatorHub (**GA**):

1. Red Hat OpenShift Logging Operator (CLO)
2. Loki Operator

COO, OpenTelemetry, and Tempo are installed by the chart when `services.monitoring.dependencies` are enabled and monitoring is `Managed`.

See [§7 Operator reference](#7-operator-reference). For CLO/Loki install details, see [OpenShift Logging documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_logging/6.6/html/installing_logging/overview-of-openshift-logging-installation).

### Configuration

Helm: `values-mode1-builtin.yaml` in [§2.1](#21-install-with-helm-all-modes). Equivalent `DSCInitialization` fragment:

```yaml
spec:
  monitoring:
    managementState: Managed
    namespace: redhat-ods-monitoring
    metrics:
      storage:
        size: 5Gi
        retention: 90d
    traces:
      sampleRatio: "0.1"
      storage:
        backend: pv
        size: 5Gi
        retention: 2160h
```

#### Enabling built-in logs (LokiStack + ClusterLogForwarder)

Metrics and traces are projected from DSCI / Helm values. **Logs are not** in the DSCI v2 schema or chart Mode 1 values today — configure them on the platform `Monitoring` CR after Helm.

1. Provide **S3-compatible object storage** (AWS S3, MinIO, etc.) and a bucket for Loki.
2. Create a secret in the monitoring namespace (`redhat-ods-monitoring`) with keys:
   `access_key_id`, `access_key_secret`, `bucketnames`, `endpoint`, `region`, and typically `s3ForcePathStyle` / `insecure` for in-cluster S3-compatible endpoints.
3. Patch the `Monitoring` CR:

```yaml
spec:
  logs:
    storage:
      type: s3
      secretName: <your-s3-secret>
      credentialMode: static
      storageClassName: <storage-class>   # for LokiStack PVCs
```

4. Wait until `LokiStackAvailable` and `ClusterLogForwarderAvailable` are `True` on the Monitoring CR (`oc get monitoring default-monitoring -o yaml`).
5. **Pod Security:** CLO Vector collectors use `hostPath` and need a **privileged** Pod Security level on the monitoring namespace. Label the namespace and pin version labels so OpenShift does not re-sync to `baseline`:

```bash
oc label ns redhat-ods-monitoring \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/audit-version=latest \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/warn-version=latest \
  security.openshift.io/scc.podSecurityLabelSync=false \
  --overwrite
```

Without this, DaemonSet pods fail with `violates PodSecurity "baseline:latest": hostPath volumes`.

### Verification

| Check | Command / action | Result (Helm) |
|-------|------------------|---------------|
| Monitoring operands | `oc get pods -n redhat-ods-monitoring` | **Pass** — Prometheus, Alertmanager, OTel collector, Tempo, Perses, Thanos querier Running |
| Monitoring CR | `oc get monitoring default-monitoring` | **Pass** — `READY=True`, `REASON=Available` |
| Dashboards (CLI) | `oc get persesdashboard -A` | **Partial** — `data-science-tempo-traces` present; Cluster/Models PersesDashboard CRs not observed |
| Dashboards (visual) | OpenShift AI → Observe & monitor → Dashboard | _Not checked this run_ |
| Logs pipeline | CLO + Loki + Monitoring `logs` (S3) | **Pass** — Loki `v6.5.2` + CLO `cluster-logging.v6.6.1`; after S3 secret + `spec.logs` on Monitoring CR: `LokiStackAvailable` / `ClusterLogForwarderAvailable` True; collectors Ready after privileged PSA on `redhat-ods-monitoring`; marker log queried from Loki `application` tenant via LokiStack route. **Gap:** DSCI v2 / chart Mode 1 values still do not project `logs` — configure on Monitoring CR. |

---

## 4. Mode 2 — External observability only

### Prerequisites

Helm: `values-mode2-external.yaml` in [§2.1](#21-install-with-helm-all-modes) — OpenTelemetry via chart; **CLO** from OperatorHub.

| Operator | Purpose | Install |
|----------|---------|---------|
| Red Hat build of OpenTelemetry | OTel Collector and exporter pipelines | Helm (`services.monitoring.dependencies.opentelemetry=true`) |
| OpenShift Logging (CLO) | Log collection and forwarding | OperatorHub |

Do **not** install COO, Tempo, or Loki. **Logs use CLO**, not the OTel collector.

### Configuration

Helm values file sets `services.monitoring.dsci` (see [§2.1](#21-install-with-helm-all-modes)). Equivalent DSCI fragment:

```yaml
spec:
  monitoring:
    managementState: Managed
    namespace: redhat-ods-monitoring
    metrics:
      exporters:
        debug:
          verbosity: detailed
        otlp/external:
          endpoint: https://metrics.example.com:4317
          tls:
            insecure: false
    traces:
      exporters:
        debug:
          verbosity: detailed
        otlp/external:
          endpoint: https://traces.example.com:4317
```

Use a `debug` exporter during setup to confirm **metrics/traces** in collector logs. Replace `otlp/external` with your production endpoint.

### Verification

| Check | Command / action | Result (Helm) |
|-------|------------------|---------------|
| Helm apply exporters-only DSCI | `helm upgrade ... -f values-mode2-external.yaml` | **Fix in progress** — API/CRD now allows exporters-only (`traces.storage` optional; CollectorReplicas allows metrics exporters). Re-validate on cluster after operator + CRDs deploy. |
| Chart / cluster state after failure | DSCI unchanged from previous mode | Previously: Helm upgrade failed. Re-test after fix. |
| OTel Collector / external proof | — | **Pending** re-validation |
| Logs | CLO `ClusterLogForwarder` | **Not validated** — CLO not in chart |

> **Status:** Admission blockers for exporters-only Mode 2 are fixed in the operator API / DSCI CRD and vendored Monitoring CRD. Full Mode 2 e2e (collector debug exporter) still needs a cluster re-run after deploying this build.

---

## 5. Mode 3 — No observability

### Prerequisites

**None** — Helm: `values-mode3-none.yaml` in [§2.1](#21-install-with-helm-all-modes).

### Configuration

```yaml
# services.monitoring.dsci.managementState via Helm
spec:
  monitoring:
    managementState: Removed
```

### Trade-offs

No platform metrics, traces, or dashboards in the OpenShift AI console. Individual components may still expose application-level metrics independently. To enable observability later, switch to Mode 1 or 4.

### Verification

| Check | Command / action | Result (Helm) |
|-------|------------------|---------------|
| DSCI monitoring | `oc get dsci default-dsci -o jsonpath='{.spec.monitoring.managementState}'` | **Pass** — `Removed` |
| No monitoring pods | `oc get pods -n redhat-ods-monitoring` | **Pass** — no monitoring operands |
| Chart deps | Helm notes / Subscriptions | **Pass** — COO, OpenTelemetry, Tempo not requested |
| DSC | `oc get dsc default-dsc` | **Pass** — Ready |

---

## 6. Mode 4 — Custom

Helm per-pillar values: [§2.1](#21-install-with-helm-all-modes) (`values-mode4-metrics.yaml`, `values-mode4-traces.yaml`).

### Capability matrix

| Capability | Built-in | Required operators |
|------------|----------|-------------------|
| **Metrics** | MonitoringStack + OTel Collector | COO, OpenTelemetry |
| **Tracing** | Tempo + OTel Collector | OpenTelemetry, Tempo |
| **Logs** | Loki-based | CLO, Loki |
| **Dashboards** | Perses in OpenShift AI | COO |

### Examples

**Metrics + external exporter only** — operators: COO, OpenTelemetry

```yaml
spec:
  monitoring:
    managementState: Managed
    metrics:
      storage:
        size: 10Gi
        retention: 30d
      exporters:
        prometheusremotewrite/external:
          endpoint: https://prometheus.example.com/api/v1/write
```

**Traces only (built-in PV)** — operators: OpenTelemetry, Tempo

```yaml
spec:
  monitoring:
    managementState: Managed
    traces:
      storage:
        backend: pv
        size: 5Gi
```

### Verification

| Sub-case | Expected operands | Result (Helm, from scratch after Mode 3 — None) |
|----------|-------------------|-----------------------------------------------|
| Metrics only | MonitoringStack, collector; **no Tempo** | **Pass** — Prometheus/Alertmanager/Thanos + collector + Perses; no Tempo |
| Traces only | Tempo, collector; no MonitoringStack | **Pass** — Tempo + collector; no Prometheus. **Perses also deploys** with `data-science-tempo-traces` dashboard (same when switching metrics→traces — not merely leftover) |
| Dashboards | Perses when metrics and/or Tempo dashboards enabled | Perses present for metrics-only and traces-only; Cluster/Models PersesDashboard CRs not observed this run |

---

## 7. Operator reference

| Operator | Package (OperatorHub) | Namespace | Helm chart |
|----------|----------------------|-----------|------------|
| Cluster Observability Operator | `cluster-observability-operator` | `openshift-cluster-observability-operator` | `services.monitoring.dependencies.clusterObservability` |
| Red Hat build of OpenTelemetry | `opentelemetry-product` | `openshift-opentelemetry-operator` | `services.monitoring.dependencies.opentelemetry` |
| Tempo Operator | `tempo-product` | `openshift-tempo-operator` | `services.monitoring.dependencies.tempo` |
| Loki Operator | `loki-operator` | `openshift-operators-redhat` | Chart may install when monitoring deps resolve; else OperatorHub |
| OpenShift Logging (CLO) | `cluster-logging` | `openshift-logging` | **Not in chart** — OperatorHub (`stable-6.6` validated) |

**COO subscription example** (OperatorHub — CLO/Loki use the same pattern; chart installs COO/OTel/Tempo when enabled):

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
spec:
  channel: stable
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

Use the same pattern for `opentelemetry-product` and `tempo-product`. Confirm channels for your OpenShift version.

> **Note:** Dashboards are delivered via Perses CRDs managed by COO. There is no separate Perses Operator prerequisite.

---

## 8. Release notes — Observability dashboards GA

> Draft section for RHOAI 3.6 release notes.

In Red Hat OpenShift AI **3.5**, observability dashboards that were previously Technology Preview transitioned to **General Availability (GA)**. RHOAI **3.6** documentation adds explicit guidance for four observability deployment modes.

**Dashboards GA since 3.5** (from 3.5 admin guide):

| Dashboard | Access |
|-----------|--------|
| Cluster | Cluster administrators |
| Models | All users (namespace-scoped) |
| Usage | MaaS usage metrics |
| LLM Traffic | llm-d |
| LLM Performance | llm-d |
| LLM Utilization | llm-d |

**What's new in 3.6 documentation:**

- Four observability deployment modes with operator prerequisite matrices
- Per-mode Helm chart values (`services.monitoring`) and verification steps
- GA prerequisite guidance for observability platform operators
- Built-in logs require S3-compatible storage on the Monitoring CR (not via DSCI/Helm today) and privileged Pod Security on the monitoring namespace for CLO collectors

---

## Additional resources

- [rhai-on-openshift-chart — How Component Dependencies Work](https://github.com/opendatahub-io/odh-gitops/tree/main/charts/rhai-on-openshift-chart#how-component-dependencies-work) (same model for `services.monitoring`)
- [odh-gitops — rhai-on-openshift-chart](https://github.com/opendatahub-io/odh-gitops/tree/main/charts/rhai-on-openshift-chart)
- [RHOAI 3.5 — Manage observability](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/managing_openshift_ai/managing-observability_managing-rhoai)
- [Namespace-restricted metrics](NAMESPACE_RESTRICTED_METRICS.md)
- [Accelerator metrics](ACCELERATOR_METRICS.md)
- [Troubleshooting](troubleshooting.md)
