---
name: ur-runtime-overhead
description: Use when measuring Unified Runtime host API time in an Intel XPU workload.
---

# Unified Runtime Overhead

Measure UR host API entry to return, including nested runtime calls.

**Inputs:** window, PID/TIDs and user-selected unitrace ([requirements](../SKILL.md#requirements)), or a saved capture. Inclusive UR needs no GDB/perf or other-layer capture.

## Procedure

1. Verify unitrace version/help. Wrap the warmed workload phase in uniquely named ITT ranges; preserve workload waits and drain measurement-only work afterward:

   ```bash
   "$UNITRACE" --chrome-ur-logging --chrome-itt-logging \
     --output-dir-path "$RUN_DIR/runtime" "$PYTHON" "$RUN_DIR/workload.py" \
     > "$RUN_DIR/unitrace.stdout.log" 2> "$RUN_DIR/unitrace.stderr.log"
   ```

   Check local flag/output-option support; retain status and original JSON. Add `--chrome-call-logging` for requested L0 measurement/subtraction.
2. Select actual UR host APIs, e.g. `urUSMGetMemAllocInfo`. The validated schema uses `ph="X"`, `cat="cpu_op"`, PID/TID and microsecond `ts`/`dur`. Exclude metadata, flow, ITT and GPU events; validate other schemas before use.
3. Parse with `json.loads(raw_json, parse_float=Decimal)`. Set `start_ns = Decimal(ts)*1000`, `end_ns = start_ns + Decimal(dur)*1000`; preserve original indices. Use same-file ITT windows, or [verified mapping](../SKILL.md#window-clocks) for native comparison. File order need not be chronological.
4. Filter to declared PID/TIDs, clip complete records to selected windows, then union overlaps per thread into U. Nested loader/tracing/adapter representations count once. Pass intervals to shared accounting when subtraction is requested.

## Example

A 10 us UR call containing 4 us of L0 gives 10 us inclusive and 6 us after removal. The latter requires measured L0 overlap. Across calls, union intervals and exclude gaps.

## Output format

Write `05_ur_runtime_overhead.json/.log` using the [common format](../SKILL.md#output-format):

| Field | Value |
| --- | --- |
| `measurement` | `ur` |
| `boundary.start/end` | Selected host API entry/return |
| `details` | Exact keys: `event_schema` (string), `api_counts` (list of `{name: string, calls: integer/null}` across selected windows) |
| `artifacts.intervals` | `intervals/ur.jsonl`; raw sources retain original JSON event indices |

State whether L0 was removed and which window clock was used. Missing L0 leaves valid UR inclusive time available but subtraction null. Retain UR calls without a SYCL parent; API counts validate coverage, not hotspots.

## Checks

Check a raw record/window match, duplicates/nesting, threads, invalid durations/records and expected calls. Independently recompute unions/intersections and apply shared output checks. Missing events are a gap unless zero activity is established; failed mapping blocks affected combined results.
