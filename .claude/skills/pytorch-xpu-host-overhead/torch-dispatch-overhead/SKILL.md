---
name: torch-dispatch-overhead
description: Use when measuring native PyTorch dispatch and preparation before backend execution for a CUDA or XPU workload.
---

# Torch Dispatch Overhead

Measure native dispatch start (`t0`) to backend execution entry (`t1`), including preparation. Discover the actual backend endpoint for the workload.

**Inputs:** workload/phase, execution mode, existing Python, RUN_DIR. **Tools:** Linux GDB with Python, binutils and perf uprobes. Use the [parent](../SKILL.md) for shared capture/accounting; inclusive dispatch needs no runtime tracer.

## 1. Find the loaded binaries

Generate `discovery_workload.py`: initialize/warm up the workload, save PID/native TID and `/proc/self/maps`, then call `signal.raise_signal(signal.SIGTRAP)`. After the marker, run identifiable repetitions and save correctness. Preserve workload semantics; run this wrapper only under GDB.

```bash
gdb -q -nx -batch -ex 'set pagination off' -ex 'set confirm off' \
  -ex 'handle SIGTRAP stop nopass' -ex run \
  -ex 'info sharedlibrary' -ex 'info proc mappings' \
  --args "$PYTHON" "$RUN_DIR/discovery_workload.py" \
  > "$RUN_DIR/discovery.log" 2>&1
```

This run stops at the initialized marker. Use saved mappings to locate PyTorch/backend libraries; `ldd` misses dynamically loaded libraries. Save identities and symbol availability. Missing DWARF alone does not require rebuilding.

## 2. Locate start and end instructions

For each mapped `LIB`, choose a unique `LABEL`:

```bash
sha256sum "$LIB" > "$RUN_DIR/$LABEL.sha256"
readelf -nW "$LIB" > "$RUN_DIR/$LABEL.notes.txt"
readelf -lW "$LIB" > "$RUN_DIR/$LABEL.segments.txt"
nm -anC "$LIB" > "$RUN_DIR/$LABEL.symbols.txt"
nm -DC --defined-only "$LIB" > "$RUN_DIR/$LABEL.exports.txt"
```

Find the operator wrapper, e.g. `rg 'at::_ops::mm::call|at::native::.*mm'` in the symbol output. Disassemble its address range:

```bash
objdump -drC --start-address="$BEGIN_VA" --stop-address="$END_VA" "$LIB" \
  > "$RUN_DIR/$LABEL.selected_disassembly.txt"
```

Follow matching source/call instructions through dispatch-key handling to backend execution. If dispatch is inlined, select the instruction before key selection and record excluded prologue/cache work. Follow backend wrappers/imports to the actual execution entry; mm may use oneDNN or BLAS. Save its callsite and expected return PC for caller filtering.

Record enclosing-function entry, dispatch start and backend entry. For ELF address A in an executable LOAD segment:

```text
file_offset = p_offset + A - p_vaddr
runtime_address = mapping_start + file_offset - mapping_file_offset
```

Use the same file's executable mapping containing that offset. `nm`/`objdump` addresses are ELF addresses; GDB addresses are relocated; uprobes use file offsets. Recompute relocation per process and offsets after binary changes.

## 3. Validate with GDB

Generate `validate_dispatch.py`. At the marker, verify hashes, relocate candidates and retain `gdb.Breakpoint(f"*{address:#x}", internal=True)` objects. Handlers record evidence and return `False`:

| Point/callback | Record/action |
| --- | --- |
| Enclosing entry | Push call ID on a per-thread stack; save input identity; create `gdb.FinishBreakpoint(gdb.newest_frame(), internal=True)` carrying that ID. |
| Dispatch start | Attach PC/instruction to the active ID; check input/key accesses against disassembly. |
| Backend entry | Require the declared caller/path; attach caller PC, backtrace and relevant arguments. |
| Finish `stop()` | Close that exact ID with a return record. |
| Finish `out_of_scope()` | Record unwind/unresolved closure. |

Use `gdb.selected_inferior().pid`, `gdb.selected_thread().ptid[1]`, and `gdb.newest_frame()` / `frame.older()` / `frame.pc()`. At x86-64 function entry, read the return address at `$rsp`; other argument/mid-function reads need ABI/disassembly checks. Append repetition, ID, boundary, PC, caller and errors to `dispatch_gdb.jsonl`.

```bash
gdb -q -nx -batch -ex 'set pagination off' -ex 'set confirm off' \
  -ex 'handle SIGTRAP stop nopass' -ex run \
  -ex "source $RUN_DIR/validate_dispatch.py" -ex continue \
  --args "$PYTHON" "$RUN_DIR/discovery_workload.py" \
  > "$RUN_DIR/validation.log" 2>&1
```

Verify inputs, caller identity and start → backend → closure per repetition; retain exceptions/open calls. Pair finishes by ID: optimized-frame callbacks may arrive out of order. GDB provides boundary evidence, not performance timings. Missing/ambiguous endpoints are unsupported; do not substitute Python brackets or ATen RecordFunction spans.

## 4. Collect and calculate

Write `boundaries.json`: library/hash/build ID, symbols, ELF addresses/file offsets, roles, allowed backend caller and validation references. Generate:

```text
p:GROUP/dispatch_start /absolute/torch_library.so:0xDISPATCH_START_FILE_OFFSET
p:GROUP/backend_entry /absolute/backend_library.so:0xBACKEND_ENTRY_FILE_OFFSET
r:GROUP/dispatch_return /absolute/torch_library.so:0xENCLOSING_FUNCTION_ENTRY_FILE_OFFSET
```

Use ELF file offsets. The return probe is installed at enclosing-function entry and only closes the invocation. At validated x86-64 backend entries, append `caller=$stack0:x64` for caller filtering; not at mid-function dispatch points.

Use [shared native capture](../SKILL.md#shared-native-capture) for registration, RAW timestamps and cleanup. Parse full records with per-PID/TID invocation stacks, retain all three source points, then clip/union valid `[t0,t1)` prefixes into D. Nesting, multiple backend entries and exceptions require a validated pairing rule; never borrow the next invocation's endpoint.

Example: start 0 us, backend entry 10 us, wrapper return 25 us gives **10 us** dispatch time.

## Output format

Write `03_torch_dispatch_overhead.json/.log` using the [common format](../SKILL.md#output-format):

| Field | Value |
| --- | --- |
| `measurement` | `torch_dispatch` |
| `boundary.start/end` | Validated dispatch instruction / backend execution entry |
| `details` | Exact keys: `backend`: string; `boundary_manifest`: relative path or null; `excluded_prefix_work`: string list |
| `artifacts.intervals` | `intervals/torch_dispatch.jsonl`; retain start/backend/closure sources |

Link raw GDB/perf evidence. Report valid prefix counts and inclusive D; shared accounting handles requested subtraction. Explain excluded wrapper work and unsupported paths.

## Checks

Check one raw pair manually; independently recompute unions. Check losses, missing closures, wrong callers and window crossings. Validating mm does not establish whole-model coverage. Apply the parent's output checks.
