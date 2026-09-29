# Platform logs — work status and GC fix

## Done (this repo — opendatahub-operator)

- Added `spec.monitoring.logs` on DSCI (`MonitoringCommonSpec`), aligned with Monitoring CR shape.
- Project DSCI `logs` → `Monitoring` `default-monitoring` in `BuildModuleCR`.
- Unit tests + CRD/api-docs generate.
- **Cluster verified:** DSCI `logs` projects correctly; LokiStack becomes Ready.

## Also verified (related)

- **odh-gitops #168:** Mode 1 / Helm chart installs Loki Operator (Subscription Helm-managed, CSV Succeeded).

## Blocked — full CLF Ready

After projection + Loki Ready, ClusterLogForwarder stays not Ready:

- `ClusterRoleMissing` — collector SA not authorized for application logs.
- CRBs `…-collect-app-logs` / `…-loki-writer` are deleted every reconcile.

## Root cause

**odh-observability** applies those CRBs as `rbac.authorization.k8s.io/v1`.  
Shared GC (`odh-platform-utilities` `pkg/controller/gc`) also discovers OpenShift’s twin API `authorization.openshift.io/v1` for the **same** object. Desired-set match uses full GVK → twin looks “not desired” → delete.

Creating as `rbac…` is correct. Do **not** switch create to the OpenShift legacy API.

## What needs to be fixed

1. **odh-platform-utilities (preferred / shared):** GC must not treat OpenShift RBAC API aliases (`Role` / `RoleBinding` / `ClusterRole` / `ClusterRoleBinding` under `authorization.openshift.io`) as separate deletable types vs `rbac.authorization.k8s.io` (skip or canonicalize GVK).
2. **odh-observability (optional local harden):** canonicalize GVK in `collectGarbage` desired-set predicate until utilities is bumped.
3. **odh-gitops (after operator API lands):** Helm `services.monitoring.dsci.logs` values/wiring — **required**, depends on DSCI `logs` CRD.

## Suggested order

1. Operator PR (DSCI `logs` + projection) — this branch’s code.
2. Utilities GC fix (+ observability bump / local harden).
3. Gitops `dsci.logs` Helm wiring.
