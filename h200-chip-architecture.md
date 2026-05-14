# The H200 and Modern AI Chip Architecture

*A field guide to what's actually inside the chips training and running modern AI models*

---

## Why this matters

Most discussions of AI hardware bottom out at "you need a lot of GPUs." That's true but not very useful. The actual architecture — what's inside a Hopper-class chip, why HBM3e changed the inference game, what an SM and a Tensor Core actually do — explains a lot of things that are otherwise mysterious: why models can't just be "made faster," why context window expansion is expensive, why DeepSeek's training on Huawei chips was a real signal rather than marketing, and why Nvidia's moat is durable.

This guide focuses on the **H200** as the anchor, since it's the workhorse of frontier AI infrastructure in 2026. The H100 came before it (same compute, less memory). The B100/B200 (Blackwell) and GB200 came after (new architecture, more compute, way more memory). Understanding the H200 well makes the others easy.

---

## The 60-second version

The H200 is a giant chip designed around two ideas:

1. **Do enormous amounts of matrix multiplication in parallel.** Neural networks are mostly matrix math, so the chip is mostly matrix-math units (called Tensor Cores) replicated thousands of times.
2. **Feed those matrix units fast enough.** Modern AI workloads are usually memory-bound, not compute-bound — the matrix units sit idle waiting for weights to arrive. So the chip pairs that compute with an extraordinarily fast memory system (HBM3e).

Everything else — the SM hierarchy, NVLink, the Transformer Engine, FP8 precision — exists to serve those two goals.

---

## Headline specs

| Spec | H200 SXM | H100 SXM (for comparison) |
|------|----------|--------------------------|
| Architecture | Hopper | Hopper |
| GPU die | GH100 (same as H100) | GH100 |
| Process node | TSMC 4N (custom 5nm) | TSMC 4N |
| Transistors | ~80 billion | ~80 billion |
| SMs (streaming multiprocessors) | 132 | 132 |
| CUDA cores | 16,896 | 16,896 |
| 4th-gen Tensor Cores | 528 | 528 |
| **Memory** | **141 GB HBM3e** | 80 GB HBM3 |
| **Memory bandwidth** | **4.8 TB/s** | 3.35 TB/s |
| FP8 throughput (with sparsity) | ~3,958 TFLOPS | ~3,958 TFLOPS |
| TDP | 700W | 700W |
| NVLink | 4th-gen, 900 GB/s bidirectional | 4th-gen, 900 GB/s |
| Form factor | SXM5 or NVL (PCIe Gen5) | SXM5 or PCIe |
| Approx. price (2026) | $35-45K (NVL), ~$45-50K (SXM) | $25-35K |

The crucial thing this table makes obvious: **the H200 is the H100 with better memory.** Same die, same compute. The 1.76× memory capacity and 1.43× memory bandwidth are what make it a different product — because AI inference is overwhelmingly memory-bound.

---

## The big picture: what a GPU actually is

Forget the gaming-card mental model. A modern data-center GPU is a **massively parallel matrix-math machine** with a hierarchical compute structure and a custom-built memory subsystem glued to it.

The H200, like the H100, is built from the **GH100 die**. Inside that die, the structure is hierarchical:

```
GH100 die (full implementation: 144 SMs, but H200/H100 ship with 132 enabled)
│
├── 8 GPCs (Graphics Processing Clusters)
│   │
│   └── 9 TPCs (Texture Processing Clusters) per GPC
│       │
│       └── 2 SMs (Streaming Multiprocessors) per TPC
│           │
│           ├── 128 CUDA cores
│           ├── 4 Tensor Cores (4th generation)
│           ├── Tensor Memory Accelerator (TMA)
│           ├── Register file (256 KB)
│           └── Shared memory / L1 cache (256 KB combined)
│
├── 60 MB L2 cache (split across two partitions)
└── 6 HBM3e memory stacks (24 GB each = 144 GB physical, 141 GB usable)
```

A few things to internalize from this:

