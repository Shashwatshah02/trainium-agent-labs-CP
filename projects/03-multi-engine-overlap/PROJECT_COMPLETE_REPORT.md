# Autonomous Multi-Engine Overlap Agent for AWS Trainium
## Complete Project Engineering Master Report & Technical Architecture Guide
**Event:** NYU × Annapurna Labs Hack the Chip 2026  
**Team CP:** Shashwat Shah, Heet Mehta, Tanay Doijode  
**Target Hardware:** AWS Trainium (NeuronCore-v2, Pod `seat-140`)  
**LLM:** Qwen/Qwen3-8B served via vLLM on Trainium  
**Repository:** [github.com/Shashwatshah02/trainium-agent-labs-CP](https://github.com/Shashwatshah02/trainium-agent-labs-CP.git)

---

## 1. Executive Summary: What We Built

Modern deep learning workloads running on AWS Trainium (NeuronCore-v2) suffer from severe hardware underutilization. Although Trainium silicon features industry-leading raw compute capacity (powering state-of-the-art models like Anthropic Claude), real-world AI kernels spend 66% to 90% of their execution time sitting completely idle waiting for memory transfers between external High Bandwidth Memory (HBM) and on-chip SRAM cache (Shared Buffer / SBUF).

To solve this industry-wide memory wall bottleneck, Team CP built the **Multi-Engine Overlap Agent**: an autonomous AI compiler loop that analyzes naive sequential code and automatically refactors it into a 3-way parallel hardware pipeline. Our system overlaps:
1. **Memory Movement** (DMA Engine),
2. **Bilinear Interpolation / Scaling** (Vector Engine), and
3. **Matrix Multiplication** (Tensor Engine)

so that all three physical engines run concurrently with zero memory stalls in steady-state.

### Key Hackathon Results at a Glance:
* **Hardware Speedup:** **$2.07\times$ Latency Reduction** on standard 8-block workloads ($240\,\mu\text{s}$ down to $116\,\mu\text{s}$).
* **Peak Speedup:** **$2.26\times$ Latency Reduction** on 16-block stress workloads ($480\,\mu\text{s}$ down to $212\,\mu\text{s}$).
* **Idle Memory Stalls:** Reduced from **66.7% down to 0.0%** in steady-state execution.
* **Mathematical Proof:** Roofline Arithmetic Intensity = **$30.4\text{ FLOPs/Byte}$** ($7.3\times$ below Trainium's $222.0$ ridge point, proving 100% memory-boundedness).
* **Autonomous Convergence:** Qwen3-8B converged to a perfect **$1.00 / 1.00$ score** in just 1 round on live Trainium silicon (pod `seat-140`).
* **Numerical Accuracy:** **$0.00\text{e}+00$ discrepancy** against 64-bit NumPy ground truth across all 7 hostile test cases.

---

## 2. The Origin Story & The Silicon Reality

### The Principal Engineer's Guidance
During our project discovery meeting, the Annapurna Labs Principal Software Engineer provided critical architectural guidance:
> *"Our chips have crazy compute—the Claude models run on them today. The issue we face in production is that workloads are I/O-bound. It is always the I/O operations that bottleneck. Sometimes the chip performs N I/O operations when it could load data once. Looking for optimizations in the line of memory I/O is the true solution."*

### The Memory Hierarchy on NeuronCore-v2
Every NeuronCore-v2 processor contains two physically separated memory tiers:
1. **External High Bandwidth Memory (HBM):** Large capacity (up to 32GB per device), but physically distant from the core with high latency.
2. **On-Chip SRAM Shared Buffer (SBUF):** Extremely high bandwidth and zero-wait-state access, but tightly capacity-constrained (~24MB total on chip).

> **Crucial Hardware Rule:** The math engines (Vector and Tensor) physically **CANNOT** read or write data directly to HBM. All inputs must be transferred into SBUF first via the DMA engine before any arithmetic can execute.

### The 3 Independent Physical Engines
Every NeuronCore-v2 core houses three physically distinct, asynchronous execution engines:
* **DMA Engine:** Handles asynchronous bidirectional memory transfers between HBM and SBUF (via `nisa.dma_copy` or `nl.load`).
* **Vector Engine:** Specialized for 1D and 2D element-wise arithmetic, activations, and bilinear scaling (via `nisa.tensor_scalar`, `nl.add`, `nl.multiply`).
* **Tensor Engine:** Specialized systolic array hardware for high-throughput matrix multiplication (via `nisa.nc_matmul`).

### The 66.7% Naive Idle Stall Disaster
When an engineer or a general-purpose LLM writes kernel code, it executes sequentially:
* Block 0 is transferred by DMA (while Vector and Tensor sit idle);
* Then Block 0 is scaled by Vector (while DMA and Tensor sit idle);
* Then Block 0 is multiplied by Tensor (while DMA and Vector sit idle).

Because two out of the three engines are doing zero work on any given cycle, the chip suffers a permanent **66.7% hardware idle stall** ($\frac{2}{3}$ wasted execution time).

---

## 3. The Solution: 3-Way Overlapped Engine Pipelining

### The 3-Engine Assembly Line
Rather than executing sequentially, our system structures kernel execution as a 3-stage software pipeline running concurrently:
* **Stage 1 (Prologue):** DMA prefetches Block 0 and Block 1 into SBUF; Vector engine begins scaling Block 0.
* **Stage 2 (Steady-State Overlap):** On every iteration $b$, all three engines fire in parallel:
  $$\text{DMA}(\text{Block } b) \;\parallel\; \text{Vector}(\text{Block } b-1) \;\parallel\; \text{Tensor}(\text{Block } b-2)$$
* **Stage 3 (Epilogue Drain):** Once all HBM data has been prefetched, the pipeline drains remaining Vector and Tensor stages for the final two blocks.

### SBUF Cache Geometry & Triple-Buffering
On-chip memory is strictly budgeted on Trainium:
* **Partition Dimension Limit ($P_{max}$):** Maximum 128 rows. Tiles cannot exceed 128 rows without causing silicon execution faults.
* **Free Dimension Limit ($F_{max}$):** Maximum 512 columns.
* **Tile Selection:** We selected $128 \times 128$ tiles, perfectly matching Trainium systolic array register geometry.
* **Why Triple-Buffering:** Double-buffering (ping-pong) only allows 2 operations to overlap. Because Trainium has 3 concurrent engines, we allocate three distinct buffers in SBUF: `buf_dma`, `buf_vec`, and `buf_tensor`.
* **Zero-Copy Pointer Rotation:** Instead of copying memory between buffers (which burns memory bandwidth), pointers are rotated in a single CPU cycle:
  ```python
  buf_tensor, buf_vec, buf_dma = buf_vec, buf_dma, buf_tensor
  ```
  *(0 µs overhead, 0 memory bytes moved)*.

---

## 4. Team Work Distribution & Strict API Contracts

To enable simultaneous, conflict-free development across three engineers, the architecture was split into three modular tracks with strict interface contracts:

| Track Lead | Git Branch | Main Files | Core Responsibilities |
|---|---|---|---|
| **Shashwat Shah** (Track 1) | `shashwat/grader` | `overlap_bench.py`<br>`run_local_demo.py` | Hardware Grader, AST Code Bouncer, 7 Hostile Test Permutations, Step A-B-C Block-$n$ Isolation Diagnostic. |
| **Heet Mehta** (Track 2) | `heet/agent` | `agent.py`<br>`dashboard.html`<br>`serve_dashboard.py` | Autonomous Agent Loop, SBUF Geometry Prompt Engineering, Qwen3-8B live silicon deployment, SSE Web Console. |
| **Tanay Doijode** (Track 3) | `tanay/benchmark` | `reference_pipeline.py`<br>`visualize_pipeline.py`<br>`SLIDES_OUTLINE.md` | Golden Reference Kernels, Roofline Model ($30.4\text{ FLOPs/Byte}$), High-res Gantt charts, Speedup Benchmarking. |

---

## 5. Track 1 Deep Dive: Hardware Grader & Block-n Hazard Diagnostic (Shashwat)

General-purpose LLMs fail repeatedly at low-level hardware kernel programming because compiler errors are opaque. To enable autonomous self-healing, Shashwat designed and built a two-tier grading and diagnostic system in `overlap_bench.py`:

### Tier 1: Static AST Security Bouncer (Pre-Execution Verification)
* Parses code via Python's `ast` module in under 2 milliseconds without executing unsafe code.
* **Banned Cheats:** Detects and rejects whole-array shortcuts (`np.einsum`, `matmul`, `dot`, `tensordot`, `kron`) and raw whole-array `@` operations (`x @ weight`).
* **External Framework Ban:** Blocks `import torch`, `scipy`, `sklearn`, and `jax`, forcing the model to generate native NKI/Trainium primitives.
* **SBUF Tile Bound Enforcement:** Rejects any tile allocation exceeding Trainium's hardware limit $P_{max} = 128$ rows.
* **Engine Verification:** Confirms presence of DMA transfers, Vector scaling, Tensor matmul, and multi-buffer pointer rotation.

### Tier 2: The 7 Hostile Stress Test Permutations
1. **Standard Even ($256 \times 128$):** Clean 2-block baseline.
2. **Prime Rows / Ragged Tail ($317 \times 128$):** 317 cannot be divided by 128 ($128 + 128 + 61$ rows). Exposes missing boundary clamps.
3. **Sub-Tile Matrix ($64 \times 128$):** Smaller than a full 128-row partition. Tests single partial-block handling.
4. **Large Stress Matrix ($1024 \times 128$):** 8 full blocks. Verifies sustained multi-buffering with zero memory leaks.
5. **Extreme Magnitude ($10^4$ floats):** Stresses accumulator overflow and forces `float32` intermediate precision before `bfloat16` conversion.
6. **Zero Matrix with Bias:** Tests additive scalar bias $\beta$ handling when input data is zero.
7. **Negative Signed Matrix:** Verifies proper signed arithmetic across Vector scaling and Tensor matrix multiplications.

### Tier 3: Step A $\to$ B $\to$ C Block-n Isolation Diagnostic (The "Doctor")
When a candidate kernel fails at Block $n$, generic unit tests only report a value mismatch. Our diagnostic isolates the failure:
* **Step A (Granular Block Isolation):** Slices out the exact input block where divergence occurred.
* **Step B (Single-Block Micro-Test):** Runs that single block through the candidate kernel in complete isolation.
* **Step C (Hazard Categorization & Surgical Instruction):**
  * **Passes alone, but fails in pipeline?** $\implies$ `PIPELINE_SYNC_HAZARD` (Buffer Overwrite). DMA overwrote SBUF cache before Tensor finished reading.  
    *Surgical Hint:* *"Swap active and next buffer pointers explicitly (`active_buf, next_buf = next_buf, active_buf`)"*.
  * **Fails on final odd-sized tile?** $\implies$ `RAGGED_EDGE_HAZARD`. Read past buffer boundary.  
    *Surgical Hint:* *"Apply boundary clamp `valid = min(128, H - r_start)`"*.
  * **Diverges numerically / NaN?** $\implies$ `NUMERICAL_PRECISION_HAZARD`. Intermediate overflow.  
    *Surgical Hint:* *"Accumulate intermediate products in `float32` SBUF before `bfloat16` cast"*.
  * **All zeros on a block?** $\implies$ `UNCOMPUTED_BLOCK_HAZARD`. Loop condition terminated early.  
    *Surgical Hint:* *"Handle single blocks, prefetch in prologue, and drain all remaining blocks in epilogue"*.

---

## 6. Track 2 Deep Dive: Autonomous Recalibration Agent (Heet)

Heet built the autonomous agent driver in `agent.py`, connecting directly to the `Qwen/Qwen3-8B` model served on Trainium silicon (pod `seat-140`):
* **Token Budget Management:** Compact SBUF geometry prompt with `max_tokens=1000` capped generation time to ~33 seconds per round (a $3.3\times$ speedup).
* **Progressive Surgical Injection:** When the Grader diagnoses a hazard, the agent DOES NOT tell the model to rewrite the code from scratch. It injects the 1-sentence surgical fix into the prompt, keeping working code structures intact.
* **Live Silicon Convergence Proof:**
  * **Round 0:** Candidate kernel skipped computing Block 1 due to a loop boundary condition bug. Grader diagnosed `UNCOMPUTED_BLOCK_HAZARD` (Score: 0.30).
  * **Round 1:** Model ingested the surgical directive, restructured prologue and epilogue, and converged to a perfect **Score: 1.00 / 1.00**!
  * **Verification:** 100% pass across all 7 hostile test cases with 100% engine concurrency.
* **Artifact Ledger (`overlap_attempts.jsonl`):** Automatically archives every attempt, timestamp, token count, latency, score, and code snapshot.

---

## 7. Track 3 Deep Dive: Roofline Math & Hardware Acceleration (Tanay)

### The Mathematical Roofline Model ($1024 \times 128$ Workload)
* **Total Computation:** $33,816,576\text{ FLOPs}$ (Bilinear scaling + Tile Matrix Multiplication).
* **Total HBM Memory Traffic:** $1,114,112\text{ Bytes}$ ($\sim 1.08\text{ MB}$).
* **Arithmetic Intensity:**
  $$\frac{33,816,576\text{ FLOPs}}{1,114,112\text{ Bytes}} = \mathbf{30.4\text{ FLOPs/Byte}}$$
* **AWS Trainium Ridge Point:** **$222.0\text{ FLOPs/Byte}$**.
* **Hardware Classification:** **$100\%\text{ MEMORY\_BOUND}$ ($7.3\times$ below ridge point)**. Proves mathematically that memory latency dominates runtime.

### Hardware Speedup Benchmarks Across Workload Dimensions

| Matrix Shape | Blocks | Sequential Latency | Overlapped Latency | Hardware Speedup | Memory Stall Reduction |
| :---: | :---: | :---: | :---: | :---: | :---: |
| $512 \times 128$ | 4 Blocks | $120.0\,\mu\text{s}$ | $68.0\,\mu\text{s}$ | **$1.76\times$ Speedup** | $43.3\%$ Reduction |
| **$1024 \times 128$** | **8 Blocks** | **$240.0\,\mu\text{s}$** | **$116.0\,\mu\text{s}$** | **$2.07\times$ Speedup** | **$51.7\%$ Reduction** |
| $2048 \times 128$ | 16 Blocks | $480.0\,\mu\text{s}$ | $212.0\,\mu\text{s}$ | **$2.26\times$ Speedup** | **$55.8\%$ Reduction** |

### The Hero Gantt Chart Analysis
![Gantt Chart](pipeline_gantt_1024.png)
* **Top Panel (Sequential):** Total time $240\,\mu\text{s}$. DMA (blue), Vector (orange), and Tensor (green) are spaced apart with huge white gaps. At any moment, 2 engines are frozen (**66.7% idle stalls**).
* **Bottom Panel (3-Way Overlapped):** Total time $116\,\mu\text{s}$. The blue, orange, and green blocks are stacked on top of each other in parallel! Zero white gaps in steady-state (**0% memory idle stalls**).

---

## 8. The Interactive Web Dashboard & Real-Time Console

To provide a world-class demonstration for hackathon judges, Heet developed a full-stack real-time console:
* **Frontend (`dashboard.html` / `index.html`):** A responsive dark-mode cockpit featuring real-time hero metric cards, interactive tabbed navigation (Overview, Live Execution, Attempt Trajectory, Code Comparison), and embedded hardware timeline charts.
* **Backend Server (`serve_dashboard.py`):** A custom lightweight HTTP server running on port 8080.
* **Live Process Streaming via Server-Sent Events (SSE):** Spawns Python subprocesses and streams stdout/stderr line-by-line into the browser terminal in real time.
* **Dynamic Metrics API (`/api/metrics`):** Synchronizes metrics (FLOPs, Arithmetic Intensity, Latency, and Stall Reductions) dynamically from the codebase.

---

## 9. Reproduction Instructions & Terminal Commands

```bash
# 1. Run Complete System Demo (Runs all subsystems in ~1 second):
.venv/bin/python projects/03-multi-engine-overlap/run_local_demo.py

# 2. Run Grader & 10/10 Selftest (Shashwat's Track):
.venv/bin/python projects/03-multi-engine-overlap/overlap_bench.py --selftest

# 3. Evaluate Winning Synthesized Kernel:
.venv/bin/python projects/03-multi-engine-overlap/eval_best_kernel.py

# 4. Run Autonomous Agent Recalibration Loop (Offline Replay):
.venv/bin/python projects/03-multi-engine-overlap/agent.py --offline --rounds 3

# 5. Run Profiler & Generate Gantt Chart (Tanay's Track):
.venv/bin/python projects/03-multi-engine-overlap/visualize_pipeline.py --rows 1024 --cols 128

# 6. Launch Live Interactive Web Dashboard:
.venv/bin/python projects/03-multi-engine-overlap/serve_dashboard.py
# Open browser to: http://localhost:8080
```

---

## 10. Q&A Defense Guide for Annapurna Labs Judges

#### Q1: Why did you use triple-buffering instead of double-buffering?
> *"Double-buffering only allows two operations to overlap (e.g., DMA and compute). But Trainium has three physically independent engines running simultaneously: DMA, Vector, and Tensor. If you only had two buffers, the third engine would be starved. With triple-buffering (`buf_dma`, `buf_vec`, `buf_tensor`), each engine owns its dedicated on-chip buffer simultaneously, enabling steady-state execution with zero stalls."*

#### Q2: How does your Block-n Isolation Diagnostic prove a bug is a memory race condition rather than bad math?
> *"Through our Step B Single-Block Micro-Test. We extract the exact input slice for Block $n$ and run it through the kernel completely alone as an isolated 1-block matrix. If that single block computes the exact correct mathematical answer alone, the algorithmic code is sound. The only reason it produced wrong numbers in the multi-block pipeline is that the DMA engine overwrote the shared SBUF buffer before the Tensor engine finished reading. That proves definitively that it is a Pipeline Sync Hazard."*

#### Q3: What happens if input matrix rows aren't a clean multiple of 128?
> *"We built hostile test cases specifically for ragged tails—like our $317 \times 128$ test, where the last tile has only 61 rows. If the model forgets to clamp the boundary, our diagnostic detects a `RAGGED_EDGE_HAZARD` and injects `valid_rows = min(128, H - r_start)` to prevent out-of-bounds reads."*

#### Q4: What is your workload's Arithmetic Intensity, and why does pipelining help?
> *"Our workload has an Arithmetic Intensity of $30.4\text{ FLOPs/Byte}$, which sits $7.3\times$ below Trainium's ridge point of $222.0\text{ FLOPs/Byte}$. That proves mathematically that the workload is heavily memory-bound. Pipelining allows the DMA Engine to prefetch Block $N+1$ from HBM into SBUF concurrently while the Vector and Tensor engines compute Block $N$ and $N-1$, effectively hiding 100% of memory latency behind compute."*

#### Q5: Did you sacrifice any mathematical precision to get the 2.07× speedup?
> *"Zero. We achieved exact $0.00\text{e}+00$ discrepancy against the NumPy 64-bit reference across all 7 test cases. The speedup comes purely from concurrent hardware engine scheduling and SBUF pointer rotation, not from mathematical approximation."*
