---
name: sycl-runtime-overhead
description: Use when measuring SYCL host runtime time for an Intel XPU workload, including preparation, queries, submission, and object lifetime.
---

# SYCL Runtime Overhead

Measure validated runtime functions used for preparation, queries, submission, waits and lifetime management. Submit-only logging is partial coverage.

**Inputs:** workload/phase, environment and RUN_DIR ([requirements](../SKILL.md#requirements)). **Tools:** GDB/binutils for discovery; perf for native timing; unitrace for requested activity analysis. Inclusive SYCL needs no UR/L0 timing.

## 1. Check activities when requested

Native-only requests start at step 2. For activity analysis, inspect a supplied trace or collect one after checking unitrace's version/help.

Wrap the warmed phase in a unique `torch.profiler.itt.range_push`/`range_pop` range; put measurement-only draining afterward. Run from the workload's normal directory:

```bash
"$UNITRACE" --chrome-syclrt-logging --chrome-itt-logging \
  --output-dir-path "$RUN_DIR/activity" "$PYTHON" "$RUN_DIR/activity_workload.py" \
  > "$RUN_DIR/activity.stdout.log" 2> "$RUN_DIR/activity.stderr.log"
```

Verify local flag support; retain status/original JSON. Parse with `parse_float=Decimal` and save observed name/category/phase counts within the ITT window as `activity_inventory.json`. Check actual host boundaries. A `submit` name or SYCL category does not establish full runtime coverage.

## 2. Build the native candidate list

Generate `discovery_workload.py`: initialize/warm up, save PID/native TID and `/proc/self/maps`, then raise `SIGTRAP` before controlled repetitions. Run under GDB:

```bash
gdb -q -nx -batch -ex 'set pagination off' -ex 'set confirm off' \
  -ex 'handle SIGTRAP stop nopass' -ex run -ex 'info sharedlibrary' \
  --args "$PYTHON" "$RUN_DIR/discovery_workload.py" \
  > "$RUN_DIR/discovery.log" 2>&1
```

From mappings, identify `SYCL_LIB` and actual caller libraries. Save identities and these outputs, using distinct caller labels:

```bash
sha256sum "$SYCL_LIB" > "$RUN_DIR/sycl.sha256"
readelf -nW "$SYCL_LIB" > "$RUN_DIR/sycl.notes.txt"
readelf -lW "$SYCL_LIB" > "$RUN_DIR/sycl.segments.txt"
nm -anC --defined-only "$SYCL_LIB" > "$RUN_DIR/sycl.symbols.txt"
nm -DC --defined-only "$SYCL_LIB" > "$RUN_DIR/sycl.exports.txt"
nm -DC --undefined-only "$CALLER_LIB" > "$RUN_DIR/$CALLER_LABEL.imports.txt"
readelf -rW "$CALLER_LIB" > "$RUN_DIR/$CALLER_LABEL.relocations.txt"
```

Preserve stderr/status and caller hashes/build IDs. If full symbols are absent, use exports and mark internal coverage unavailable. Build `candidates.json`:

1. Select callable definitions (`t/T/w/W`) in executable LOAD segments: SYCL functions and constructor/destructor/lifetime helpers. Keep aliases; deduplicate by library/address. Record uninspected cold/inlined paths as gaps.
2. Match callable SYCL-owned imports. Exclude RTTI, vtables/data and other libraries' functions merely taking SYCL arguments. Save unmatched callable imports; probe the broader SYCL set to catch internal/indirect calls, including those without UR children.
3. Convert ELF A to `file_offset = p_offset + A - p_vaddr`; within the matching executable mapping, `runtime_address = mapping_start + file_offset - mapping_file_offset`.

Keep names/aliases, library hash, ELF address, file offset and importing caller. Missing DWARF does not prevent address breakpoints.

## 3. Record entries and returns in GDB

Generate `validate_sycl.py` from the candidates. Add `-ex "source $RUN_DIR/validate_sycl.py" -ex continue` after `-ex run` above. Recheck hashes and relocate from the current process's mappings.

| GDB API/callback | Recorder action |
| --- | --- |
| `gdb.Breakpoint(f"*{runtime_address:#x}", internal=True)` | Retain one entry breakpoint per candidate address. |
| Entry `stop()` | Allocate ID; record PID/TID, repetition, boundary, PC, caller and stack; add to pending map; return `False`. |
| `gdb.FinishBreakpoint(gdb.newest_frame(), internal=True)` | Retain invocation ID/boundary/thread in the finish breakpoint. |
| Finish `stop()` | Record return, remove that pending ID, return `False`. |
| `out_of_scope()` or installation failure | Record unwind/failure as a gap. |

Use `gdb.selected_inferior().pid`, `gdb.selected_thread().ptid[1]`, and `gdb.newest_frame()`, `frame.older()`, `frame.pc()`, `frame.name()`. At validated x86-64 entries, read caller return PC at `$rsp`; check other ABIs. Use a repetition marker or one repetition per run. Append `sycl_gdb.jsonl`; save pending IDs/errors at exit. GDB timings are not performance measurements.

Pair by ID/thread, not callback order. GDB finish callbacks can arrive out of LIFO order; optimization can hide ancestor frames. Select outer boundaries using entry stacks, caller PCs and lifetimes together; retain uncertain candidates. Convert ambiguous caller PCs to ELF addresses and inspect a small `objdump -drC --start-address=... --stop-address=...` region. Unresolved ancestry, imports or returns remain gaps. Validate paths with and without lower-runtime children.

## 4. Time the validated boundaries

Write `boundaries.json`: selected functions/aliases, library identities, entry offsets, callers, entry/return evidence, nesting decisions and repetition counts. Generate one pair per function:

```text
p:GROUP/sycl_N_enter /absolute/libsycl.so:0xFUNCTION_ENTRY_FILE_OFFSET
r:GROUP/sycl_N_exit /absolute/libsycl.so:0xFUNCTION_ENTRY_FILE_OFFSET
```

Both use the function-entry ELF file offset; `r` records return. Give each function a distinct name and deduplicate aliases. An ABI-validated caller fetch may be added; x86-64 entry uses `caller=$stack0:x64`.

Use [shared native capture](../SKILL.md#shared-native-capture); native-SYCL-only supplies no dispatch probes. Pair full perf records by PID/TID/boundary, check recursion/stacks, then clip and union into S. Nested unitrace annotations add no extra time; retain annotations outside S separately until validated. Cross-tool comparison requires [clock verification](../SKILL.md#window-clocks).

## Example

If activity logging shows only submit, native discovery may also find a pointer query and event destructor. Include all validated intervals, even calls without UR children; exclude gaps. Derive the function set from this workload, not a fixed mm list.

## Output format

Write `04_sycl_runtime_overhead.json/.log` using the [common format](../SKILL.md#output-format):

| Field | Value |
| --- | --- |
| `measurement` | `sycl` |
| `boundary.start/end` | Selected native entry/return; describe separately validated activity boundaries |
| `details` | Exact keys: `boundary_manifest`: relative path or null; `selected_boundaries`, `activity_names`: string lists, empty when unavailable/not collected |
| `artifacts.intervals` | `intervals/sycl.jsonl`; retain entry/return sources |

Link raw perf/GDB/activity files. Explain the validated function set and gaps. For requested full-runtime coverage with submit-only data, inclusive time is null; publish valid submission time only as `observed_subset`.

## Checks

Check controlled-call counts, pending returns, identities, losses and independent unions. Preserve bypasses and timestamp excursions when comparing collected UR/L0 intervals. Balanced counts do not prove full coverage; caller-inlined work is outside native function intervals. Apply shared output checks.
