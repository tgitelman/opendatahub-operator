# Platform logs — work status and GC fix

## Done (this repo — opendatahub-operator)

- Added `spec.monitoring.logs` on DSCI (`MonitoringCommonSpec`), aligned with Monitoring CR shape.
- Project DSCI `logs` → `Monitoring` `default-monitoring` in `BuildModuleCR`.
- Unit tests (`TestBuildModuleCR_ProjectsLogs`, `TestBuildModuleCR_LogsWithoutStorageNulled`) + CRD/api-docs generate.
- **Build / lint / unit-test:** passed on `logs-fix`.
- **Cluster verified:** DSCI `logs` projects correctly; LokiStack becomes Ready.

## Also verified (related)

- **odh-gitops #168:** Mode 1 / Helm chart installs Loki Operator (Subscription Helm-managed, CSV Succeeded).

## Pending — full CLF Ready (utilities, not our blocker)

After projection + Loki Ready, ClusterLogForwarder stays not Ready:

- `ClusterRoleMissing` — collector SA not authorized for application logs.
- CRBs `…-collect-app-logs` / `…-loki-writer` are deleted / absent under GC (not kept stably by observability reconcile).

With Monitoring logs/CLF enabled, GC removes the log-forwarder (observability) RBAC → CLF breaks. **Does not block** continuing operator/gitops work (DSCI projection, Helm `dsci.logs`); full CLF Ready waits on the utilities GC fix.

---

## Root cause — OpenShift dual API (aliases)

### API concept (GVK)

Kubernetes identifies types by **GVK** = Group / Version / Kind.

Example (standard):

- Group: `rbac.authorization.k8s.io`
- Version: `v1`
- Kind: `ClusterRoleBinding`

OpenShift also exposes the **same** Role / RoleBinding / ClusterRole / ClusterRoleBinding objects under a legacy group:

- Group: `authorization.openshift.io`
- Version: `v1`
- Kind: `ClusterRoleBinding`

For those four kinds they are **1:1 aliases**: same name, same UID, same etcd object — two API doors into one resource. Delete via either API removes the only object.

Not every kind in `authorization.openshift.io` is an alias (e.g. `RoleBindingRestriction` is OpenShift-only). GC exception must target the **four** RBAC pairs above.

### How GC hits this

1. **odh-observability** applies CLF CRBs as `rbac.authorization.k8s.io/v1` (correct).
2. Shared GC (`odh-platform-utilities` `pkg/controller/gc`, checked at **v0.3.0** / **v0.4.0**):
   - Discovers types via `ServerPreferredResources()` (`pkg/resources.ListAvailableAPIResources`).
   - Prefers one version **per group**; `rbac…` and `authorization.openshift.io` are **different groups** → **both** CRB types are collected.
   - Lists by label / ownership, then applies caller predicates.
3. Observability `collectGarbage` desired-set is keyed by **full GVK + namespace + name**. Rendered objects use rbac GVK → OpenShift twin is **not** in the set → treated as stale → **delete**.
4. That delete removes the only binding → CLF `ClusterRoleMissing`.

Creating as `rbac…` is correct. Do **not** switch create to the OpenShift legacy API.

**Not unique to logs CRBs.** Same alias issue applies to any owned Role/RoleBinding/ClusterRole/ClusterRoleBinding on this GC path. Logs CRBs are loud because CLF reports `ClusterRoleMissing`; other CRBs can churn without a Ready failure.

### Revalidation on cluster (summary)

- Monitoring `logs` + `LokiStackAvailable=True`; CLF `Authorized=False` / `ClusterRoleMissing`.
- `…-collect-app-logs` / `…-loki-writer` absent while logs enabled (poll ~45s: no stable create).
- Twin proof: same UID under `rbac…` and `authorization.openshift.io` (Logging-owned CRBs and hand-created ones).
- **Negative:** `oc delete clusterrolebinding.authorization.openshift.io <name>` → rbac CRB gone too (same object).
- Hand-created CRB with only `part-of=monitoring` and **no ownerRef** was **not** GC’d (utilities default `onlyOwned=true` requires ownerRef). Controller-applied objects that are owned + fail desired-set match are the ones GC deletes.
- Workaround for local poking: recreate CRBs **without** monitoring ownership label so GC does not select them (not a product fix).

Avoid “remove RBAC so the controller cannot read `authorization.openshift.io`” — brittle workaround; fix aliases in GC instead.

---

## What needs to be fixed

### 1. odh-platform-utilities (agreed — repo owner)

Fix in shared GC (`pkg/controller/gc`). **Agreed with utilities repo owner** to add a specific alias exception. Not a blocker for our operator/gitops work; needed for CLF Ready / end-to-end logs.

GC must not treat OpenShift RBAC API aliases as separate deletable types vs `rbac.authorization.k8s.io`.

**Options:**

- **Skip:** when listing/considering types, ignore `authorization.openshift.io` Role / RoleBinding / ClusterRole / ClusterRoleBinding if the `rbac…` twin is also in play (or always prefer rbac for GC scan when both exist).
- **Canonicalize:** before desired-set / ownership compare, map OpenShift RBAC GVK → `rbac.authorization.k8s.io` equivalent so keys match.

**Edge case (must keep working):** if a controller **only** creates/lists/deletes via `authorization.openshift.io` (and may only have delete RBAC on that group), GC must still work on that path. Do **not** blind-drop the whole OpenShift API group. Exception = **dedupe aliases** when both views exist (or when comparing against rbac desired objects) — not “never touch OpenShift RBAC.”

### 2. odh-observability (optional local harden)

Canonicalize GVK in `collectGarbage` desired-set predicate only if needed before utilities lands / is bumped — not required to unblock our side.

### 3. odh-gitops (after operator API lands)

Helm `services.monitoring.dsci.logs` values + **`values.schema.json`** (`monitoringConfig.logs`) — required; template already passthroughs non-empty `dsci.*` keys. (Chart work started on branch `feat/monitoring-dsci-logs`: values/schema/example; no dedicated field snapshot required — same as metrics/traces.)

## Suggested order

1. Operator PR (DSCI `logs` + projection) — this branch’s code (drop unrelated docs commit from PR if still present).
2. Gitops `dsci.logs` Helm wiring (independent of utilities).
3. Utilities GC fix (owner) — when ready; optional observability bump/harden.
