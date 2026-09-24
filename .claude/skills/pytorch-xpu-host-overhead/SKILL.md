---
name: pytorch-xpu-host-overhead
description: Use when measuring PyTorch dispatch, SYCL, Unified Runtime, or Level Zero host overhead for a workload.
---

# PyTorch Host Overhead

Measure selected host intervals during forward, prefill, decode, or another workload phase. Report window totals and raw evidence. Host elapsed time includes waiting and descheduling; GPU time is separate.

## Measurements

| Skill | Measures |
| --- | --- |
| [Torch dispatch](torch-dispatch-overhead/SKILL.md) | Native dispatch start to backend execution entry, including preparation. |
| [SYCL](sycl-runtime-overhead/SKILL.md) | Validated runtime function entry to return. |
| [UR](ur-runtime-overhead/SKILL.md) | Unified Runtime host API entry to return. |
| [L0](l0-runtime-overhead/SKILL.md) | Level Zero host API entry to return. |

## Layers and overlap

```text
Torch dispatch
  SYCL
    UR
      L0
```

For the validated XPU/oneDNN `torch.mm(a, b)` path:

| Layer | Start | End |
| --- | --- | --- |
| Dispatch | Before tensor dispatch-key reads in `at::_ops::mm::call` | `dnnl_sycl_interop_primitive_execute` entry |
| SYCL | Selected function entry, e.g. `detail::get_pointer_type`, `queue::submit_with_event_impl` | Same invocation's return |
| UR | API entry, e.g. `urUSMGetMemAllocInfo`, `urEnqueueKernelLaunchWithArgsExp` | Same API's return |
| L0 | API entry, e.g. `zeMemGetAllocProperties`, `zeCommandListAppendLaunchKernel` | Same API's return, not GPU completion |

Queries before primitive entry may overlap dispatch; submission after it does not. Merge actual intervals, not assumed nesting. Rediscover boundaries for other builds/backends.

## Requirements

- Workload, phase, execution mode, device, inputs and correctness check.
- Existing binaries and Python; do not modify/rebuild PyTorch or runtimes, or compile benchmarks.
- **unitrace, for runtime traces:** use the user's installation. Ask if unspecified; search existing installations only when delegated or the user reports none. Record version, help and executable/collector identities.
- **RUN_DIR:** fresh directory for generated Python/GDB/shell code and evidence. No bundled scripts or old-helper dependencies.

Resolve command variables from this run and create output directories before use.

## Workflow

Open only the requested children and required subtraction dependencies. Share one capture across selected layers.

