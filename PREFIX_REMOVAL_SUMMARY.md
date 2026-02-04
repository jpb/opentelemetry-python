# Prefix Support Removal from PrometheusMetricReader

## Summary
The `prefix` parameter was removed from `PrometheusMetricReader` in **PR #3137**, which was merged on **January 31, 2023**.

## Details

### Pull Request
- **PR Number**: [#3137](https://github.com/open-telemetry/opentelemetry-python/pull/3137)
- **Title**: "remove ability to set a global metric prefix for prometheus exporter"
- **Author**: @gursoz
- **Merged By**: @lzchen
- **Merged Date**: January 31, 2023 at 18:19:21 UTC
- **Commit SHA**: 1d251539f22522f044fc9662c3405fac6bafb91a

### Related Issue
- **Issue Number**: [#3077](https://github.com/open-telemetry/opentelemetry-python/issues/3077)
- **Title**: "Prometheus exporter: consider removing the ability to set a global metric prefix"
- **Opened By**: @dashpole
- **Opened Date**: December 6, 2022

### Reason for Removal
The prefix parameter was removed to comply with the **OpenMetrics specification**. Specifically, the [OpenMetrics spec on target metadata](https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md#supporting-target-metadata-in-both-push-based-and-pull-based-systems) states:

> Exposers MUST NOT prefix MetricFamily names or otherwise vary MetricFamily names based on target metadata.

The specification clarifies that metric name prefixes should not be used to differentiate instances of an application being scraped. The global prefix feature violated this requirement.

### What Changed

#### Before (with prefix support):
```python
from opentelemetry.exporter.prometheus import PrometheusMetricReader

# Could specify a prefix that would be applied to all metrics
reader = PrometheusMetricReader(prefix="MyAppPrefix")
```

#### After (without prefix support):
```python
from opentelemetry.exporter.prometheus import PrometheusMetricReader

# No prefix parameter - only disable_target_info is available
reader = PrometheusMetricReader(disable_target_info=False)
```

### Implementation Changes
The PR made the following key changes:
1. Removed the `prefix` parameter from `PrometheusMetricReader.__init__()`
2. Removed the `prefix` parameter from `_CustomCollector.__init__()`
3. Removed the logic that prepended the prefix to metric names
4. Updated all tests to remove prefix usage
5. Added a CHANGELOG entry documenting the removal

### Code Differences
```python
# OLD: PrometheusMetricReader accepted prefix
def __init__(self, prefix: str = "") -> None:
    # ...
    self._collector = _CustomCollector(prefix)

# NEW: PrometheusMetricReader no longer accepts prefix
def __init__(self) -> None:
    # ...
    self._collector = _CustomCollector()
```

```python
# OLD: _CustomCollector stored and used prefix
def __init__(self, prefix: str = ""):
    self._prefix = prefix
    # ...

# In metric translation:
metric_name = ""
if self._prefix != "":
    metric_name = self._prefix + "_"
metric_name += self._sanitize(metric.name)

# NEW: _CustomCollector no longer uses prefix
def __init__(self):
    self._callback = None
    # ...

# In metric translation:
metric_name = self._sanitize(metric.name)
```

## Current Status
As of the current codebase, `PrometheusMetricReader` only accepts a `disable_target_info` parameter:
```python
def __init__(self, disable_target_info: bool = False) -> None:
```

## Documentation Issues Found
The removal of the prefix parameter was not fully reflected in the documentation:

1. **In `/exporter/opentelemetry-exporter-prometheus/src/opentelemetry/exporter/prometheus/__init__.py`**:
   - Lines 40-41 still show example usage with prefix:
     ```python
     prefix = "MyAppPrefix"
     reader = PrometheusMetricReader(prefix)
     ```

2. **In `/docs/examples/metrics/prometheus-grafana/prometheus-monitor.py`**:
   - Lines 13-14 still show usage with prefix:
     ```python
     prefix = "MyAppPrefix"
     reader = PrometheusMetricReader(prefix)
     ```

These documentation examples are outdated and should be updated to remove the prefix parameter usage.

## Timeline
- **December 6, 2022**: Issue #3077 opened by @dashpole
- **January 22, 2023**: First PR attempt (#3136) opened
- **January 23, 2023**: PR #3137 opened (superseding #3136)
- **January 31, 2023**: PR #3137 merged, prefix support officially removed

## Migration Path
Applications using the old API:
```python
reader = PrometheusMetricReader(prefix="myapp")
```

Should be updated to:
```python
reader = PrometheusMetricReader()
```

If metric name differentiation is needed, it should be handled through proper instrumentation library naming and meter scopes rather than global prefixes.
