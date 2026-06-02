# Coverage-guided vulnerability hunting loop

This skill drives an iterative campaign combining fuzzing, source-level coverage analysis, and targeted seed engineering. It generates a goal file that enforces a strict methodology preventing the agent from moving forward without completing each verification step.

## When invoked

Ask the user (max 4 questions):

1. **What is the target binary/process?**
2. **What is the reachable attack surface from the attacker's position?** (be specific: which components are in scope, which are NOT)
3. **What fuzzing tools are available or preferred?** (default: libFuzzer + ASAN + UBSAN)
4. **Where should the work directory be?** (default: `./fuzz_campaign/`)

## What it produces

Two files: `FUZZ_GOAL.md` (the goal prompt) and `FUZZ_LOG.md` (the research log).

### 1. `FUZZ_GOAL.md`

```markdown
# Coverage-guided vulnerability hunting: <target>

## Role

You are a vulnerability researcher performing a systematic hunt for exploitable memory corruption bugs in <target>. You combine automated fuzzing with manual source code review, using coverage data to guide your search. Your goal is not to run a fuzzer and hope — it is to understand the target's code deeply enough to construct the exact inputs that reach vulnerable code paths.

## Target details

- **Binary:** <path>
- **Source:** <path>
- **Components in scope:** <list — ONLY these>
- **NOT in scope:** <explicit exclusion list>
- **Attack position:** <description>

## Definition of done

A crash that:
1. Is reproducible
2. Is confirmed by ASAN/UBSAN (not a test assertion)
3. Has an exploitable root cause (heap overflow, UAF, type confusion — NOT null deref or benign assert)
4. Has a minimized reproducer saved to disk
5. Is in an in-scope component

## Hard rules

1. **ONLY target in-scope components.**
2. **Never conclude "no bugs exist."**
3. **Log everything** to `FUZZ_LOG.md`.
4. **Each cycle must advance** — never relaunch the same fuzzer with the same seeds.
5. **A coverage plateau does NOT mean all code is covered.** It means the fuzzer's mutation strategy cannot reach new code. Your job is to figure out what code is unreached and engineer inputs that reach it.

## Anti-patterns — things you MUST NOT do

- **Do NOT say "coverage saturated" or "all code covered" based on an edge count plateau.** A plateau means the FUZZER is stuck, not that the code is covered. You must enumerate specific uncovered functions and explain each gap.
- **Do NOT launch a new fuzzer config as a substitute for coverage analysis.** Changing `-device` flags or `QEMU_FUZZ_OBJECTS` without first understanding what code is missing is cargo-cult fuzzing.
- **Do NOT claim a seed "targets" a code path without verifying coverage increased.** Run the seed, check the edge count, diff against baseline. If edges didn't increase, the seed didn't reach the target.
- **Do NOT audit code that isn't in scope.** Every minute on an out-of-scope device is wasted.
- **Do NOT confuse "instrumented PCs" with "covered PCs."** The binary has N instrumented points. The fuzzer covers a subset. You must determine which subset.

## Cycle methodology

Every cycle has 5 steps. Each step has a **gate** — a concrete artifact you must produce before proceeding. You CANNOT skip a gate.

### Step 1: Run fuzzer until plateau

Run or check the fuzzer. A plateau is: <5 new edges in the last 50K executions.

**Gate:** Log the final coverage count and execution count. Example:
```
Cycle N Step 1: cov=3902, execs=97K, plateau confirmed (0 new edges in last 50K)
```

### Step 2: Build the coverage map

This is the most critical step. You must produce a **function-level coverage table** for every in-scope source file.

**Method:**
1. Replay the entire corpus through the fuzzer (`-runs=0`) and record the total edge count
2. For each in-scope source file, extract ALL function names (using `nm`, `addr2line`, or `grep` on source)
3. For each function, determine if the fuzzer COVERS it. Methods:
   - Add a `__attribute__((noinline))` breakpoint and check if the edge count changes when the function is removed
   - Check if the function's address range falls within covered PC ranges
   - Read the function's source and trace whether any call path from a covered entry point reaches it
4. Produce a table:

```
| Function | File:Line | Covered? | Why uncovered | Reachable? |
|----------|-----------|----------|---------------|------------|
| virtqueue_ordered_fill | virtio.c:929 | NO | Requires IN_ORDER negotiated + request completion | YES |
| handle_discard | virtio-blk.c:940 | NO | Needs VIRTIO_BLK_T_DISCARD in DMA request type | YES |
| flush_queued_data_bh | serial-bus.c:325 | NO | Needs chardev backpressure (throttle) | MAYBE |
```

**Gate:** The table must list EVERY function in the in-scope files with a YES/NO coverage status. If you write "likely covered" or "probably reached", that is not acceptable — verify it.

### Step 3: Deep analysis of uncovered-but-reachable functions

For EACH function marked "NO" + "YES" (uncovered but reachable):

1. **Read the function source code.** Not the function name — the actual code. Understand what it does.
2. **Trace the call chain backwards** from the function to the nearest guest-reachable entry point:
   ```
   virtqueue_ordered_fill
     ← called by virtqueue_fill (virtio.c:1020) when VIRTIO_F_IN_ORDER is set
       ← called by virtio_blk_req_complete (virtio-blk.c:68)
         ← called by virtio_blk_rw_complete (virtio-blk.c:137) on I/O completion
           ← triggered by blk_aio_preadv completing
             ← triggered by virtio_blk_handle_request processing a valid descriptor
               ← triggered by virtqueue_pop succeeding
                 ← triggered by guest kicking the virtqueue via MMIO notify write
   ```
3. **Identify the EXACT preconditions** that must be true at EACH step of the chain:
   - What feature bits must be negotiated?
   - What device state must be set (DRIVER_OK, queue enabled)?
   - What descriptor format is required (out_sg, in_sg, lengths)?
   - What DMA data content is required (request type, sector, flags)?
4. **Determine WHY the fuzzer can't reach it:**
   - Is it a state depth problem? (needs 5+ sequential operations)
   - Is it a magic value problem? (needs specific bytes in specific positions)
   - Is it an async/timing problem? (needs completion callback to fire)
   - Is it a feature negotiation problem? (needs specific feature bits)
5. **Assess exploitability:** If this function has a bug, would it be exploitable? Read the code for: unchecked sizes, missing bounds, integer overflows, use-after-free patterns, state confusion.

**Gate:** For each uncovered-but-reachable function, produce:
- The full call chain (entry point → target function)
- The exact preconditions at each step
- The reason the fuzzer misses it
- A concrete seed specification (not "add IN_ORDER feature" but the exact bytes)

### Step 4: Craft and VERIFY seeds

For each uncovered-but-reachable function:

1. **Write the seed** as a binary file in the generic_fuzz format (or whatever format the harness uses)
2. **Run ONLY that seed** through the fuzzer and record the edge count:
   ```bash
   mkdir /tmp/single_seed_test && cp seed /tmp/single_seed_test/
   $FUZZER -runs=1 /tmp/single_seed_test/ 2>&1 | grep 'cov:'
   ```
3. **Compare against baseline** (corpus-only coverage):
   - If edges INCREASED: the seed reached new code. Log which functions are now covered.
   - If edges DID NOT increase: the seed failed. Analyze why: wrong byte offsets? Missing precondition? Go back to Step 3 and revise.
4. **Iterate on failed seeds.** Do not move to Step 5 until at least one seed demonstrably increases coverage into a previously-uncovered function.

**Gate:** For each seed, log:
```
Seed: inorder_blk_completion
Target function: virtqueue_ordered_fill
Baseline coverage: 3902 edges
Seed-only coverage: 3902 edges → FAILED (no new edges)
Analysis: Feature negotiation succeeds but descriptor ring at 0x200000 is all zeros,
          virtqueue_pop returns NULL because avail ring idx=0. Need to set up avail ring.