- **The SM is the fundamental unit of compute.** When people say a GPU has "132 SMs," that's the count that matters most for raw throughput.
- **CUDA cores are not the right number to look at.** You'll see "16,896 CUDA cores" in marketing material, but for AI workloads what matters is the **Tensor Cores** (528 of them) and how fast they're fed.
- **The chip is symmetric**, and that's the whole point: it's the same SM replicated 132 times, all running the same kernel on different data. This is "single instruction, multiple data" (SIMD) at extreme scale.

---

## Inside an SM (Streaming Multiprocessor)

Each SM is itself divided into **four quadrants**, each containing:

- **16 INT32 units** — for integer math (loop counters, indexing, etc.)
- **32 FP32 units** — for single-precision floating-point (the classic "CUDA core")
- **16 FP64 units** — for double-precision (used in HPC, not really in AI)
- **1 fourth-generation Tensor Core** — for matrix-multiply-accumulate

So each SM has 4 Tensor Cores total. Across 132 SMs that's 528 Tensor Cores.

### The Tensor Core: why this chip exists

A Tensor Core is a specialized circuit that performs **matrix-multiply-accumulate (MMA)** in one operation. Given matrices A, B, and C, it computes `D = A × B + C` in a single hardware instruction, on small matrix tiles, in parallel.

Why does this matter? Because every neural network layer reduces to enormous matrix multiplications. A standard FP32 unit can multiply two scalars per clock. A Tensor Core can multiply a small matrix by another small matrix per clock — orders of magnitude more arithmetic per cycle.

The 4th-generation Tensor Cores in Hopper deliver:
- **2× the MMA throughput per SM** vs A100 (Ampere) on equivalent data types
- **4× the throughput** when using the new FP8 data type vs A100's FP16
- **Sparsity support**: when matrices have a structured pattern of zeros (2:4 sparsity, meaning 2 zeros in every 4 elements), the Tensor Core skips them and effectively doubles throughput

### FP8 and the Transformer Engine

One of Hopper's signature features is **FP8 precision** — 8-bit floating-point numbers, which sounds absurdly low until you understand how they're used.

FP8 comes in two flavors:
- **E4M3** (4 exponent bits, 3 mantissa bits): more precision, less range. Used for weights and activations in the forward pass.
- **E5M2** (5 exponent bits, 2 mantissa bits): more range, less precision. Used for gradients in the backward pass.

The **Transformer Engine** is a piece of hardware-plus-software that automatically chooses which precision to use for each tensor on each layer, dynamically. The idea: some operations need high precision (gradient accumulation), most don't (intermediate activations). By dropping precision where it's safe, you get up to 2× speedup over FP16 with no measurable accuracy loss.

This is why the H100/H200 was a step-change for transformer training and inference specifically. The chip has hardware that *knows about transformer math*. AMD and others have caught up, but Nvidia got there first and the software ecosystem is built around it.

### The Tensor Memory Accelerator (TMA)

A small but architecturally significant feature: a dedicated hardware unit that handles large memory transfers between global memory (HBM3e) and shared memory (on-SM) asynchronously, without occupying CUDA threads.

Before TMA, you'd use CUDA threads to manage memory transfers, which meant those threads couldn't do useful math while data was in flight. TMA frees up the entire SM to keep doing matrix multiplications while data streams in for the next tile.

This matters because **most AI workloads are memory-bound**, and any feature that overlaps compute with memory movement is a big deal in practice.

---

## The memory subsystem (the actual headline)

Here is where the H200 earns its name. Same die as the H100, but a completely different memory configuration.

### HBM3e

**HBM** stands for **High Bandwidth Memory**. It's a different physical technology from the GDDR memory in gaming GPUs — instead of being arranged in a flat row of chips next to the GPU, HBM is **stacked vertically** and bonded directly to the GPU package through silicon interposers. The result: enormously wide memory buses (1024 bits per stack vs 32 bits for GDDR) at moderate clock speeds, yielding extraordinary bandwidth per stack.

