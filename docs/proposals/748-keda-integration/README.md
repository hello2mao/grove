# GREP-748: KEDA Integration

<!-- toc -->
- [GREP-748: KEDA Integration](#grep-748-keda-integration)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [User Stories](#user-stories)
      - [Story 1: PD group scaling with normalized pressure](#story-1-pd-group-scaling-with-normalized-pressure)
      - [Story 2: SLO-aware scaling with a composite latency formula](#story-2-slo-aware-scaling-with-a-composite-latency-formula)
    - [Limitations/Risks \& Mitigations](#limitationsrisks--mitigations)
      - [Optional KEDA dependency with restart constraint](#optional-keda-dependency-with-restart-constraint)
      - [Provider switch introduces transient requeue cycles](#provider-switch-introduces-transient-requeue-cycles)
      - [Scaling blocked during coherent update](#scaling-blocked-during-coherent-update)
      - [User-managed HPA coexistence](#user-managed-hpa-coexistence)
  - [Design Details](#design-details)
    - [API Changes](#api-changes)
      - [`AutoScalingProvider` enum](#autoscalingprovider-enum)
      - [Extended `AutoScalingConfig`](#extended-autoscalingconfig)
      - [New `KEDASpec` type](#new-kedaspec-type)
    - [KEDA Component Operator](#keda-component-operator)
    - [HPA / KEDA Mutual Exclusion](#hpa--keda-mutual-exclusion)
    - [Provider Switch — Two-Phase Handoff](#provider-switch--two-phase-handoff)
    - [ScaledObject Construction](#scaledobject-construction)
    - [KEDA-Injected Metadata Preservation](#keda-injected-metadata-preservation)
    - [Status Propagation](#status-propagation)
      - [Write boundary](#write-boundary)
      - [`KEDAScalingReady` condition](#kedascalingready-condition)
    - [Validation](#validation)
      - [CEL rules on `AutoScalingConfig`](#cel-rules-on-autoscalingconfig)
      - [Webhook validation (`validateKEDAConfig()`)](#webhook-validation-validatekedaconfig)
    - [Monitoring](#monitoring)
    - [Dependencies](#dependencies)
    - [Test Plan](#test-plan)
    - [Graduation Criteria](#graduation-criteria)
  - [Alternatives](#alternatives)
    - [Separate top-level KEDA field on AutoScalingConfig](#separate-top-level-keda-field-on-autoscalingconfig)
    - [Conditional KEDA operator registration](#conditional-keda-operator-registration)
    - [Runtime user-managed HPA conflict detection for KEDA](#runtime-user-managed-hpa-conflict-detection-for-keda)
<!-- /toc -->

## Summary

This GREP adds KEDA (Kubernetes Event-driven Autoscaling) as a second autoscaling backend for Grove `PodClique` and `PodCliqueScalingGroup` resources, complementing the existing HPA backend. Users select KEDA by setting `provider: KEDA` on the existing `AutoScalingConfig` and specifying event-driven triggers in a new `kedaConfig` field. Grove manages the full `ScaledObject` lifecycle and propagates KEDA scaling readiness back to `PodCliqueSet` through a new `KEDAScalingReady` status condition. 

## Motivation

Grove's existing HPA backend supports resource, custom, and external metrics through the Kubernetes autoscaling API. However, AI inference workloads are often driven by higher-level signals such as request queue depth, end-to-end latency, and prefill/decode load, which require additional metrics integration. HPA also does not natively support arbitrary cross-metric expressions, so such logic must be evaluated upstream.

### Goals

* Add a `Provider` field to `AutoScalingConfig` to select HPA or KEDA as the autoscaling backend.
* Add a `KEDASpec` type to `AutoScalingConfig` for configuring KEDA `ScaledObject` triggers, polling parameters, cooldown, scale-to-zero, fallback, and advanced HPA overrides.
* Manage the `ScaledObject` lifecycle (create, update, delete) via a new KEDA component operator, following the same patterns as the existing HPA operator.
* Propagate KEDA scaling readiness to `PodCliqueSet.Status.Conditions` via a new `KEDAScalingReady` condition.
* Support KEDA as an optional dependency: controller starts normally when the KEDA CRD is absent, surfacing a clear error only for workloads that request `Provider=KEDA`.

### Non-Goals

* Automatic KEDA installation.
* Detecting or blocking conflicts between user-managed HPAs and Grove-managed or KEDA-managed HPAs (consistent with existing `Provider=HPA` behavior).
* Detecting or blocking conflicts between user-managed `ScaledObjects` and Grove-managed `ScaledObjects` targeting the same resource (consistent with the non-detection policy for user-managed HPAs).

## Proposal

The proposal extends the existing `AutoScalingConfig` with a `Provider` enum (`HPA` | `KEDA`, defaulting to `HPA`) and a new `KEDAConfig` field for KEDA-specific parameters. `Metrics` (HPA path) and `KEDAConfig` (KEDA path) are mutually exclusive and enforced by CEL rules at the API-server level, with additional webhook validation for semantic constraints.

A new `keda` component operator manages the `ScaledObject` lifecycle using the same `computeExpected → deleteExcess → createOrUpdate → RunConcurrentlyWithSlowStart` pattern as the HPA operator. One `ScaledObject` is created per (PCS replica × PCLQ/PCSG) using the same name-generation helpers as HPA, so the resource name is deterministic and stable. The KEDA operator is always registered in `CreateOperatorRegistry` (not conditionally); CRD absence is detected dynamically inside `Sync` via `meta.IsNoMatchError`, keeping the constructor signature consistent with other component operators.

When a user switches between `Provider=HPA` and `Provider=KEDA`, a strict two-phase handoff ensures no two HPAs target the same `/scale` subresource simultaneously. Each direction uses an explicit barrier in the respective operator's `Sync` method that issues a requeue signal until the outgoing resource is fully absent from cache.

KEDA scaling status is propagated to `PodCliqueSet.Status.Conditions` through a new `KEDAScalingReady` condition, written in a serial post-sync step after all G2 concurrent operators complete, to avoid concurrent status write races.

### User Stories

#### Story 1: PD group scaling with normalized pressure

As an AI platform engineer, I operate a disaggregated inference fleet where each `PodCliqueScalingGroup` replica is a fixed-ratio prefill + decode group. The system pressure is not a single number: prefill queue depth and decode queue depth each need to be normalized against their own capacity targets, and the group count should reflect the maximum of the two so that a prefill bottleneck is never masked by an idle decode tier.

HPA can evaluate multiple metrics and use the largest replica recommendation, so this max-based policy is possible when each queue metric is already exposed with an appropriate target. The challenge is expressing the normalization and combination logic directly from the raw queue signals; with HPA, I would need those metrics exposed in target-ready form or computed upstream.

With KEDA's Prometheus triggers and `scalingModifiers`, I can define each queue's pressure relative to its capacity target and combine them with `max(prefillPressure, decodePressure)` directly in the `ScaledObject`. KEDA evaluates the formula on each polling interval, so I do not need a separate component to compute and publish the derived pressure metric.

#### Story 2: SLO-aware scaling with a composite latency formula

As a platform operator, I want an inference `PodCliqueScalingGroup` to scale based on latency headroom rather than proxy queue depth. Concretely, I want to hold P90 time-to-first-token under 200 ms and P90 time-per-output-token under 50 ms simultaneously, scaling out when either SLO is at risk and scaling in only when both have recovered.

HPA scales proportionally (current / target ratio), which does not fit threshold-based SLOs. Expressing "scale out when P90 TTFT exceeds 200 ms" requires reshaping the latency signal into a utilization ratio before HPA can act on it.

With KEDA, I use Prometheus queries over vLLM's histograms to expose P90 TTFT and P90 TPOT in seconds as named triggers `ttft` and `tpot`. I combine them into a single `scalingModifiers` formula, `max(ttft / 0.2, tpot / 0.05)`, with a target of `1` and metric type `Value`. The composite metric exceeds its target when either SLO is breached and falls below it only when both latencies are below their respective limits. KEDA supplies this metric to HPA for proportional scaling, subject to the configured scaling behavior and stabilization windows, without a separate component to compute and publish the composite metric.

### Limitations/Risks & Mitigations

#### Optional KEDA dependency with restart constraint

The `Owns(ScaledObject)` watch is registered once at `SetupWithManager` time and cannot be added dynamically. If KEDA is installed after Grove starts, ScaledObject CRUD works on every PCS reconcile, but status changes that KEDA makes to a `ScaledObject` do not re-trigger PCS reconcile until Grove is restarted. The `KEDAScalingReady` condition will only advance when another PCS event fires for a different reason. Users must restart Grove after installing KEDA to get automatic status tracking. This is a known operational constraint and is documented.

#### Provider switch introduces transient requeue cycles

Switching from `Provider=HPA` to `Provider=KEDA` (or vice versa) requires at least one extra reconcile cycle before the new resource appears. The `KEDA → HPA` direction may require multiple cycles while KEDA's controller clears its internal HPA finalizer. 

#### Scaling blocked during coherent update

Grove's `scaleguard` admission webhook rejects any `spec.replicas` write — including writes issued by KEDA through the `/scale` subresource — while a coherent update is in progress on the owning `PodCliqueSet`. KEDA scaling is therefore paused for the duration of the update and resumes automatically once the update completes. This is identical to the behavior of the existing HPA backend and is intentional: scaling mid-update could race with the rolling template change. Workloads that require continuous scaling during updates should not use the coherent update feature.

Additionally, KEDA's fallback replica count must be 0 or `>= MinAvailable` on the target component (enforced by `validateKEDAFallback`). A fallback value in `(0, MinAvailable)` would cause KEDA's fallback write to be rejected by `scaleguard`, making the fallback non-functional. This constraint matches the existing `scaleguard` rule applied to all `spec.replicas` writes.

#### User-managed HPA coexistence

Grove does not detect or block conflicts between user-managed HPAs and KEDA's internal HPA. If a user retains a self-managed HPA targeting the same resource while `Provider=KEDA` is set, KEDA will create its own internal HPA and both will compete on the same `/scale` subresource. This is consistent with the existing `Provider=HPA` behavior, where Grove also does not detect user-managed HPA conflicts. The admission webhook issues a warning (not a rejection) advising users to remove self-managed HPAs before switching to `Provider=KEDA`.

## Design Details

### API Changes

The `AutoScalingConfig` type is extended with a `Provider` field and a new `KEDAConfig` field. No structural changes are made to `PodCliqueSpec` or `PodCliqueScalingGroupConfig` — the extended `AutoScalingConfig` is a drop-in replacement.

#### `AutoScalingProvider` enum

```go
// +kubebuilder:validation:Enum=HPA;KEDA
type AutoScalingProvider string

const (
    AutoScalingProviderHPA  AutoScalingProvider = "HPA"
    AutoScalingProviderKEDA AutoScalingProvider = "KEDA"
)
```

#### Extended `AutoScalingConfig`

```go
type AutoScalingConfig struct {
    // Provider selects the autoscaling backend. Defaults to HPA.
    // +kubebuilder:default=HPA
    Provider    AutoScalingProvider            `json:"provider,omitempty"`
    MinReplicas *int32                         `json:"minReplicas,omitempty"`
    MaxReplicas  int32                         `json:"maxReplicas"`
    // Metrics is used when Provider=HPA. Mutually exclusive with KEDAConfig.
    // +optional
    Metrics []autoscalingv2.MetricSpec         `json:"metrics,omitempty"`
    // KEDAConfig is used when Provider=KEDA. Must contain at least one trigger.
    // Mutually exclusive with Metrics.
    // +optional
    KEDAConfig *KEDASpec                       `json:"kedaConfig,omitempty"`
}
```

#### New `KEDASpec` type

```go
type KEDASpec struct {
    // Triggers defines at least one KEDA ScaleTrigger. Required when Provider=KEDA.
    Triggers []kedav1alpha1.ScaleTriggers `json:"triggers"`
    // PollingInterval is the interval in seconds between trigger checks.
    // +optional
    PollingInterval *int32 `json:"pollingInterval,omitempty"`
    // CooldownPeriod is seconds to wait after the last active trigger before
    // scaling back to minReplicas.
    // +optional
    CooldownPeriod *int32 `json:"cooldownPeriod,omitempty"`
    // InitialCooldownPeriod is seconds to wait after ScaledObject creation before
    // KEDA starts evaluating triggers. Useful for model load time.
    // +optional
    InitialCooldownPeriod *int32 `json:"initialCooldownPeriod,omitempty"`
    // IdleReplicaCount is the replica count when no triggers are active.
    // +optional
    IdleReplicaCount *int32 `json:"idleReplicaCount,omitempty"`
    // Fallback defines the replica count to hold when metrics are unavailable.
    // +optional
    Fallback *kedav1alpha1.Fallback `json:"fallback,omitempty"`
    // Advanced exposes HPA behavior overrides and scalingModifiers.
    // horizontalPodAutoscalerConfig.name must not be set (controller owns it).
    // +optional
    Advanced *kedav1alpha1.AdvancedConfig `json:"advanced,omitempty"`
}
```

### KEDA Component Operator

A new `keda` component operator is registered in `CreateOperatorRegistry` at the same G2 sync group as `KindHorizontalPodAutoscaler`:

```go
reg.Register(component.KindScaledObject, keda.New(cl, scheme))
```

The constructor signature is consistent with other component operators (`cl`, `scheme` only). KEDA CRD absence is detected dynamically inside `Sync` via `meta.IsNoMatchError` on the first KEDA API call, which is surfaced as a descriptive error through `PCS.Status.LastErrors`. The operator is always registered, not conditionally.

At `SetupWithManager` time the PodCliqueSet controller calls `IsCrdAvailable` for `keda.sh/ScaledObject`:

- **CRD present** → register `Owns(&kedav1alpha1.ScaledObject{})` with `childResourcesPredicate` so any KEDA mutation re-triggers reconcile.
- **CRD absent** → skip the watch; controller starts normally.

KEDA types are added to the scheme unconditionally via `kedav1alpha1.AddToScheme(s)` in `internal/client/scheme.go`.

RBAC marker on the PodCliqueSet controller:

```go
//+kubebuilder:rbac:groups=keda.sh,resources=scaledobjects,verbs=get;list;watch;create;update;patch;delete
```

### HPA / KEDA Mutual Exclusion

The existing HPA operator's `computeExpectedHPAs` currently considers a component eligible for HPA if `ScaleConfig != nil`. This is updated to also exclude `Provider=KEDA` components, since KEDA manages its own internal HPA:

```go
if pclqTemplateSpec.Spec.ScaleConfig == nil ||
   pclqTemplateSpec.Spec.ScaleConfig.Provider == grovecorev1alpha1.AutoScalingProviderKEDA {
    continue
}
```

The same guard applies to the PCSG loop inside `computeExpectedHPAs`.

### Provider Switch — Two-Phase Handoff

Because G2 components run concurrently, a same-cycle delete + create cannot guarantee ordering when switching providers. Two explicit barriers prevent two HPAs from simultaneously targeting the same resource:

**HPA → KEDA (KEDA operator barrier):** Before creating any `ScaledObject`, the KEDA operator's `Sync` checks whether a Grove-managed HPA still exists for any of the expected targets. If yes, it returns `errContinueReconcileAndRequeue` without issuing any Create.

```go
if r.groveHPAExistsForKEDATargets(ctx, expectedTargets, pcs.Namespace) {
    return errContinueReconcileAndRequeue("waiting for Grove HPAs to be removed before creating ScaledObjects")
}
```

**KEDA → HPA (HPA operator barrier):** Before creating any Grove HPA, the HPA operator's `Sync` checks whether a `ScaledObject` exists for any of the expected targets — regardless of `DeletionTimestamp`. If any `ScaledObject` exists, it returns `errContinueReconcileAndRequeue` without issuing any Create.

```go
if r.scaledObjectExistsForHPATargets(ctx, expectedTargets, pcs.Namespace) {
    return errContinueReconcileAndRequeue("waiting for KEDA ScaledObjects to be fully removed before creating Grove HPAs")
}
```

`scaledObjectExistsForHPATargets` treats a `meta.IsNoMatchError` response (KEDA CRD absent) as equivalent to "no ScaledObjects exist" and returns `false`, allowing the barrier to clear immediately. This prevents a dead loop when the user switches back to `Provider=HPA` in a cluster where KEDA was never installed.

Each direction requires at least one extra reconcile cycle. The KEDA → HPA direction may require multiple cycles while KEDA clears its internal HPA finalizer.

### ScaledObject Construction

One `ScaledObject` is created per (PCS replica × PCLQ/PCSG), using the same name-generation helpers as HPA:

```
<pcs-name>-<replica>-<clique-name>   → targets PodClique
<pcs-name>-<replica>-<pcsg-name>     → targets PodCliqueScalingGroup
```

`ScaledObject.Spec` fields are mapped from `AutoScalingConfig` and `KEDASpec`:

```
ScaledObject.Spec.ScaleTargetRef:
  APIVersion: grovecorev1alpha1.SchemeGroupVersion
  Kind:       "PodClique" | "PodCliqueScalingGroup"
  Name:       <deterministic per-replica name>

ScaledObject.Spec:
  MinReplicaCount       ← AutoScalingConfig.MinReplicas
  MaxReplicaCount       ← AutoScalingConfig.MaxReplicas
  PollingInterval       ← KEDASpec.PollingInterval
  CooldownPeriod        ← KEDASpec.CooldownPeriod
  InitialCooldownPeriod ← KEDASpec.InitialCooldownPeriod
  IdleReplicaCount      ← KEDASpec.IdleReplicaCount
  Fallback              ← KEDASpec.Fallback
  Advanced              ← KEDASpec.Advanced
  Triggers              ← KEDASpec.Triggers (verbatim)
```

Owner reference is set to the PCS for cascade deletion. Labels follow `GetDefaultLabelsForPodCliqueSetManagedResources` plus a `component: ScaledObject` label for selector queries.

The sync flow mirrors the HPA operator exactly:

```
computeExpectedScaledObjects(pcs) → []scaledObjectInfo
deleteExcessScaledObjectTasks(...)
createOrUpdateScaledObjectTasks(...)
RunConcurrentlyWithSlowStart(ctx, logger, 1, tasks)
```

When `Provider=HPA` (no KEDA), `computeExpectedScaledObjects` returns an empty list and any existing `ScaledObjects` owned by the PCS are deleted.

### KEDA-Injected Metadata Preservation

Grove uses `k8sutils.CreateOrPatchSpec` (Get → mutate → Patch). KEDA injects its own label (`scaledobject.keda.sh/name`) and finalizer (`finalizer.keda.sh`) after `ScaledObject` creation. The mutate function preserves these before applying desired state:

```go
func (r _resource) buildResource(pcs *grovecorev1alpha1.PodCliqueSet,
    scaledObject *kedav1alpha1.ScaledObject, info scaledObjectInfo) error {

    // lo.Assign(existing, desired): desired keys overwrite existing on conflict;
    // KEDA-injected keys (e.g. scaledobject.keda.sh/name) are not in getLabels()
    // output, so they survive every patch cycle without explicit handling.
    // scaledObject.Finalizers is intentionally not reset — the patch body
    // will not include finalizers, so the server-side value is preserved.
    scaledObject.Labels = lo.Assign(scaledObject.Labels, getLabels(pcs.Name, info.objectKey.Name))
    scaledObject.Spec = buildSpec(info)
    return controllerutil.SetControllerReference(pcs, scaledObject, r.scheme)
}
```

`CreateOrPatchSpec` uses `client.MergeFrom`, which generates a JSON merge patch. Fields absent from the patch body are left untouched on the server, so the KEDA finalizer is preserved without explicit handling.

### Status Propagation

#### Write boundary

G2 components run concurrently and share the same PCS object. To avoid concurrent writes to `PCS.Status.Conditions`, the KEDA operator's `Sync` method does not write to PCS status directly. Instead, a new `mutateKEDACondition` step is added inside the existing `reconcileStatus` function (in `reconcilestatus.go`), alongside `mutateUpdateInProgressCondition` and `mutateTopologyLevelUnavailableConditions`. This reuses the existing snapshot → mutate → single `Status().Update()` pattern: all condition mutations accumulate in memory and are flushed in one atomic write at the end of `reconcileStatus`.

#### `KEDAScalingReady` condition

`mutateKEDACondition` enumerates all `ScaledObjects` owned by the PCS and aggregates their readiness:

| ScaledObject state | Condition contribution |
|---|---|
| Not yet visible in cache | `False`, reason `ScaledObjectProgressing`, message `"ScaledObject \"<name>\": not yet visible in cache"` |
| Conditions nil / uninitialized | `False`, reason `ScaledObjectProgressing`, message `"ScaledObject \"<name>\": conditions not yet available"` |
| `Ready=True` | `True` |
| `Ready=False` | `False`, reason verbatim from KEDA, message `"ScaledObject \"<name>\": <KEDA message>"` |
| `Ready=Unknown` | `False`, reason `ScaledObjectProgressing`, message `"ScaledObject \"<name>\": ScaledObject is not yet ready"` |

When all owned `ScaledObjects` report `Ready=True`, the PCS-level condition is `True`. If any are not ready, the condition is `False`; the `Reason` comes from the first failing `ScaledObject` (sorted alphabetically) and `Message` lists all failures.

When `Provider=HPA` on all components (no `ScaledObjects` owned by PCS), the `KEDAScalingReady` condition is absent from PCS status entirely. When the last `Provider=KEDA` component is removed, the condition is cleared.

### Validation

#### CEL rules on `AutoScalingConfig`

| Rule | Error |
|---|---|
| `provider == 'KEDA' && kedaConfig == null` | `kedaConfig must be set when provider is KEDA` |
| `provider == 'HPA' && kedaConfig != null` | `kedaConfig must not be set when provider is HPA` |
| `provider == 'KEDA' && size(metrics) > 0` | `metrics must be empty when provider is KEDA` |
| `kedaConfig != null && size(kedaConfig.triggers) == 0` | `at least one trigger is required` |

#### Webhook validation (`validateKEDAConfig()`)

Called from the existing autoscaling validation path on create and update, alongside the existing `validateScaleConfig`:

1. `len(Triggers) >= 1` — defence-in-depth alongside CEL.
2. `validateKEDAAdvancedFields` — `scalingModifiers` allowed; `advanced.horizontalPodAutoscalerConfig.name` forbidden (controller owns the HPA name).
3. `validateKEDAFallback` — `Fallback.Replicas` must be 0 or >= the target component's `MinAvailable`. A value in the range `(0, MinAvailable)` would cause KEDA to request a replica count that Grove's admission scale guard then rejects, making the fallback non-functional.
4. Warning (not rejection) when `Provider=KEDA` is set, advising users to remove any self-managed HPAs targeting the same resource.

### Monitoring

KEDA scaling health is observable through the new `KEDAScalingReady` condition on `PodCliqueSet.Status.Conditions`:

* `KEDAScalingReady=True` — all owned `ScaledObjects` are ready and KEDA is actively managing scaling.
* `KEDAScalingReady=False` with reason `ScaledObjectProgressing` — a `ScaledObject` is not yet ready (transient state during creation or KEDA initialization).
* `KEDAScalingReady=False` with a KEDA-provided reason — KEDA has reported a failure; the condition `Reason` and `Message` are propagated verbatim from the failing `ScaledObject`.

When the KEDA CRD is absent and `Provider=KEDA` is configured, a descriptive error is surfaced through `PCS.Status.LastErrors` by the KEDA operator's `Sync`.

### Dependencies

* KEDA installed in the cluster with `ScaledObject` CRD available (`keda.sh/v1alpha1`).
* Grove operator RBAC updated with `get;list;watch;create;update;patch;delete` permissions on `keda.sh/scaledobjects`.
* The scale subresource enabled on `PodClique` and `PodCliqueScalingGroup` CRDs (already present).

KEDA is an optional dependency. When the CRD is absent, the controller starts normally and only workloads that explicitly request `Provider=KEDA` surface an error.

### Test Plan

Unit tests (`components/keda/keda_test.go`):

* `buildExpectedScaledObject()` maps all `KEDASpec` fields correctly.
* `scaleTargetRef` points to `PodClique` for standalone PCLQ targets.
* `scaleTargetRef` points to `PodCliqueScalingGroup` for PCSG targets.
* KEDA-injected labels are preserved across patch cycles.

Integration tests (envtest):

| Scenario | Assertion |
|---|---|
| `Provider=KEDA` on PCLQ | `ScaledObject` created with correct triggers and `scaleTargetRef` |
| `Provider=KEDA` on PCSG | `ScaledObject` created, targets `PodCliqueScalingGroup` |
| Trigger update | `ScaledObject` reflects new triggers after reconcile |
| Remove `ScaleConfig` | `ScaledObject` deleted; `KEDAScalingReady` condition cleared |
| `Provider=HPA` + `kedaConfig` set | Webhook rejects with validation error |
| `Provider=KEDA` + no triggers | Webhook rejects |
| KEDA CRD absent | Controller starts; `Sync` returns clear error; no panic |
| `KEDAScalingReady` propagation | `Reason` equals KEDA `ScaledObject` reason verbatim; `Message` starts with `ScaledObject "<name>": ` |
| Switch `Provider` HPA → KEDA | Grove HPA deleted; KEDA barrier requeues until HPA gone; `ScaledObject` then created |
| Switch `Provider` KEDA → HPA | HPA barrier requeues until `ScaledObject` fully absent; Grove HPA created only once `ScaledObject` gone from cache |
| `Provider=KEDA` on PCLQ; HPA operator runs concurrently | HPA operator produces no HPA for that PCLQ |
| User-managed HPA + `Provider=KEDA` | Webhook warning issued; `ScaledObject` created; coexistence is not blocked |
| Multiple `ScaledObjects`, one failing | `KEDAScalingReady=False`; `Reason` from first failing (alphabetical); `Message` lists all failures |
| `Fallback.Replicas` in `(0, minAvailable)` | Webhook rejects with message naming the `minAvailable` value |
| KEDA installed post-Grove-start (no restart) | `ScaledObject` CRUD works; `KEDAScalingReady` does not auto-update until next PCS event; no panic |

e2e tests (kind cluster with real KEDA controller):

| Scenario | Assertion |
|---|---|
| HPA → KEDA switch | At no point do two HPAs target the same resource simultaneously |
| KEDA → HPA switch | Grove HPA does not appear until KEDA's internal HPA is gone |
| `KEDAScalingReady` transitions | Condition moves `False → True` after KEDA marks `ScaledObject` ready |

### Graduation Criteria

Alpha:

* KEDA operator supports `ScaledObject` create, update, and delete for both `PodClique` and `PodCliqueScalingGroup` targets.
* Two-phase handoff barrier prevents concurrent HPA and `ScaledObject` presence during provider switches.
* `KEDAScalingReady` condition propagated to `PodCliqueSet` status.
* Unit and integration tests cover the operator, validation, status propagation, and handoff paths.
* KEDA is an optional dependency; controller starts and HPA workloads run normally when KEDA is absent.

Beta:

* e2e tests run in CI against a KEDA-enabled kind cluster.
* Documentation covers KEDA installation prerequisites, `AutoScalingConfig` field usage, the Grove restart requirement when KEDA is installed post-start, and known limitations.
* At least one non-kind environment validates KEDA trigger evaluation against real metrics.

GA:

* The integration is used by multiple Grove workloads without known correctness issues.
* Monitoring and status conditions are sufficient for operators to diagnose KEDA initialization, trigger, and handoff failures.
* Upgrade and rollback behavior is documented.

## Alternatives

### Separate top-level KEDA field on AutoScalingConfig

KEDA configuration could be placed at the same level as `Metrics` without a `Provider` discriminator, using presence of `KEDAConfig` to imply KEDA is selected. This was rejected because an explicit `Provider` enum makes the intent unambiguous, prevents accidental simultaneous presence of both `Metrics` and `KEDAConfig`, and is forward-compatible with additional autoscaling backends.

### Conditional KEDA operator registration

The KEDA operator could be conditionally registered in `CreateOperatorRegistry` only when KEDA is available (similar to how `KindComputeDomain` is only registered when MNNVL is enabled). When KEDA is absent and a user configures `Provider=KEDA`, the reconciler would silently skip the unregistered operator, providing no user-visible error. This was rejected in favor of always registering the operator and detecting CRD absence dynamically via `meta.IsNoMatchError` inside `Sync`, which surfaces a clear error to `PCS.Status.LastErrors` and keeps the constructor signature consistent with other component operators.

### Runtime user-managed HPA conflict detection for KEDA

An earlier design added a runtime guard in the KEDA operator `Sync` that listed all HPAs targeting the same resource and blocked `ScaledObject` creation if any user-managed HPA was found. This was removed for consistency: Grove does not perform this detection for `Provider=HPA` either, and the asymmetry would be surprising. A webhook warning is sufficient to advise users, consistent with the existing behavior for all autoscaling providers.