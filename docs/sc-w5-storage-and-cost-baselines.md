# SC-W5 Storage & Cost Baselines

> **Reference baseline:** Captures the initial contract posture for storage
> management, pruning behavior, and operational cost boundaries.

## Table of Contents

- [Purpose](#purpose)
- [Storage Namespace Canonicalization](#storage-namespace-canonicalization)
- [Pruning-by-Age Chronology](#pruning-by-age-chronology)
- [Storage Footprint Telemetry](#storage-footprint-telemetry)
- [Critical Path Cost Baseline](#critical-path-cost-baseline)
- [Regression Detection](#regression-detection)

---

## Purpose

This document defines the baseline expectations for:

| Domain | Focus Area |
|--------|------------|
| Storage | Key namespace collision prevention |
| Chronology | Deterministic pruning-by-age behavior |
| Telemetry | Storage footprint signals and monitoring |
| Cost | `calculate_sla` critical-path gas regression checks |

---

## Storage Namespace Canonicalization

### Key Management

- Contract storage keys are defined as `Symbol` constants in `apexchainx_calculator/src/lib.rs`
- Each semantic domain uses a unique key prefix to prevent collisions

### Canonicalization Rules

| Rule | Description | Enforcement |
|------|-------------|-------------|
| No duplicates | Each semantic domain maps to exactly one key | Compile-time via constants |
| No overlap | Key prefixes must be disjoint across domains | Test regression guard |
| Versioned keys | Schema changes use new keys; old keys are deprecated | Migration path required |

### Regression Guard

```rust
// Test additions should fail if they introduce key collisions
#[test]
fn test_no_storage_key_collisions() {
    // Verifies all storage keys are unique
}
```

---

## Pruning-by-Age Chronology

### Behavior Specification

| Property | Requirement |
|----------|-------------|
| Determinism | Pruning must produce identical results for identical input states |
| Ordering | Older records are removed first, regardless of insertion order |
| Completeness | All records exceeding the retention target are removed |
| Event emission | Pruning operations emit a `pruned` event with count and cutoff |

### Chronology Scenarios

| Scenario | Records | Prune Target | Expected Result |
|----------|---------|--------------|-----------------|
| Chronological insert | [1,2,3,4,5] | 3 | [3,4,5] retained |
| Reverse insert | [5,4,3,2,1] | 3 | [3,2,1] retained |
| Mixed chronology | [3,1,4,2,5] | 3 | [4,2,5] retained |

---

## Storage Footprint Telemetry

### Metrics to Track

| Metric | Description | Monitor |
|--------|-------------|---------|
| History length (pre-prune) | Total records before compaction | Dashboard |
| History length (post-prune) | Records remaining after compaction | Dashboard |
| Retained-count target | Desired maximum history size | Configuration |
| Prune cadence | Time between pruning operations | Alert if too frequent |
| Storage bytes | Actual on-chain storage consumption | Cost analysis |

### Alert Thresholds

| Condition | Severity | Action |
|-----------|----------|--------|
| History > 90% of retention target | Warning | Schedule prune |
| Prune frequency > 1/hour | Warning | Increase retention target |
| Storage growth > 10% week-over-week | Alert | Review incident volume |

---

## Critical Path Cost Baseline

### `calculate_sla` Cost Profile

| Dimension | Baseline | Regression Threshold |
|-----------|----------|---------------------|
| CPU instructions | TBD (measured) | +10% from baseline |
| Memory | TBD (measured) | +15% from baseline |
| Storage reads | 2-4 (config + state) | +1 additional read |
| Storage writes | 1 (result record) | +1 additional write |

### Testing Requirements

- Baseline tests compare behavior across repeated runs
- Regressions are detected by comparing against stored baselines
- CI pipeline flags any `calculate_sla` cost change > 10%

---

## Regression Detection

### Automated Checks

| Check | Trigger | Action |
|-------|---------|--------|
| Storage cost regression | PR to main | CI failure if >10% increase |
| CPU instruction regression | PR to main | CI warning if >10% increase |
| Key collision | Any commit | CI failure if new collision detected |
| Pruning determinism | Any commit | CI failure if pruning order changes |

### Manual Review Triggers

- Any `calculate_sla` cost change > 5%
- New storage keys added without corresponding test updates
- Changes to pruning logic or chronology behavior