The **3e** in HBM3e is the generation. Each generation roughly doubles bandwidth:
- HBM (original, 2015): ~128 GB/s per stack
- HBM2: ~256 GB/s per stack
- HBM2e: ~460 GB/s per stack
- HBM3: ~670 GB/s per stack (this is what the H100 used)
- **HBM3e: ~800 GB/s per stack (H200 uses this)**
- HBM4: ~1.5 TB/s per stack (coming with Rubin generation)

### Why 141 GB and not 144 GB?

The H200 has **six HBM3e stacks** of **24 GB each**, for 144 GB physical. But it ships with **141 GB usable** — Nvidia reserves a portion for error correction overhead and some internal headroom. The marketing number is 141.

### Why bandwidth matters more than you'd think

Modern LLM inference is dominated by **memory bandwidth, not compute**.

To generate each token, the model has to read every parameter from memory at least once. A 70B parameter model in FP16 is 140 GB. At 4.8 TB/s, you can read those parameters about 34 times per second — which is roughly your upper bound on tokens per second for that model on one GPU, regardless of how much compute the chip has.

This is why the H200 doubles the inference throughput of the H100 on Llama2-70B despite having identical compute. The bottleneck was never compute; it was memory bandwidth and capacity.

This is also why **HBM is the actual battleground** of AI hardware competition. Nvidia, AMD, and the Chinese alternatives are not really competing on raw FLOPS — they're competing on memory hierarchy.

### The memory hierarchy in full

On a Hopper-class GPU, memory is structured as:

| Level | Size | Bandwidth | Latency |
|-------|------|-----------|---------|
| Registers (per SM) | 256 KB | Effectively free | ~1 cycle |
| Shared memory / L1 (per SM) | 256 KB | ~33 TB/s aggregate | ~30 cycles |
| L2 cache (shared) | 60 MB | ~12 TB/s | ~200 cycles |
| HBM3e (global memory) | 141 GB | 4.8 TB/s | ~500 cycles |
| NVLink to neighbor GPU | — | 900 GB/s | ~1000+ cycles |
| PCIe to host CPU | — | ~64 GB/s | ~5000+ cycles |

**Performance engineering on these chips is essentially the art of keeping data in the highest possible level of this hierarchy for as long as possible.** A kernel that hits HBM on every access can be 10–100× slower than one that reuses data in shared memory.