Revised seed: [description of fix]
Revised coverage: 3948 edges → SUCCESS (+46 edges including virtqueue_ordered_fill)
```

### Step 5: Relaunch with verified seeds

Add ONLY the seeds that passed verification in Step 4. Relaunch the fuzzer. Return to Step 1.

**Gate:** Log the new baseline coverage after adding seeds. It MUST be higher than the previous cycle's plateau. If not, the seeds didn't work and you must go back to Step 4.

## Mandatory research steps (run once, early in the campaign)

These are one-time research activities. Do them in Cycle 1 or 2, not later.

### N-day search

1. **Clone the target's upstream repository** (full clone, not shallow)
2. **Diff ALL in-scope source files** between the target version and current master
3. **For each security-relevant change** (fix, bounds check, validation added):
   - Record the commit hash, date, author, and description
   - Check if the bug exists in the target version
   - If YES: this is a confirmed n-day. Add it to the n-day inventory with full code snippets.
4. **Search for oss-fuzz findings:** `git log --grep='oss-fuzz' -- <in-scope-files>`
5. **Search for CVE fixes:** `git log --grep='CVE-' -- <in-scope-files>`
6. **Search for all security-pattern keywords:** `git log --grep='heap-buffer-overflow\|use-after-free\|out-of-bound\|double.free\|integer.overflow' -- <in-scope-files>`

### Feature/attack surface enumeration

1. **From a live instance** (if available), enumerate the exact device configuration: PCI devices, feature bits, queue counts
2. **Verify the machine type** — do not assume (e.g., q35 vs virt)
3. **Map every guest-writable interface:** MMIO registers, PCI config space, virtqueue descriptors, DMA data
4. **For each interface, identify the handler function** in the source code

## Key files

- `FUZZ_GOAL.md` — this file
- `FUZZ_LOG.md` — research log
- `crashes/` — crash artifacts
- `corpus*/` — fuzzer corpora
- `coverage/` — coverage reports and function tables
- `seeds/` — crafted seeds with verification results
```

### 2. Output the `/goal` command

After writing the files, output:

```
/goal Read and follow <workdir>/FUZZ_GOAL.md. Log all work to <workdir>/FUZZ_LOG.md. Do not stop until the Definition of Done is met. Each cycle: run fuzzer → build function-level coverage table → deep-analyze uncovered-but-reachable functions → craft and VERIFY seeds → relaunch. Never conclude "no bugs exist." Never say "coverage saturated" without enumerating every uncovered function.
```

Also create `FUZZ_LOG.md` with:

```markdown
# Coverage-guided vulnerability hunting log: <target>

**Target:** <target>
**Start date:** <today>
**Attack surface:** <from user input>
**Tools:** <from user input>
**Method:** Fuzz → function-level coverage table → deep call-chain analysis → verified seed crafting → relaunch

---

## Cycle log
```

## Adaptation rules

- If the target is QEMU, include the generic_fuzz seed format specification with virtio DMA patterns
- If the target is a PostgreSQL extension, include SQL seed patterns
- Always include the anti-pattern warnings — they are the most important part
- Always include the gate requirements — they prevent the agent from skipping steps
- Always include the seed verification protocol — untested seeds are worthless
