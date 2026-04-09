---
name: vgpr-pressure-analysis
description: >
  Analyze VGPR register pressure in FlyDSL GPU kernels by combining ISA metadata
  (spill count, scratch size) with source-level liveness analysis. Identifies which
  variable groups cause peak VGPR pressure, locates the spill root cause, and suggests
  optimization directions. Works with ISA files from /dump-ir or saved ISA snapshots.
  Usage: /vgpr-pressure <isa_file> [kernel_source]
tools: Read,Bash,Grep,Glob
---

# VGPR Pressure Analysis

Analyze VGPR register pressure in a FlyDSL GPU kernel by combining ISA metadata
with source-level variable liveness analysis. Identify the root cause of VGPR spill
and produce actionable optimization suggestions.

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `<ISA_FILE>` | Yes | Path to the final ISA assembly file (e.g., `15_final_isa.s` from `/dump-ir`) |
| `[KERNEL_SOURCE]` | No | Path to the kernel Python source file. If omitted, attempts auto-detection. |

If no ISA file is provided, ask the user. If no kernel source is provided, try to
locate it from the dump directory structure or ask the user.

---

## Step 1: Extract ISA Hardware Metrics

### 1.1 Read Metadata

From the ISA file's `.amdgpu_metadata` YAML section, extract:

```bash
grep -E '\.(vgpr_count|vgpr_spill_count|agpr_count|sgpr_count|sgpr_spill_count|private_segment_fixed_size):' <ISA_FILE>
grep 'amdhsa.target:' <ISA_FILE>
```

Key fields:

| Field | What it means |
|-------|--------------|
| `.vgpr_count` | Allocated VGPRs (max 256 on gfx950/gfx942) |
| `.vgpr_spill_count` | Number of VGPR values spilled to scratch memory |
| `.agpr_count` | AccVGPRs used (0 on gfx950 for non-AGPR kernels) |
| `.sgpr_count` | Allocated SGPRs |
| `.private_segment_fixed_size` | Scratch memory bytes per thread |
| `amdhsa.target` | GPU architecture (gfx942, gfx950, etc.) |

**Quick diagnosis**:
- `vgpr_spill_count == 0` → No spill problem. Report "no VGPR pressure issue" and skip to summary.
- `vgpr_count == 256` + `vgpr_spill_count > 0` → Maxed out, spilling. Continue analysis.
- `vgpr_spill_count > 100` → Severe spill. High priority optimization target.

### 1.2 Hot Loop Instruction Statistics

Identify the hot loop (main `scf.for` body) in the ISA. The hot loop is typically
the largest basic block between two branch labels (`s_cbranch_*` targeting the same label):

```bash
# Find hot loop boundaries
grep -n 'LBB0_' <ISA_FILE> | head -10
# Example: .LBB0_2 (line 450) to s_cbranch_vccnz .LBB0_2 (line 973)
```

Then count instruction types within the hot loop:

```bash
# Replace START and END with actual line numbers
sed -n 'START,ENDp' <ISA_FILE> | grep -c 'scratch_load'
sed -n 'START,ENDp' <ISA_FILE> | grep -c 'scratch_store'
sed -n 'START,ENDp' <ISA_FILE> | grep -c 'buffer_load'
sed -n 'START,ENDp' <ISA_FILE> | grep -c 'v_mfma'
sed -n 'START,ENDp' <ISA_FILE> | grep -c 'v_cndmask'
sed -n 'START,ENDp' <ISA_FILE> | grep -c 's_nop'
```

Present as:

| Hot Loop Metric | Count | Note |
|----------------|-------|------|
| Total lines | | |
| scratch_load | | spill reads |
| scratch_store | | spill writes |
| buffer_load | | VMEM data loads |
| v_mfma | | MFMA compute |
| v_cndmask | | conditional select |
| s_nop | | pipeline nops |

Calculate: `scratch_ops_ratio = (scratch_load + scratch_store) / total_vmem_ops`
If > 30%, scratch is a major bottleneck.

---

## Step 2: Source Code Variable Grouping

Read the kernel source file and identify all VGPR-consuming variables. Organize
them into standard categories.