This is why writing fast CUDA code is hard, why Flash Attention was a big deal (it's a clever way to compute attention while keeping data in SRAM), and why most engineers will never write a custom kernel — the optimized libraries (cuDNN, cuBLAS, FlashAttention) already do it for them.

---

## Interconnects: scaling beyond one chip

A single H200 has 141 GB. Frontier models have hundreds of billions to trillions of parameters. So they have to be sharded across many GPUs, and the speed at which those GPUs can talk to each other becomes a primary determinant of training and inference performance.

### NVLink

Nvidia's high-speed GPU-to-GPU interconnect. The H200 uses **4th-generation NVLink** with **900 GB/s bidirectional bandwidth per GPU**.

Compare to PCIe Gen5: about 64 GB/s. NVLink is roughly 14× faster.

In an **HGX H200 8-GPU board** (the standard configuration for AI servers), all 8 GPUs are connected to each other through NVLink and an NVSwitch fabric, giving any-to-any communication at the full 900 GB/s. The aggregate memory across 8 H200s is ~1.1 TB — enough to fit a 1T-parameter model in FP8.

### NVL variant

The H200 NVL is the PCIe-form-factor version (vs the SXM5 used in HGX boards). NVL stands for "NVLink" and refers to a more limited bridging configuration:
- 2-way or 4-way NVLink bridges between adjacent cards (still 900 GB/s between bridged pairs)
- Lower TDP (~600W vs 700W)
- Air-coolable, drop-into-existing-server-rack
- About 15-20% lower throughput than SXM5

NVL exists because not every enterprise wants to redo their entire data center to install liquid-cooled SXM5 systems. It's a less powerful but more deployable variant.

### Beyond the box: InfiniBand and Spectrum-X

When you go beyond 8 GPUs (one HGX node), you leave NVLink territory and enter networking territory. Nvidia's answer here is **InfiniBand** (acquired via Mellanox in 2019), specifically the **NDR 400 Gbps** generation for current-gen AI clusters.

Spectrum-X is Nvidia's Ethernet-based alternative aimed at hyperscalers who want to use standard Ethernet rather than proprietary InfiniBand. The performance is close-but-not-identical; the play is about ecosystem lock-in.

Either way, the scaling story for frontier training looks like:

```
GPU → SXM5 socket → HGX board (8 GPUs, NVLink) → Server node → InfiniBand → Rack → 
Cluster → Supercluster (10k-100k GPUs) → "Gigacluster" (100k+ GPUs, 1+ GW power)
```

This stack is what people mean when they say "compute" in the AI capex discussion. A frontier training run touches every layer of it.

---

## Multi-Instance GPU (MIG)

Briefly: the H200 supports partitioning itself into **up to 7 isolated GPU instances** that share the physical hardware but appear to applications as separate GPUs.

Useful for:
- Serving many small models on one expensive GPU
- Multi-tenant cloud deployments
- Development environments where you don't need a full H200 per developer

Less useful for frontier training or large-model inference (you want the whole chip).

---

## Comparison to siblings and rivals

### vs H100

Same compute. 1.76× memory capacity. 1.43× memory bandwidth. About 1.5–1.9× faster on LLM inference (memory-bound). About 1.0–1.1× faster on training (compute-bound at large batch sizes). The H200 makes the most sense for inference workloads and for training models too large to fit on H100 with reasonable batch sizes.

### vs B100/B200 (Blackwell)

The Blackwell generation, shipping in volume from 2024-2025:
- New architecture (not Hopper-derived)
- ~2.5× the FP8 compute of H200
- 192 GB HBM3e (B200) — even more memory
- New FP4 precision support — drops some workloads to 4-bit
- ~1000W TDP (B200) — requires liquid cooling
- **Two dies per package** (B200 is technically two reticle-limited dies stitched together via NVLink-C2C)

### vs GB200 (Grace Blackwell)

GB200 is one ARM-based "Grace" CPU plus two B200 GPUs in a single package, connected via NVLink-C2C at 900 GB/s. The pitch is that CPU and GPU share a coherent memory space — useful when models spill out of GPU memory into host memory.

### vs AMD MI300X / MI325X

AMD's flagship. The MI300X has 192 GB of HBM3 (more than H200's 141 GB), competitive compute, and dramatically lower price per chip. The reason it hasn't displaced Nvidia: the software ecosystem. CUDA is 18 years old; ROCm (AMD's equivalent) is still catching up, especially for training. AMD is winning some inference deployments where memory capacity matters more than software polish.

### vs Google TPU v5p / Trillium

Google's custom AI chips, used internally to train Gemini and offered via GCP. Different architecture (systolic array rather than SM/Tensor Core hierarchy), but conceptually similar end function. The catch: TPUs are not generally available outside GCP, and the software stack is JAX-flavored rather than PyTorch-native.

### vs Huawei Ascend

China's domestic alternative, used by DeepSeek to train V4 — the first time a frontier-class model trained on non-Nvidia, non-US hardware. Performance per chip is lower than H100, but Huawei has been able to ship them at scale despite US export controls. A serious geopolitical signal even if not yet a serious commercial threat.

---

## How to read AI hardware news after this

A few mental shortcuts the above unlocks:

**"Model X needs N GPUs"** → calculate roughly: parameter count × bytes per parameter ÷ HBM per GPU. A 1T-parameter MoE in FP8 needs ~1 TB of GPU memory, so 8 H200s minimum. Add ~30-50% headroom for activations, KV cache, and optimizer state during training.

**"X% faster inference than H100"** → almost always means "the memory bandwidth is X% higher" if it's a non-Blackwell chip. Inference is memory-bound.

