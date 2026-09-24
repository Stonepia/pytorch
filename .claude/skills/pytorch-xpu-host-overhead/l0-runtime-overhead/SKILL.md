---
name: l0-runtime-overhead
description: Use when measuring Level Zero host API time for an Intel XPU workload.
---

# Level Zero Runtime Overhead

Measure L0 host API entry to return: argument setup, submission, queries, events and waits. A UR Level Zero adapter function belongs to UR; L0 API return is not GPU completion.

**Inputs:** backend, window/PID/TIDs and user-selected unitrace ([requirements](../SKILL.md#requirements)), or a saved capture. L0-only needs no GDB/perf or other-layer capture.

## Procedure

1. Check unitrace version/help and verify Level Zero activity; loading its adapter is insufficient. Bracket the warmed phase with unique ITT ranges:

   ```bash
   "$UNITRACE" --chrome-call-logging --chrome-itt-logging \
     --output-dir-path "$RUN_DIR/runtime" "$PYTHON" "$RUN_DIR/workload.py" \
     > "$RUN_DIR/unitrace.stdout.log" 2> "$RUN_DIR/unitrace.stderr.log"
   ```

   Verify local flags/output options; retain status/original JSON. This flag also logs OpenCL: classify actual API names.
2. Select valid `ze`, `zet`, `zel`, `zex` host APIs using collector/header evidence. Exclude UR adapter functions, metadata, flow and GPU records. The validated schema uses `ph="X"`, `cat="cpu_op"`, PID/TID, `ts` and `dur`.
3. Parse microsecond timestamps/durations with Decimal and convert to ns without float. Use same-file ITT windows; native comparison requires [clock mapping](../SKILL.md#window-clocks). Validate other schemas before pairing.
4. Preserve raw indices; filter to declared PID/TIDs, clip complete records to selected windows, then union same-thread overlaps into Z. For requested subtraction, pass Z to shared accounting. L0's own remainder equals Z.

Keep workload-owned waits inside the window; place measurement-only draining afterward.

## Example

A schematic mm path:

| Phase | Example L0 work |
| --- | --- |
| Before backend entry | `zeMemGetAllocProperties` inside a SYCL/UR query |
| After backend entry | Argument setup and `zeCommandListAppendLaunchKernel` inside submission |

Two L0 calls of 2 us and 3 us contribute 5 us, excluding gaps. Only actual overlap before backend entry can be removed from dispatch.

## Output format

Write `06_l0_runtime_overhead.json/.log` using the [common format](../SKILL.md#output-format):

| Field | Value |
| --- | --- |
| `measurement` | `l0` |
| `boundary.start/end` | Verified L0 host API entry/return |
| `details` | Exact keys: `backend` (`level_zero` when verified, otherwise null), `event_schema` (string), `api_counts` (list of `{name: string, calls: integer/null}` across selected windows) |
| `artifacts.intervals` | `intervals/l0.jsonl`; retain raw JSON event indices |

For a requested remainder, removed time is zero and remaining time equals inclusive Z; otherwise both are null with attribution `not_requested`. Label dependency-only collection. Explain API selection and interval unioning; add a relative timeline when useful.

## Checks

Check API classification, timestamp/window conversion, duplicates, separate threads and independent unions. Apply shared output checks. Preserve L0 outside UR; explain serialization allowance separately without trimming. Missing/unsupported coverage is null, not measured zero.
