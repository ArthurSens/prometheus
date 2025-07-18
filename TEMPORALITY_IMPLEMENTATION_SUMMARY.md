# Temporality Label Implementation Summary

## Overview

This document summarizes the implementation of temporality labels in the OTLP endpoint and the corresponding PromQL engine annotations for Prometheus.

## Features Implemented

### 1. Temporality Label Addition (`__temporality__`)

Similar to how `__type__` and `__unit__` labels are added to OTLP metrics, a new `__temporality__` label is now added when both feature flags are enabled:
- `type-and-unit-labels`
- `otlp-native-delta-ingestion`

#### Label Values:
- `"delta"` - for metrics with delta temporality
- `"cumulative"` - for metrics with cumulative temporality
- No label - when temporality is unknown or not set

### 2. PromQL Engine Annotations

Extended the PromQL engine to provide helpful annotations when functions are used inappropriately with different temporalities:

#### Rate/Increase Functions:
- **Info annotation** when `rate()` is used on delta temporality metrics: *"rate() should not be used on delta temporality metrics. The rate is already calculated"*
- **Info annotation** when `increase()` is used on delta temporality metrics: *"increase() should not be used on delta temporality metrics. The rate is already calculated"*
- **No annotation** for cumulative temporality (expected usage)

#### Sum Over Time Function:
- **Info annotation** when `sum_over_time()` is used on cumulative temporality metrics: *"sum_over_time() should not be used on cumulative temporality metrics. Use rate() or increase() for rate calculations"*
- **No annotation** for delta temporality (expected usage)

## Files Modified

### Core Implementation:
1. **`storage/remote/otlptranslator/prometheusremotewrite/metrics_to_prw.go`**
   - Added `AddTemporalityLabels` field to Settings struct

2. **`storage/remote/otlptranslator/prometheusremotewrite/helper.go`**
   - Extended `addTypeAndUnitLabels()` to support temporality
   - Added `addMetadataLabels()` function that handles type, unit, and temporality labels
   - Updated signature of `addHistogramDataPoints()` to accept temporality parameter

3. **`storage/remote/otlptranslator/prometheusremotewrite/number_data_points.go`**
   - Updated `addSumNumberDataPoints()` to use temporality information

4. **`storage/remote/otlptranslator/prometheusremotewrite/histograms.go`**
   - Updated `addExponentialHistogramDataPoints()` and `addCustomBucketsHistogramDataPoints()` to use temporality

5. **`storage/remote/write_handler.go`**
   - Added `AddTemporalityLabels` field to `OTLPOptions` struct
   - Added `nativeDeltaIngestionEnabled` field to `rwExporter` struct
   - Updated configuration logic to enable temporality labels only when both feature flags are set

### PromQL Engine Annotations:
6. **`util/annotations/annotations.go`**
   - Added new error constants:
     - `RateOnDeltaTemporalityInfo`
     - `IncreaseOnDeltaTemporalityInfo`
     - `SumOverTimeOnCumulativeTemporalityInfo`
   - Added constructor functions for these annotations

7. **`promql/engine.go`**
   - Extended annotation logic for `rate()` and `increase()` functions
   - Added `enableTypeAndUnitLabels` field to `EvalNodeHelper` struct
   - Updated all `EvalNodeHelper` creation sites to pass the flag

8. **`promql/functions.go`**
   - Modified `funcSumOverTime()` to check for temporality and generate appropriate annotations

### Test Implementation:
9. **`storage/remote/write_test.go`**
   - Added comprehensive test cases for temporality labels:
     - Delta temporality test case
     - Cumulative temporality test case  
     - No temporality label test case
   - Added helper function `generateOTLPWriteRequestWithTemporality()`
   - Extended `handleOTLP()` function to support temporality options

10. **`promql/engine_test.go`**
    - Extended `TestRateAnnotations` with temporality test cases
    - Created new `TestSumOverTimeAnnotations` test suite

## Feature Flag Requirements

The temporality functionality is only enabled when **both** feature flags are set:
- `--enable-feature=type-and-unit-labels`
- `--enable-feature=otlp-native-delta-ingestion`

This ensures that temporality labels are only added when the system is configured to handle both type/unit metadata and native delta ingestion.

## Behavior Summary

| Scenario | Temporality Label | Rate/Increase Behavior | Sum Over Time Behavior |
|----------|-------------------|------------------------|------------------------|
| Delta metrics | `__temporality__=delta` | Info annotation (shouldn't use) | No annotation (OK to use) |
| Cumulative metrics | `__temporality__=cumulative` | No annotation (OK to use) | Info annotation (shouldn't use) |
| Unknown temporality | No label | Type-based annotations | No annotation |
| Feature flags disabled | No label | Existing behavior | Existing behavior |

## Testing

All functionality is thoroughly tested with:
- Unit tests for OTLP label addition (`TestOTLPWriteHandler`)
- Unit tests for PromQL annotations (`TestRateAnnotations`, `TestSumOverTimeAnnotations`)
- Test coverage for all temporality scenarios (delta, cumulative, unknown)
- Test coverage for feature flag combinations

The implementation maintains backward compatibility and only adds new functionality when the appropriate feature flags are enabled.