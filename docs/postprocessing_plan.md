# Post-processing Plan for Pitch-by-Pitch CSV Data

This document outlines how to transform the raw pitch-by-pitch CSV (see provided sample) into enriched datasets suitable for downstream analytics or visualization.

## 1. Ingestion & Schema Handling

1. Load the CSV with explicit column definitions to preserve data types:
   - Use 64-bit integers for identifiers (`id`, `bIdx`, runner indices, tags) to avoid overflow.
   - Parse floating-point fields (`zoneX`, `zoneY`, spin metrics, positions) as `float32` unless higher precision is required for analytics.
   - Treat categorical codes (`pType`, `call`, `result`, handedness flags) as enumerations backed by integer columns plus lookup tables.
2. Normalize sentinel values:
   - Convert `255` runner indices to `null` and store presence separately via boolean runners (`b1`, `b2`, `b3`).
   - Treat empty strings in catcher fields (`cPlr`) as missing values.
3. Persist the raw load and a schema version so later processing can validate compatibility.

## 2. Validation Layer

1. **Structural checks:** ensure inning numbers are non-decreasing, pitch counts increment correctly per plate appearance, and required fields for specific outcomes are present (e.g., `ltSec`, `lx`, `lz` for balls in play).
2. **Domain checks:**
   - Velocity (`spdK`) within plausible bounds (e.g., 40–200 km/h).
   - `zoneX`/`zoneY` within [-3, 3] to detect mis-scaled zones.
   - Result codes consistent with umpire call (e.g., `call = 5` with `result ≠ 0` flagged for review).
3. Emit a validation report capturing row indices and descriptions for downstream review.

## 3. Enrichment & Derived Metrics

### 3.1 Game State Reconstruction

1. Track inning/half/out/runner state transitions to derive plate appearance identifiers.
2. Compute base-out states before and after each pitch.
3. Calculate run expectancy and change in win probability (requires historical lookup tables).

### 3.2 Pitch Classification Enhancements

1. Map `pType` integer codes to descriptive labels and groupings (e.g., fastballs vs. breaking balls).
2. Derive pitch movement relative to league-average movement for handedness-matched cohorts using `sH` and `sV`.
3. Compute release-to-plate timing from `pSec` deltas and integrate spin efficiency metrics if available.

### 3.3 Batter & Runner Actions

1. Identify swings (`bSwing = 1` or `call = 4`) and misses vs. contact via `bSwing` and `result`.
2. Determine batted-ball quality metrics: exit velocity (`ev`), launch angle (`ldeg`), spray angle (`sdeg`).
3. Track runner advancement timings using `sb2Sec`, `sb3Sec`, `sbhSec` and tag flags, converting bitmasks into runner IDs.

## 4. Output Products

1. **Pitch-level table:** enriched metrics with validation flags and derived fields.
2. **Plate appearance summary:** aggregated stats (pitch counts, outcomes, run value).
3. **Runner movement log:** start/end bases, advancement timings, defensive touches (`ass` flags).
4. **Metadata tables:** enumerations for pitch types, results, defensive positions.

Store outputs as Parquet for analytics and optionally as CSV for interoperability. Include accompanying JSON schema definitions to facilitate integration with BI tools.

## 5. Testing Strategy

1. Unit tests for parser normalization (e.g., sentinel handling, enumeration mapping).
2. Scenario tests derived from the provided sample covering:
   - Strikeout sequence with pitches in/out of zone.
   - Balls in play with catcher fields populated.
   - Runner advancement events (`sb2Sec`, `sb3Sec`, `sbhSec`).
3. Regression tests to ensure derived metrics remain stable as raw schema evolves.

## 6. Automation & Tooling

1. Package transformations in a modular Python project (e.g., `pydantic` models for validation, `polars` or `pandas` for processing).
2. Provide CLI entry points to run full pipelines or specific steps (e.g., `ingest`, `validate`, `enrich`).
3. Integrate with CI to run validation/tests on sample fixtures and linting (e.g., `ruff`, `pytest`).

## 7. Documentation

1. Maintain data dictionary mapping integer codes to descriptions.
2. Document known caveats (e.g., missing `ltSec` for some results, zeroed spin values for certain pitch types).
3. Supply onboarding guide for analysts detailing how to run the pipeline and interpret outputs.

This plan should enable structured development of a robust post-processing workflow while remaining adaptable as additional data fields or game scenarios emerge.