### 2.1 Extract Tile Constants

Search for the key constants that determine variable counts:

```python
# Search for these assignments in the kernel source:
m_repeat = tile_m // 16           # M-dimension repeat count
num_acc_n = n_per_wave // 16      # N-dimension accumulator count
k_unroll = tile_k_bytes // 64     # K-dimension unroll factor (K64 micro-steps)
sb_per_tile = tile_k // scale_block_k  # Scale blocks per tile
ku_per_sb = scale_block_k // 64   # K64 steps per scale block
num_x_loads = bytes_per_thread_x // x_load_bytes  # X tile load count
```

Use Grep to find actual values:

```
Grep(pattern="m_repeat|num_acc_n|k_unroll|sb_per_tile|num_x_loads", path=<KERNEL_SOURCE>)
```

### 2.2 Variable Categories

Scan the source code for each category:

#### Category A: Loop-Carried State
Search for `range(..., init=...)` and `yield` to identify loop-carried values.

```
Grep(pattern="init_state|init=|yield ", path=<KERNEL_SOURCE>)
Grep(pattern="_pack_loop_state|_unpack_loop_state", path=<KERNEL_SOURCE>)
```

Typical components:
- `acc_gate` / `acc_up`: m_repeat × num_acc_n values, each `vec4_f32` → 4 VGPRs
- `b_gate_flat` / `b_up_flat`: k_unroll × 2 × num_acc_n values, each `i64` → 2 VGPRs
- `a0_prefetch`: 2 values, each `i64` → 2 VGPRs

#### Category B: Scale Values
```
Grep(pattern="load_scales|combined_gate|combined_up|s_a_vecs|s_w_gate|s_w_up", path=<KERNEL_SOURCE>)
```

Look for:
- `combined_gate`/`combined_up`: pre-multiplied scales, m_repeat × num_acc_n × 4 × 2 scalars
- Or vec4 combined: m_repeat × num_acc_n × 2 vec4_f32 values

#### Category C: X Tile Regs
```
Grep(pattern="load_x_tile|x_regs|num_x_loads", path=<KERNEL_SOURCE>)
```

- `x_regs`: num_x_loads × vec4_i32 → 4 VGPRs each

#### Category D: B Tile Regs
```
Grep(pattern="load_b_tile|b_gate_tile|b_up_tile|b_tile", path=<KERNEL_SOURCE>)
```

- Per load: k_unroll × 2 × num_acc_n values, each `i64` → 2 VGPRs

#### Category E: MFMA Block Accumulators
```
Grep(pattern="block_gate_accs|block_up_accs|block_accs.*acc_init", path=<KERNEL_SOURCE>)
```

- `block_gate_accs`/`block_up_accs`: m_repeat × num_acc_n × vec4_f32 each

#### Category F: MFMA Temporaries
```
Grep(pattern="_pack128|a128|bg128|bu128|vec8_i32|vec4_i64", path=<KERNEL_SOURCE>)
```

- `a128`: 1 × vec8_i32 = 8 VGPRs (per mi iteration, transient)
- `bg128`/`bu128`: 1 × vec8_i32 each = 8 VGPRs (per ni iteration, transient)
- Pre-pack `a0,a1,a2,a3`: 4 × i64 = 8 VGPRs (transient before pack)

#### Category G: LDS Load Temps
```
Grep(pattern="lds_load_packs|a0_prefetch", path=<KERNEL_SOURCE>)
```

---

## Step 3: VGPR Cost Model

Calculate VGPR cost for each variable using the type cost table:

| MLIR / FlyDSL Type | VGPRs | Common Usage |
|---------------------|-------|--------------|
| `vec4_f32` | 4 | Accumulator, scale FMA result |
| `i64` | 2 | B tile pack, a0_prefetch, LDS load pack |
| `f32` / `i32` | 1 | Scale scalar, index |
| `vec4_i32` | 4 | X tile load (buffer_load_dwordx4) |
| `vec8_i32` | 8 | gfx950 MFMA 128-bit packed input |
| `vec4_i64` | 8 | gfx950 _pack128 intermediate |
| `T.index` | 1 | Address calculation (lowered to i32/i64) |