**"X TFLOPS" claims** → only meaningful with the precision specified (FP8 ≠ FP16 ≠ FP32) and the sparsity caveat. A chip advertising "5 PFLOPS" of FP8 with 2:4 sparsity has half that in dense FP8.

**"Cluster of X GPUs"** → the interconnect matters as much as the chip count. 16,000 H200s on slow networking will train a frontier model worse than 8,000 H200s on InfiniBand NDR.

**"Custom silicon" announcements** → check the memory subsystem and the software stack, not the compute claims. Anyone can build matrix-multiply units; nobody has matched HBM + NVLink + CUDA as a complete package.

---

## What to expect next

Some things that are already shipping or will be soon:

- **HBM4** memory: starts shipping in late 2026/2027, paired with Rubin-generation Nvidia chips. Expect ~1.5 TB/s per stack, so possibly 8+ TB/s aggregate for next-gen chips.
- **FP4 and lower** precision: Blackwell introduced FP4; expect more workloads to drop to it. Training will likely stay at FP8 minimum; inference will keep pushing lower.
- **Photonic interconnects**: replacing copper with optical links between GPUs and racks. Several startups (Ayar Labs, Lightmatter) and Nvidia itself are working on this. Promises step-change improvements in inter-GPU bandwidth.
- **Wafer-scale chips**: Cerebras has been doing this for years, building chips the size of dinner plates. Niche today, but the design constraint that reticle-limited dies impose on Nvidia is real, and someone might break out of it.
- **3D-stacked compute**: putting compute logic directly on top of memory rather than next to it. The endgame for memory-bound workloads.

---

## Quick glossary of chip terms

**CUDA core** — General-purpose floating-point unit in an SM. The "thousands of CUDA cores" marketing number. Less important than Tensor Cores for AI.

**Tensor Core** — Specialized matrix-math unit. The reason AI runs on GPUs and not CPUs.

**SM (Streaming Multiprocessor)** — The fundamental unit of compute. Contains CUDA cores, Tensor Cores, registers, shared memory.

**GPC (Graphics Processing Cluster)** — A group of SMs sharing some control hardware.

**TPC (Texture Processing Cluster)** — A pair of SMs. Holdover terminology from the chip's graphics ancestry; doesn't really matter for AI.

**HBM (High Bandwidth Memory)** — Stacked memory technology used in data-center GPUs. HBM3, HBM3e, HBM4 are generations.

**TDP (Thermal Design Power)** — The power the chip is rated to dissipate. 700W for H200 SXM, 1000W for B200. Drives the cooling and power requirements of the entire data center.

**SXM / NVL / PCIe** — Form factors. SXM is the high-performance socketed format (requires special server boards, often liquid-cooled). NVL/PCIe is the standard PCIe card format (air-coolable, drop-in).

**NVLink** — Nvidia's GPU-to-GPU interconnect. Currently 4th generation, 900 GB/s per GPU.

**Transformer Engine** — Hopper-onward feature that dynamically picks FP8 vs FP16 precision per operation to speed up transformer math.

**TMA (Tensor Memory Accelerator)** — Hardware unit for async memory transfers between HBM and on-SM shared memory.

**MIG (Multi-Instance GPU)** — Hardware-level partitioning of one physical GPU into multiple logical GPUs.

**FP8 / FP16 / FP32 / FP64 / FP4** — Floating-point precisions. The number is the total bits used to represent each value. Lower precision = smaller memory footprint and faster compute, at the cost of numerical accuracy.

**E4M3 / E5M2** — The two FP8 variants. Differ in how the 8 bits are split between exponent and mantissa.

**Sparsity** — Exploiting structured patterns of zeros in matrices to skip computation. The 2:4 pattern (2 zeros per 4 elements) is what Nvidia's hardware accelerates.

---

*Last updated: May 2026. Hardware specs and pricing are accurate to roughly Q1-Q2 2026. The Blackwell generation has largely supplanted Hopper at the very high end of new deployments, but the H200 remains the dominant chip in production AI infrastructure and the right anchor for understanding modern GPU architecture.*
