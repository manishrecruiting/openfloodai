# ADR-0002: Multi-Source Risk Fusion

## Status

Proposed

## Context

OpenFloodAI currently relies on a single signal -- visual water detection
from camera frames -- to assess flood risk. A single camera can be
obscured by fog, darkness, or debris, and visual detection alone cannot
distinguish rising river levels from a rain-covered lens.

Real-world flood monitoring requires corroboration from multiple
independent data sources. Nepal's flood-prone river basins in particular
benefit from combining camera signals with upstream gauge data (DHM),
precipitation forecasts, and seismic activity (earthquake-triggered
landslide dams are a major flood cause in the Himalayan region).

## Decision

We propose a multi-source risk fusion engine that combines the existing
visual signal with external data sources to produce a corroborated risk
assessment.

### Architecture

```
Camera frames ──> Water detection ──> Visual signal
                                          │
External APIs ──> Data adapters ──> Source signals
                                          │
                                    ┌─────▼─────┐
                                    │  Temporal  │
                                    │  Window    │
                                    │ (sliding)  │
                                    └─────┬─────┘
                                          │
                                    ┌─────▼─────┐
                                    │   Risk     │
                                    │  Fusion    │
                                    └─────┬─────┘
                                          │
                                    Risk state output
```

### Temporal aggregation

A sliding window (default 10 minutes) smooths instantaneous readings
and prevents single-frame false positives from triggering alerts:

- Window tracks water ratio readings over time
- Trend detection: RISING, FALLING, STABLE, VOLATILE
- Statistical summary: mean, max, min, standard deviation
- Configurable thresholds for WATCH and WARNING_CANDIDATE states

### Risk fusion algorithm

Each data source contributes a 0.0-1.0 risk factor:

| Source | Weight | Signal |
|--------|--------|--------|
| Visual (camera) | 0.35 | Water coverage ratio |
| USGS Water | 0.15 | Gage height vs flood stage |
| DHM Nepal | 0.15 | Water level vs danger threshold |
| Precipitation | 0.10 | Recent + forecast rainfall |
| NWS Alerts | 0.10 | Active flood warnings |
| USGS Earthquake | 0.05 | Recent seismic activity |
| NASA EONET | 0.05 | Active flood/storm events |
| ReliefWeb | 0.05 | Humanitarian flood reports |

Combined risk = weighted sum of available source factors, normalized
by the sum of available weights (graceful degradation when sources
are offline).

### Risk states

| State | Meaning |
|-------|---------|
| NORMAL | Combined risk below watch threshold |
| WATCH | Elevated risk, monitoring intensified |
| WARNING_CANDIDATE | High risk, warrants operator attention |
| UNKNOWN | Insufficient data to assess |

### Graceful degradation

- If only the camera is available, the system behaves like the
  current single-source pipeline
- Each unavailable source is logged but does not prevent assessment
- Minimum confidence threshold: at least one source must be available
- Data sources are refreshed on a configurable interval (default 5min)
  to avoid overwhelming external APIs

### Proposed modules

```
src/openfloodai/risk_engine/
    temporal.py        # TemporalWindow, trend detection
    multi_source.py    # Risk fusion, source weight config

src/openfloodai/vision/
    water_detection.py # HSV water coverage detection
    camera_health.py   # Frame quality assessment

src/openfloodai/ingestion/
    stream.py          # RTSP/MJPEG camera stream capture

src/openfloodai/edge/
    monitor.py         # Continuous monitoring loop
    multi_site.py      # Concurrent multi-site threading
```

## Consequences

- Higher confidence in risk assessments through source corroboration
- Resilience to single-source failures (camera obstruction, API outage)
- Nepal-specific data (DHM) given equal weight to global sources
- Additional network calls for data source polling (bounded by interval)
- More complex risk logic to test and validate
- External API availability is best-effort, not guaranteed

## Open questions

- Should source weights be configurable per deployment site?
- How should the system handle conflicting signals (high visual but
  no upstream data support)?
- Should we add a learning component that adjusts weights based on
  observed accuracy?
- What validation dataset should we use to tune thresholds?

## Testing plan

Before implementation, we propose:

1. Unit tests for temporal window statistics and trend detection
2. Unit tests for risk fusion with various source combinations
3. Property-based tests for graceful degradation (random source subsets)
4. Integration tests with recorded camera footage and historical API data
5. Validation against known flood events in Nepal (Koshi 2024, etc.)