**Formula**: `VGPRs(category) = count(variables) × cost(type)`

Build a summary table:

| Category | Variable | Count | Type | VGPRs | Reducible? |
|----------|----------|-------|------|-------|-----------|
| A. Loop acc | acc_gate | m_repeat × num_acc_n | vec4_f32 | ... | No (output) |
| A. Loop B tile | b_gate_flat | k_unroll × 2 × num_acc_n | i64 | ... | Yes (lifecycle) |
| A. Loop a0_pf | a0_prefetch | 2 | i64 | ... | Yes (remove) |
| B. Scales | combined | m_repeat × num_acc_n × 4 × 2 | f32 | ... | Yes (lazy) |
| C. X tile | x_regs | num_x_loads | vec4_i32 | ... | No (min load) |
| D. B tile cur | b_gate_cur | k_unroll × 2 × num_acc_n | i64 | ... | Yes (per-ku) |
| E. Block accs | block_g/u_accs | m_repeat × num_acc_n × 2 | vec4_f32 | ... | Yes (reuse) |
| F. MFMA temps | a128/bg128/bu128 | 3 | vec8_i32 | ... | No (transient) |
| G. LDS temps | a0-a3 pre-pack | 4 | i64 | ... | No (transient) |

---

## Step 4: Liveness Analysis

This is the core analysis step. For each **Critical Point** in the code, determine
which variable groups are simultaneously alive and sum their VGPR cost.

### Critical Points

Analyze these locations in the kernel code (in order of typical peak pressure):

#### CP1: Stage 0 Compute (while loads for Stage 1 are queued)

At the point where `compute_tile_bs_s1()` executes in Stage 0, these are all alive:

| Alive Variable Group | Reason | VGPRs |
|---------------------|--------|-------|
| Loop-carried state (full) | Will become yield output | ... |
| B tile for next stage | Just loaded, waiting for stage 1 | ... |
| X regs for next stage | Just loaded, waiting for LDS store | ... |
| Combined scales | Needed by compute | ... |
| Block accs | Inside compute_tile | ... |
| MFMA temps | Inside MFMA loop | ... |
| **TOTAL** | | **???** |

#### CP2: Yield Point

At `yield`, all loop-carried values must be simultaneously alive (SSA hard constraint):

| Alive Variable Group | Reason | VGPRs |
|---------------------|--------|-------|
| acc_gate + acc_up | Output accumulators | ... |
| B tile next (if loop-carried) | For next iteration | ... |
| a0_prefetch (if loop-carried) | For next iteration | ... |
| **TOTAL** | | **???** |

Plus any values still live from the final compute before yield.

#### CP3: Inside MFMA + Scale FMA

During the innermost compute loop when both MFMA and scale FMA execute:

| Alive Variable Group | VGPRs |
|---------------------|-------|
| current_gate + current_up | ... |
| block_gate_accs + block_up_accs | ... |
| combined scales (current sb) | ... |
| a128 + bg128 + bu128 | ... |
| B tile (remaining ku packs) | ... |
| **TOTAL** | **???** |

### Finding the Peak

The **peak point** is whichever CP has the highest total. Report:

```
Peak VGPR pressure: CP1 (Stage 0 Compute) = XXX VGPRs
Hardware limit: 256 VGPRs
Overflow: XXX - 256 = YY VGPRs → causes ZZZ spills
```

---

## Step 5: Diagnosis and Optimization Suggestions

Based on the peak analysis, identify which variable groups contribute most to the
overflow and suggest specific optimizations.

### Optimization Catalog

#### OPT-A: Remove Non-Acc Values from Loop State (High Impact)

**Condition**: Loop-carried state includes B tiles, prefetch, or other non-accumulator values.

**Method**: Create a `do_one_stage()` function that loads B tiles internally.
Move `load_b_tile()` calls inside the stage function instead of carrying B tile
values across iterations via `yield`.

**VGPR savings**: `count(non-acc loop values) × type_cost`
(e.g., 16 × i64 + 2 × i64 = 36 VGPRs for B tiles + a0_prefetch)

**Trade-off**: B tiles loaded twice per iteration (once per stage) instead of pre-loaded.
Net VMEM effect is usually positive because scratch ops saved >> extra B loads.