| When | Action and output |
| --- | --- |
| Every run | Record scope, commands, tool versions and binary hashes/build IDs in `manifest.json`. |
| New native measurement | Child discovers/validates boundaries; [shared capture](#shared-native-capture) records them. |
| New API measurement | Child supplies unitrace flags; combine selected flags in one invocation. |
| Saved capture | Validate recorded scope/identities; skip discovery and collection. |
| Analysis | Pair, align [clocks](#window-clocks), calculate [totals](#accounting), write [reports](#output-format). |

Retain generated code, raw files, command stdout/stderr and exit statuses.

## Shared native capture

**Dispatch and native SYCL:** validated probe definitions + workload/windows → `perf.data`, `perf.txt` and capture/cleanup logs. GDB validates boundaries; perf supplies timings.

Probe recipes: [dispatch](torch-dispatch-overhead/SKILL.md#4-collect-and-calculate), [SYCL](sycl-runtime-overhead/SKILL.md#4-time-the-validated-boundaries). Merge selected definitions into one unique group.

| Stage | Action / evidence |
| --- | --- |
| Prepare/register | Recheck identities/offsets; generate `probes.add`, `events.txt`; user registers probes and saves readback/status. |
| Ready/attach | Ordinary-user workload publishes PID/TIDs and maps, then waits. Attach perf disabled; release work only after `enable`/`ack`. |
| Record/finish | Save windows/correctness; stop and flush perf and any unitrace output. |
| Cleanup/decode | User removes only this group. Save remaining-probe, primary/cleanup statuses, perf header/text/stderr and loss diagnostics. |

Append tracefs definitions with `dd if="$RUN_DIR/probes.add" of=/sys/kernel/tracing/uprobe_events oflag=append conv=notrunc status=none`; seeking to EOF can fail. Create control/ack FIFOs and a comma-separated `EVENT_LIST` of `GROUP:event` names. Check installed perf support:

```bash
perf record --per-thread --no-buildid-cache --clockid CLOCK_MONOTONIC_RAW --delay=-1 \
  --control="fifo:$RUN_DIR/control.fifo,$RUN_DIR/ack.fifo" \
  -p "$WORKLOAD_PID" -e "$EVENT_LIST" -o "$RUN_DIR/perf.data"
perf script --ns -i "$RUN_DIR/perf.data" -F trace:pid,tid,time,event,trace \
  > "$RUN_DIR/perf.txt" 2> "$RUN_DIR/perf_script.stderr.log"
```

Strip newline/NUL from acknowledgments; retain failures/timeouts. This `--per-thread` path disables inheritance: cover new threads or report the gap; snapshots miss transient threads. Balanced counts alone do not establish coverage.

Prepare concrete privileged commands for the user. Do not execute sudo or change system settings; keep the workload under the ordinary account.

## Window clocks

**All measurements:** workload markers + collector timestamps → `windows.json` (window ID, PID/TID, clock, start/end) and `clock_checks.json` (status/evidence).

Warm up first. Keep workload-owned waits inside the window; put measurement-only draining after it. Save repetition and correctness.

- **Native only:** Python `time.clock_gettime_ns(time.CLOCK_MONOTONIC_RAW)` and perf RAW timestamps.
- **API only:** uniquely named `torch.profiler.itt.range_push`/`range_pop` windows in the same unitrace file; enable `--chrome-itt-logging`.
- **Mixed:** verify a same-process mapping before intersecting perf/unitrace intervals.

For the validated unitrace 2.4 timer, JSON timestamps are RAW time plus an epoch offset. Inspect the selected version's timer/serialization before using this mapping:

1. Find/hash the loaded collector via `/proc/self/maps`; verify exported `_ZN8UniTimer17epoch_start_time_E` with `nm -D`. Open it with `ctypes.CDLL(path, mode=os.RTLD_NOLOAD | os.RTLD_NOW)` and read `ctypes.c_uint64.in_dll(handle, symbol).value`. Never load another collector or reuse another process's offset.
2. Record the offset before/after collection. Bracket uniquely named ITT push/pop calls with RAW reads (four readings per anchor), before/between/after windows.
3. Parse JSON with `parse_float=Decimal`; compute `start_ns = Decimal(ts)*1000 - offset_ns`, then `end_ns = start_ns + Decimal(dur)*1000`.
4. Check PID/tool identity, perf clock, unchanged offset and converted ITT endpoints against their RAW brackets. Any serialization allowance needs version-specific evidence; it is not overall accuracy.

Failed mapping invalidates cross-clock attribution; retain independently valid same-clock results.

## Accounting

**Paired intervals + windows → inclusive, removed and remaining time.** Pair before clipping; use half-open intervals and same-thread unions. For dispatch/SYCL/UR/L0, name these unions D/S/U/Z.

Inclusive-only requests need no other layer. For requested subtraction, collect the layers in the overlap column:

| Layer | Inclusive | Removed overlap | Remaining |
| --- | --- | --- | --- |
| Dispatch | D | D intersect (S union U union Z) | D minus (S union U union Z) |
| SYCL | S | S intersect (U union Z) | S minus (U union Z) |
| UR | U | U intersect Z | U minus Z |
| L0 | Z | Empty | Z |

Subtract intersections, never scalar totals. This priority is accounting, not CPU instruction ownership. CUDA dispatch uses the same boundary method; CUDA runtime subtraction is outside this package.

For a valid four-layer breakdown, other/unassigned is W minus the union of D/S/U/Z. A subset's complement is only "outside selected intervals." Neither it nor dispatch's remainder is automatically Python overhead.

Union overlapping windows per thread before aggregation. Multi-thread totals are thread-time; report wall time separately. Independently check unions/subtraction, duplicates, bypasses and distinct threads.

## Output format

Each selected child writes its named `.json` and `.log`; the parent writes `summary.json` and `REPORT.md`. The `.log` is a derived report. Children define filenames, measurement IDs and `details` fields.

### Child JSON

Exact top-level keys:

| Keys | Content |
| --- | --- |
| `schema_version`, `measurement`, `role` | `1`; child ID; `requested` or `dependency_only`. |
| `scope` | Strings: `phase`, `device`, `execution_mode`, `clock`. |
| `boundary` | Strings: `start`, `end`. |
| `rows` | One result per selected window/PID/TID, including unavailable results. |
| `total` | Same result shape; `window_id="total"`, PID/TID null. `window_ns` is selected thread-time. |
| `details` | Child-specific evidence. |
| `artifacts` | RUN_DIR-relative paths: `manifest`, `windows`, `intervals`, `checks`; `raw` is a path list. Unavailable artifacts use null/empty list with an explanation. |
| `limitations` | List of coverage/interpretation limits. |

Exact result shape (illustrative values):

```json
{
  "window_id": "w0", "pid": 100, "tid": 100,
  "window_ns": "100000", "calls": 1, "unmatched_records": 0,
  "inclusive_ns": "10000", "removed_ns": null, "remaining_ns": null,
  "status": "validated_for_declared_scope",
  "attribution_status": "not_requested", "reason": null
}
```

- `*_ns`: exact decimal strings or null. Counts/PID/TID: integers or null if unknown. Window/interval IDs: strings.
- `calls`: valid logical invocations intersecting a window, before merging; totals count each once per thread. `unmatched_records`: unpaired/invalid records attributable to that scope.
- `status` covers inclusive time; `attribution_status` covers subtraction. States: `validated_for_declared_scope`, `partial_coverage`, `unsupported`, `invalid_capture`; attribution also allows `not_requested`.
- Missing/invalid requested values are null, not zero; explain them in `reason`/`limitations`. Partial coverage leaves requested inclusive time null; optionally add `observed_subset={inclusive_ns, scope}` to affected rows/total. It cannot replace the requested total; an unavailable window makes the full aggregate unavailable.
- Unrequested subtraction: both removed/remaining null. Missing dependencies: preserve valid inclusive time, leave subtraction null and mark it partial/unsupported; invalid data/clocks invalidate affected attribution. Valid results satisfy `inclusive_ns = removed_ns + remaining_ns`.

Save full paired intervals before clipping in `intervals/<measurement>.jsonl`. Required keys: `interval_id`, `pid`, `tid`, `layer`, `boundary`, `clock`, `start_ns`, `end_ns`, `sources`. Preserve every endpoint reference: e.g. `{"file":"perf.txt","line":12}` or `{"file":"runtime/trace.json","event_index":4}`.

### Child log

Use this Markdown layout, with window rows and a Total row. Display ms; null becomes `N/A` with a reason. Round only display values.

```text
Measurement: <name>
Scope: <phase, mode, device, windows/threads>
Boundary: <start> to <end>

| Window | PID/TID | Calls | Unmatched | Inclusive (ms) | Removed (ms) | Remaining (ms) | Status / attribution |
| ...    | ...     | ...   | ...       | ...            | ...          | ...            | ...                  |

Method: <tools, boundaries/APIs, calculation>
Evidence: <raw records, intervals, checks>
Limitations: <gaps and unavailable-result reasons>
```

### Parent summary

Exact `summary.json` keys:

| Key | Content |
| --- | --- |
| `schema_version`, `selected` | `1`; IDs with child outputs, including dependencies. |
| `workload_wall_ns`, `report` | Decimal string or null; `REPORT.md`. |
| `measurements` | All four IDs mapped to `{role, json, log}`. Unrequested: `role="not_requested"`, file paths null. |
| `accounting` | `{status, other_ns, reason}`. Publish other time only for a requested, valid four-layer breakdown with common scope/clock; otherwise null with its status/reason. |

`accounting.status` uses attribution states. `other_ns` aggregates thread-time; wall time requires comparable window timestamps. Summaries index child results rather than duplicating totals.

`REPORT.md` order: scope/boundaries; selected layer totals using child-report columns; per-window tables/links; input → command → raw file → derived result → check table; limitations. Include dependency-only labels and clock/capture/cleanup outcomes. Generate no unrequested child files.

### Verification

Check pairing, API classification, clocks, losses and coverage before publishing values. Check emitted keys/types, units, references, aggregation and JSON/log agreement, including missing/invalid inputs. Conservation verifies arithmetic, not coverage or uninstrumented accuracy.