**Pattern to find**: `_pack_loop_state`, `_flat_btile_half`, B tile values in `yield`.

#### OPT-B: Reuse acc_init for Block Accumulators (High Impact)

**Condition**: Code has `block_*_accs = [acc_init] * N` — separate accumulator arrays
pre-allocated for each scale block, then accumulated via `_do_scale_fma` after all
MFMA ops in the block.

**Method**: Use `acc_init` (zero) directly as the MFMA accumulator operand. After
each MFMA, immediately `fma(mfma_result, scale, current_acc)` — no need for a
separate block_accs array.

**VGPR savings**: `2 × m_repeat × num_acc_n × 4` (e.g., 2 × 4 × 2 × 4 = 64 VGPRs)

**Trade-off**: None significant. The inline FMA is the same amount of compute, just
reordered.

**Pattern to find**: `block_gate_accs = [acc_init]`, `block_up_accs = [acc_init]`,
`_do_scale_fma`.

#### OPT-C: Lazy Scale Computation (Medium Impact)

**Condition**: `combined_gate`/`combined_up` pre-multiplied scales occupy > 30 VGPRs.
All `s_a × s_w` products are computed upfront and stored.

**Method**: Store only the raw scale values (`s_a_vecs` and `s_w_vals`) and compute
the product on-the-fly inside the MFMA loop when needed.

**VGPR savings**: `combined_vgprs - (s_a_vgprs + s_w_vgprs)`
(e.g., 64 - 20 = 44 VGPRs for 4×2 tile with 4-element rows)

**Trade-off**: Extra multiply instructions in the hot loop. Can be hidden behind
MFMA's ~64-cycle latency.

**Pattern to find**: `combined_gate = []`, `combined_up = []`, `s_a_vecs[mi][ii] * s_w_gate_vals[ni]`.

#### OPT-D: Per-KU B Tile Loading (Low Impact)

**Condition**: `load_b_tile()` loads all k_unroll packs at once. k_unroll > 1.

**Method**: Load B tile packs one ku at a time inside the MFMA loop. Release
previous ku's packs before loading the next.

**VGPR savings**: `(k_unroll - 1) × 2 × num_acc_n × 2` (e.g., 16 VGPRs for k_unroll=2)

**Trade-off**: Reduced overlap between B tile loads and MFMA compute.

#### OPT-E: Remove Cross-Stage LDS Prefetch (Low Impact)

**Condition**: `a0_prefetch` is loop-carried state.

**Method**: Remove `a0_prefetch` parameter from `compute_tile_bs_s1`. Let the compute
function load its own first A-pack from LDS without prefetching.

**VGPR savings**: `2 × 2 = 4 VGPRs`

**Trade-off**: Slightly longer latency for first LDS read in each stage. Usually
negligible.

### Generating Suggestions

For each optimization, check if the **pattern to find** exists in the source code.
If it does, include the optimization in the report, sorted by VGPR savings.

Mark optimizations as:
- **[HIGH]**: Saves ≥ 30 VGPRs or addresses the dominant pressure source
- **[MEDIUM]**: Saves 10-30 VGPRs
- **[LOW]**: Saves < 10 VGPRs or is a reserve optimization for future tile size changes

Also mark **non-reducible** categories explicitly:
- Accumulator output (acc_gate/acc_up) — GEMM algorithm requires these
- MFMA operands (a128/bg128/bu128) — hardware instruction format
- X tile minimum load granularity — buffer_load_dwordx4 is the smallest efficient load

---

## Step 6: Report Output

Present the complete analysis as a structured report:

```
# VGPR Pressure Analysis Report

## Kernel: <kernel_name>
## Target: <gpu_arch>
## Source: <kernel_source_path>
## ISA: <isa_file_path>

## 1. ISA Metrics
| Metric | Value | Status |
|--------|-------|--------|
| vgpr_count | | ⚠️/✅ |
| vgpr_spill_count | | ❌/✅ |
| agpr_count | | |
| sgpr_count | | |
| scratch_bytes | | |

### Hot Loop Instruction Distribution
| Instruction Type | Count | % of VMEM |
|-----------------|-------|-----------|
| scratch_load | | |
| scratch_store | | |
| buffer_load | | |
| v_mfma | | |
| Total VMEM ops | | |

## 2. Variable Groups & VGPR Cost
| Category | Variables | Count | Type | VGPRs | Reducible |
|----------|-----------|-------|------|-------|-----------|
| ... | ... | ... | ... | ... | Yes/No |

## 3. Peak Liveness Analysis

### Critical Point 1: <name>
| Alive Group | VGPRs |
|-------------|-------|
| ... | ... |
| **TOTAL** | **XXX** |

### Critical Point 2: <name>
...

### Peak: <CP name> = <N> VGPRs (overflow: <M>)

## 4. Optimization Suggestions (sorted by impact)

### [HIGH] OPT-A: <title> — saves ~XX VGPRs
- **What**: <description>
- **Pattern found**: <code pattern location>
- **Trade-off**: <cost>

### [MEDIUM] OPT-C: <title> — saves ~XX VGPRs
...

### Non-Reducible
- <category>: <reason>

## 5. Projected Peak After Optimization
| Scenario | Peak VGPRs | vs 256 Limit | Spill Expected |
|----------|-----------|-------------|----------------|
| Current | XXX | +YY | Yes (severe) |
| +OPT-A | ... | ... | ... |
| +OPT-B | ... | ... | ... |
| +OPT-A+B | ... | ... | No |
| +All | ... | ... | No |
```

---

## Worked Example: MoE Blockscale Stage1 Kernel

For reference, here is the analysis of `kernels/moe_blockscale_2stage.py` (baseline
at commit `2df60fd`) with tile 64×128×128 on gfx950:

**Constants**: m_repeat=4, num_acc_n=2, k_unroll=2, sb_per_tile=1, num_x_loads=2

**VGPR Cost Breakdown**:

| Category | Count × Type | VGPRs | Reducible |
|----------|-------------|-------|-----------|
| acc_gate + acc_up | 8 × vec4_f32 | 64 | No |
| b_gate_flat + b_up_flat (loop) | 16 × i64 | 32 | Yes → OPT-A |
| a0_prefetch (loop) | 2 × i64 | 4 | Yes → OPT-E |
| combined scales | 64 × f32 | 64 | Yes → OPT-C |
| x_regs | 2 × vec4_i32 | 8 | No |
| B tile next stage | 16 × i64 | 32 | Lifecycle |
| block_gate/up_accs | 16 × vec4_f32 | 64 | Yes → OPT-B |
| MFMA temps | mixed | 24 | No (transient) |

**Peak**: CP1 (Stage 0 Compute) = **292 VGPRs** (overflow 36)

**After OPT-A (Opt2)**: 292 - 36 = 256 → **200 VGPRs** (B tiles no longer at yield)
**After OPT-B (Opt1)**: 292 - 56 = **236 VGPRs**
**After OPT-A+B**: **~144 VGPRs** (well within 256)

**Actual ISA results**:
- Baseline: vgpr_spill=177, scratch=596 bytes, stage1=12020 us
- +OPT-A only: vgpr_spill=49, scratch=192 bytes, stage1=2910 us (4.13x)
- +OPT-A+B+C+D: vgpr_spill=13, scratch=48 bytes, stage1=2220 us (5.41x)

---

## Error Handling

- **ISA file not found**: Ask user to run `/dump-ir <cmd>` first to generate ISA
- **No `.amdgpu_metadata` section**: The file may not be a final ISA. Check for `15_final_isa.s`
- **No hot loop found**: Kernel may not have a main loop (single-pass kernel). Analyze the full function body instead
- **Cannot determine tile constants**: Ask user for tile_m, tile_k, tile_n values
- **vgpr_spill_count == 0**: Report "No VGPR pressure issue detected" with the current metrics. The kernel is within register budget.

## Next Steps

After analysis, suggest:
1. Apply the highest-impact optimization and re-run `/dump-ir` to verify spill reduction
2. Use `/kernel-trace-analysis` to profile before/after and measure actual performance impact
3. For severe spill (>100), consider applying multiple optimizations together
