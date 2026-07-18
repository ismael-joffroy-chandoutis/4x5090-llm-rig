**English** · [Français](README.fr.md)

# 4x RTX 5090 Open-Air Rig for Local LLM Inference

> A filmmaker's journey building a 128 GB VRAM beast for local AI inference and fine-tuning on a prosumer budget.

## Why This Build

I'm a filmmaker working with local AI inference and creative research. Cloud GPU costs add up fast, and I needed a machine that could:

- Run **70B+ parameter models** quantized (GGUF Q4/Q5) at interactive speeds as a persistent agentic inference server
- Power **ComfyUI workflows** for Flux.2 and LTX Video 2 (which requires up to 27 GB VRAM for its text encoder alone)
- Serve as a local backend for **Claude Code and agentic pipelines** (Hermes Agent, Tier 2/3 open-source models) — always-on, no queue, no API cost
- Handle **multi-model inference** via vLLM or llama.cpp with tensor parallelism across all 4 GPUs
- Serve the whole **Tailscale network** as a private LLM/image/video API node

The answer: **4x RTX 5090** (128 GB total VRAM) on an open-air frame, built incrementally with parts I already had from previous builds and some smart sourcing.

### The Model Architecture Behind This Build

Running frontier models locally is largely a moot point — Claude 4, GPT-5, and Gemini Ultra aren't open-source and won't run on any consumer hardware. The right mental model is a three-tier stack:

```
Tier 1 — Frontier APIs (Claude / GPT / Gemini)
  → Complex reasoning, one-off creative tasks, production pipelines
  → Routed via LiteLLM — same endpoint, swap model with one config line

Tier 2 — Local open-source frontier-equivalent (70B+ dense, 200B+ MoE at Q4-Q5)
  → This rig. Daily agentic tasks, Hermes Agent workflows, research queries
  → Zero API cost, no rate limits, no data leaving the network, always warm

Tier 3 — Local distilled / small models (7-32B, turbo-quant)
  → Background agents, structured extraction, classification, summarization
  → Runs on one GPU, other cards free for Tier 2 or ComfyUI simultaneously
```

**Why not just use APIs for everything?** At the query volume of daily agentic pipelines (hundreds of requests/day for research, shot analysis, document processing), API costs accumulate fast. The rig's break-even against API pricing is under 12 months of regular use. After that, every query is electricity-cost only (~€0.001/query at €0.22/kWh).

**LiteLLM as the glue:** A single local LiteLLM proxy exposes all tiers under one OpenAI-compatible endpoint. Hermes Agent, Claude Code, or any other framework sends requests to `http://localhost:4000` and doesn't know — or care — whether the response came from a local Qwen3 or a remote Claude API.

### Nothing Replaces Anything — The Full Fallback Chain

These tiers are **complementary, not substitutes**. When one layer goes down, the next one catches you. The 4×5090 rig is the final layer that never goes down.

```
Claude Code / Anthropic API
  ↓ if down or rate-limited
AWS Bedrock (same Claude model, different infrastructure)
  ↓ if Bedrock is also unavailable
DeepSeek V4 API (or OpenRouter → any frontier model)
  ↓ if all commercial APIs fail or cost matters
Local open-source full model — Tier 2 (70B+ model, check leaderboard — this rig)
  ↓ if full model is busy or task is routine
Local distilled model — Tier 3 (7-14B turbo-quant, single GPU)
```

**LiteLLM handles this automatically** with `fallbacks` and `model_list` routing:

```yaml
# litellm_config.yaml
model_list:
  - model_name: primary
    litellm_params:
      model: anthropic/claude-opus-4-6
      fallbacks: [bedrock/claude-opus-4-6, deepseek/deepseek-chat, ollama/qwen3:72b]

  - model_name: agent-daily  # Hermes Agent background tasks
    litellm_params:
      model: ollama/qwen3:32b  # local Tier 2
      fallbacks: [deepseek/deepseek-chat]

  - model_name: agent-fast   # routine extraction/classification
    litellm_params:
      model: ollama/qwen3:8b  # local Tier 3, turbo-quant
```

The result: any agent framework hitting `http://localhost:4000` gets automatic failover without code changes. Claude Code down? Invisible. Anthropic and Bedrock both down simultaneously? The rig serves the same request locally. **The local hardware is the only node with 100% uptime in this chain.**

### Token Routing Strategy — Cost, Latency, Resilience

Never route all traffic to one provider. The optimal strategy distributes workload based on task complexity, volume, and latency requirements:

| Task type | Route to | Why |
|-----------|----------|-----|
| Single complex creative decision | Claude API (Tier 1) | Needs frontier reasoning, low volume |
| Research synthesis, document analysis | Local Tier 2 (70B+) | High volume, no data leaves the machine |
| Agent tool calls, structured extraction | Local Tier 3 (7-14B turbo-quant) | 100-200 tok/s, €0 marginal cost |
| Code generation (critical path) | Claude API → fallback local | Quality matters, fallback if API down |
| Background indexing, classification | Local only (Tier 3) | High volume, routine, no internet needed |
| Privacy-sensitive content | Context-dependent | Depends on data type, jurisdiction, and workflow — local for maximum control, but not a blanket rule |

**Offline-first principle:** any agent workflow that can run locally, should run locally. Internet goes down — the rig keeps working. API rate limit hit — the rig absorbs the overflow. The cloud is the upgrade path, not the dependency.

**Practical token distribution (example daily pipeline):**
- ~10% of queries go to Tier 1 (frontier API) — the expensive, irreplaceable ones
- ~40% go to Tier 2 (local 70B+) — quality close to frontier, zero API cost
- ~50% go to Tier 3 (local distilled) — fast, cheap, background tasks

Result: API costs drop by 80-90% vs. routing everything to Claude/GPT, while maintaining frontier quality for the tasks that actually need it.

> **On privacy:** "sensitive" is context-dependent — what matters is the data type, the jurisdiction, and who needs access. Local gives maximum control. API gives maximum capability. The right answer changes per project, per client, per country. Be open to both, and let the routing config be the policy.

---

## Full Parts List

| Component | Model | Qty | Unit Price (TTC) | Total (TTC) |
|-----------|-------|-----|-------------------|-------------|
| **CPU** | AMD Ryzen 9 9950X (4.3 / 5.7 GHz) | 1 | €649.95 | €649.95 |
| **Motherboard** | ASUS ProArt X870E-CREATOR WIFI | 1 | €549.95 | €549.95 |
| **RAM** | G.Skill Flare X5 Low Profile 96 GB (2×48 GB) DDR5-6000 CL30 | 2 | €1,299.95 | €2,599.90 |
| **GPU** | MSI GeForce RTX 5090 32G VANGUARD SOC | 4 | ~€4,100* | ~€16,400* |
| **Storage** | Samsung 9100 PRO 8 TB M.2 NVMe PCIe 5.0 | 1 | €1,249.95 | €1,249.95 |
| **CPU Cooler** | Noctua NH-D15 Chromax Black | 1 | €149.95 | €149.95 |
| **AIO (spare)** | Cooler Master MasterLiquid 240 Core II ARGB | 1 | €79.95 | €79.95 |
| **Fan** | Noctua NF-A12x25 PWM chromax.black.swap | 1 | €49.96 | €49.96 |
| **Fan** | Noctua NF-A12x15 PWM chromax.black.swap | 1 | €34.96 | €34.96 |
| **PSU (ATX)** | Corsair SF1000 SFX 80+ Platinum | 1 | €214.95 | €214.95 |
| **PSU (server)** | HP 1200W Server PSU (80+ Platinum) | 2 | ~€40 (used) | ~€80 |
| **PSU (octopus)** | Chinese 1600W breakout PSU (AliExpress) | 1 | ~€60 | ~€60 |
| **Case** | Open-air mining/GPU rig frame (6-8 GPU) | 1 | ~€120 | ~€120 |
| **Risers** | PCIe 5.0 risers (LINKUP AVA5 or equivalent) | 4 | ~€70 | ~€280 |
| **Fans (extra)** | Arctic P12/P14 PWM (for GPU airflow) | 6 | ~€10 | ~€60 |
| **Breakout boards** | HP PSU breakout + 16-pin cables | 2 | ~€35 | ~€70 |

**\*GPU street prices as of April 2026 (idealo.fr ~€4,000-4,300; MSRP was €2,699 at launch Nov 2025 but supply has not caught up with demand). Check [idealo.fr](https://www.idealo.fr/prix/205775927/msi-geforce-rtx-5090.html) for current pricing.**

### Total Estimated Cost

| Category | Cost |
|----------|------|
| Components (new, TTC) | ~€21,180 |
| Additional parts (risers, fans, cables) | ~€410 |
| **Total** | **~€21,590** |

*GPU prices are volatile — the RTX 5090 launched at ~€2,699 in November 2025 and has risen to €4,000-4,300 street price due to sustained demand. Prices may shift significantly over the next 6-12 months.*

Already owning the server PSUs, open-air frame, and octopus PSU saved ~€260.

---

## Architecture Deep Dive

### The AM5 PCIe Lane Problem

This is the most important thing to understand about this build, and what most guides gloss over.

**AM5 (Ryzen 9000 series) provides 28 PCIe 5.0 lanes from the CPU:**

| Allocation | Lanes | Generation |
|------------|-------|------------|
| GPU slots (PCIEX16_1 + _2) | 16 | PCIe 5.0 |
| M.2 NVMe (2 slots) | 8 | PCIe 5.0 |
| Chipset uplink | 4 | PCIe 5.0 |

The **X870E chipset** then adds 36-44 PCIe 4.0 lanes for secondary expansion slots, additional M.2, USB, SATA, etc. But the CPU itself provides only **16 lanes** for GPU slots total.

**What the ASUS ProArt X870E-CREATOR WIFI actually offers:**

- **Slot 1 (PCIEX16_1):** PCIe 5.0 x16 from CPU — full bandwidth (~64 GB/s bidirectional)
- **Slot 2 (PCIEX16_2):** PCIe 5.0 from CPU — shares the same 16-lane pool. When both slots are populated, they auto-bifurcate to **x8/x8**
- **Slot 3:** PCIe 4.0 x4 from chipset — usable but limited (~8 GB/s)

**Key insight:** Slots 1 and 2 share the CPU's 16 GPU lanes. You get either 1 GPU at x16 or 2 GPUs at x8/x8. The BIOS also supports bifurcation modes including x8/x4/x4 and even x4/x4/x4/x4 on a single slot (for specialized riser splitters).

### How to Actually Fit 4 GPUs

For a 4-GPU setup on this board, the realistic lane allocation is:

**Option A — Best balance (recommended):**
- Slot 1 + Slot 2: x8/x8 Gen5 → **GPU 1 + GPU 2** (via risers, since the cards are too thick to sit side by side)
- Slot 3 (chipset): x4 Gen4 → **GPU 3**
- **GPU 4:** Requires a bifurcation riser on Slot 1 or 2 (splitting x8 into x4/x4), or using a PCIe switch. Alternatively, some builders use the M.2 slot with an M.2-to-PCIe x4 adapter for the 4th card.

**Option B — Maximum density via bifurcation:**
- Slot 1: x4/x4/x4/x4 bifurcation → **4 GPUs at x4 Gen5** each (~16 GB/s bidirectional)
- This requires a quad-bifurcation riser card and BIOS support

**Option C — Practical compromise (most common in real builds):**
- Slot 1: x8 Gen5 → **GPU 1**
- Slot 2: x8 Gen5 → **GPU 2**
- Slot 3 (chipset): x4 Gen4 → **GPU 3**
- **GPU 4:** on a riser via a bifurcation split of Slot 1 or Slot 2 (x4 Gen5), accepting one GPU drops to x4

All options work for LLM inference. Here's why:

### Why This Barely Matters for LLM Inference

PCIe bandwidth is mostly irrelevant for inference workloads. Here's why:

1. **Model loading is a one-time operation.** You load 70B weights into VRAM once. Even at x4 Gen4 (~8 GB/s), loading 32 GB of weights takes ~4 seconds instead of ~0.5 seconds. You do this once.

2. **Token generation is GPU-compute bound**, not PCIe-transfer bound. The bottleneck is matrix multiplication inside the GPU, not data moving over the bus.

3. **Multi-GPU tensor parallelism** (vLLM, llama.cpp) does communicate between GPUs during inference, but the volume is small compared to training. On AM5 without NVLink, inter-GPU communication goes through CPU/PCIe — slower than NVLink, but sufficient for inference at dozens of tokens/second.

4. **Benchmarks show 0-4% difference** between Gen5 x16 and Gen4 x4 for LLM inference throughput on single-GPU. For multi-GPU, the gap can widen to 5-15% depending on model size and parallelism strategy, but it's rarely the bottleneck.

**The Threadripper Alternative:** If PCIe lanes genuinely become a bottleneck (e.g., heavy training workloads with large gradient syncs), the proper platform is AMD Threadripper PRO 7000 with WRX90 (128 PCIe 5.0 lanes — 32 per GPU for 4 cards, 8-channel DDR5 up to 2 TB). Confirmed quad-5090 builds exist on Threadripper with full x16 per GPU and custom liquid cooling. But the CPU alone costs ~€3,000+ and boards are ~€1,000+. For inference-focused builds, AM5 is the pragmatic choice — you sacrifice lane width but keep 95%+ of the inference throughput at a fraction of the platform cost.

---

## Will All 4 GPUs Run at the Same Speed?

**Short answer: no.** This is the most common question about multi-GPU on AM5, and it deserves a straight answer.

### The Asymmetry

On the ProArt X870E, the 4 GPUs get unequal PCIe bandwidth:

| GPU | Slot | PCIe Link | Bandwidth | Relative Speed |
|-----|------|-----------|-----------|----------------|
| GPU 1 | CPU Slot 1 | x8 Gen5 | ~32 GB/s | Fast |
| GPU 2 | CPU Slot 2 | x8 Gen5 | ~32 GB/s | Fast |
| GPU 3 | Chipset Slot | x4 Gen4 | ~8 GB/s | Slower |
| GPU 4 | Bifurcation/adapter | x4 Gen5 or Gen4 | ~8-16 GB/s | Slower |

The GPUs themselves (CUDA cores, GDDR7 VRAM, clock speeds) are identical. It's the **pipe between GPU and CPU** that varies.

### When Does It Matter?

**Single-model inference (Ollama, one model on one GPU):** Not at all. The model loads into VRAM once, then it's pure GPU-internal compute. A GPU at x4 Gen4 generates tokens at the exact same speed as one at x8 Gen5.

**Multi-GPU tensor parallelism (vLLM, llama.cpp with 4 GPUs on one model):** It matters. GPUs exchange activation data at every token. The slowest link (x4 Gen4 at ~8 GB/s) becomes the bottleneck because faster GPUs wait for it. Estimated impact: **5-15% lower throughput** vs. 4 GPUs at equal bandwidth.

**Multi-model serving (different models on different GPUs):** Doesn't matter. Each GPU works independently. Put your most-used model on a fast-lane GPU if you want optimal single-model latency.

### Three Ways to Equalize

**Option 1: Threadripper (the real answer) — +€3,500-4,000**

AMD Threadripper PRO 7975WX + WRX90 board gives 128 PCIe 5.0 lanes. Each GPU gets a full x16 slot. No bifurcation, no compromises, no asymmetry. This is what commercial AI workstations (BIZON, Lambda, Puget Systems) use. If you're spending €8,800 on GPUs, the extra €3,500 for Threadripper is arguably worth it for a properly balanced system.

| Component | Model | Est. Price |
|-----------|-------|------------|
| CPU | AMD Threadripper PRO 7975WX (32C/64T) | ~€3,500 |
| Motherboard | ASUS WRX90E SAGE or Gigabyte WRX90 AERO D | ~€1,000-1,200 |
| RAM | Your DDR5 kits work (4 of 8 channels populated) | €0 (reuse) |

**Option 2: Force all slots to Gen4 x4 (free, all equal)**

In BIOS, force every PCIe slot to Gen4. All 4 GPUs now run at the same ~8 GB/s. You lose absolute bandwidth but gain **symmetry** — no GPU waits for a slower neighbor during tensor parallelism. For inference, the ~8 GB/s pipe is rarely the bottleneck anyway.

This is the cheapest and simplest equalization strategy. Counterintuitive but effective.

**Option 3: Software compensation with tensor-split (free, smart)**

llama.cpp and vLLM both support weighted tensor splitting. Give more model layers to fast-lane GPUs and fewer to slow-lane GPUs:

```bash
# llama.cpp: weight distribution across 4 GPUs
# GPUs 1-2 (x8 Gen5) get more layers, GPUs 3-4 (x4) get fewer
./llama-server \
  -m models/llama-3.1-70b-Q5_K_M.gguf \
  --n-gpu-layers 99 \
  --tensor-split 30,30,20,20

# vLLM: uses tensor parallelism automatically
# Less fine-grained control, but handles asymmetry reasonably well
vllm serve <current-70B-model-from-leaderboard> \
  --tensor-parallel-size 4 \
  --gpu-memory-utilization 0.90
```

The slow GPU receives less work, finishes at the same time as the fast ones, and nobody waits. This is the most pragmatic approach on AM5.

### Comparison Table

| Approach | Extra Cost | All 4 Equal? | Overall Throughput |
|----------|-----------|--------------|-------------------|
| Threadripper WRX90 | +€3,500-4,000 | Yes (4×x16 Gen5) | 100% |
| AM5 + force Gen4 x4 | €0 | Yes (4×x4 Gen4) | ~92-95% |
| AM5 + tensor-split tuning | €0 | Compensated | ~90-95% |
| AM5 + do nothing | €0 | No | ~85-95% |

### My Recommendation

For **inference** (my primary use case): stay on AM5, use `--tensor-split` to compensate. The 5-10% difference vs. Threadripper doesn't justify €3,500+.

For **training** (LoRA/QLoRA is fine, full fine-tuning is not): if you ever need to do serious multi-GPU training with large gradient syncs, sell the AM5 combo and upgrade to Threadripper. The PCIe asymmetry hurts much more during training than inference.

---

## PCIe 5.0 Risers: The Full Story

### Why PCIe 5.0 Risers Have a Bad Reputation

PCIe 5.0 runs at **32 GT/s** (gigatransfers per second), double Gen4's 16 GT/s. At these frequencies:

- **Signal integrity degrades faster** with cable length, bends, and poor shielding
- **Cheap risers introduce noise** that causes link negotiation failures, crashes under load, or automatic downgrade to Gen4/Gen3
- The **RTX 5090's multi-PCB design** (especially Founders Edition) already acts as an internal riser of sorts, compounding signal integrity challenges

Reviewers like der8auer, Igor's Lab, and Hardware Canucks documented instabilities with budget risers on RTX 50-series cards, especially when BIOS is set to "Auto" PCIe generation.

### But It's Not a Fatality

With **quality PCIe 5.0 risers**, it works fine. The key factors:

| Factor | Good | Bad |
|--------|------|-----|
| Cable length | 15-30 cm | >40 cm |
| Shielding | Multi-layer, full coverage | Thin or partial |
| Connectors | Gold-plated, reinforced | Loose or cheap |
| Bending | Gentle curves only | Sharp folds, crimped |
| Brand | LINKUP, Thermaltake, ADT-Link | No-name AliExpress |

### My Recommendation

**Primary choice:** LINKUP AVA5 PCIe 5.0 (20-30 cm versions). As of 2026, these are the most widely validated for RTX 5090 builds. They maintain full bandwidth (128 GB/s on 3DMark tests) even with moderate bends. Price: ~€55-80 each.

**Budget alternative:** Quality PCIe 4.0 risers + force Gen4 in BIOS. Performance loss: 1-3% in inference. Stability gain: significant. Price: ~€25-40 each. Many multi-5090 builds run this way without issues.

**Fallback strategy:** Start with Gen5 risers, set BIOS to Auto. If you see instabilities (GPU not detected, crashes under sustained load, GPU-Z showing unexpected Gen3/4 link), force Gen4 in BIOS. You lose almost nothing for LLM workloads.

### Validation Protocol

After assembling, verify with:

```bash
# Check PCIe link speed and width
nvidia-smi -q | grep -A 5 "PCI"

# Stress test: sustained inference for 24h
# (use your actual workload, not synthetic benchmarks)
ollama run <your-70B-model>:q4_K_M

# Monitor thermals during stress
watch -n 1 nvidia-smi
```

Check that GPU-Z shows the expected generation (Gen5 x8 or Gen4 x16/x8) and that it doesn't fluctuate during load.

---

## Power Delivery: Multi-PSU Strategy

### Power Budget

| Component | Peak Draw |
|-----------|-----------|
| RTX 5090 Vanguard SOC (×4) | 4 × 575W = 2,300W |
| Ryzen 9 9950X (PBO) | ~200W |
| RAM, SSDs, fans, board | ~80W |
| **Total peak** | **~2,580W** |
| **With transient spikes** | **~2,800-3,000W** |

A single ATX PSU cannot handle this. You need a multi-PSU setup.

### My PSU Configuration

```
PSU 1: HP 1200W Server (80+ Platinum)
  → Motherboard 24-pin (via ATX breakout)
  → CPU 8-pin
  → GPU 1 (16-pin native)
  → GPU 2 (16-pin native)

PSU 2: HP 1200W Server (80+ Platinum)
  → GPU 3 (16-pin native)
  → GPU 4 (16-pin native)

PSU 3: Corsair SF1000 (backup / headroom)
  → Available if load balancing needed
  → Or used for fans + peripherals
```

### HP Server PSU Notes

HP ProLiant 1200W units (DPS-1200FB or HSTNS-PL11) are **excellent** for multi-GPU builds:

- **80+ Platinum efficiency** — designed for 24/7 datacenter operation
- **12V single rail at 100A** — clean, high-current output
- **Dirt cheap** on the used market (~€30-50 each)
- Widely used in mining and AI rig builds since 2017

**What you need to make them work:**

- **Breakout boards** (~€15-30 on AliExpress/Amazon) that convert the server PSU connector into standard ATX/PCIe cables
- **Quality 16-pin (12V-2×6) cables** for the RTX 5090. Do NOT use cheap Molex-to-16-pin adapters — these can melt under 575W sustained load. Get cables rated for the current.
- A way to **sync startup** between PSUs (some breakout boards handle this, or use a jumper wire on the ATX 24-pin green wire)

### The Octopus PSU (AliExpress 1600W)

These "pieuvre" PSUs are essentially server PSU breakout boards with built-in cables, often sold for mining. They work, but:

- **Quality varies wildly** — test voltage stability under load before trusting it with 5090s
- Use as **supplementary** power, not primary
- Good for powering 1-2 GPUs while the HP units handle the rest

### Electrical Infrastructure

**Critical:** 4×5090 at full load draws ~2.5-3 kW from the wall. Standard European 16A / 230V circuits provide 3.68 kW max — you're close to the limit.

Recommendations:
- **Dedicated 20A or 32A circuit** to the rig
- Never share the circuit with space heaters, kettles, or other high-draw appliances
- A **UPS is optional but smart** — not for runtime, but for clean shutdown protection
- Consider **undervolting the GPUs** (see below) to reduce draw by 20-30% with minimal performance loss

---

## Cooling: Open-Air vs. Closed Case

### Why Open-Air Wins for This Build

The MSI Vanguard SOC is a **quad-slot card** (~76 mm thick, 336 mm long). Four of these in a closed case, even a full tower, creates a thermal nightmare:

- Cards would be physically touching or overlapping in standard ATX cases
- No airflow between GPUs
- Ambient case temperature rises to 50-60°C under sustained load
- GPU thermal throttling kicks in, reducing inference speed

**Open-air advantages:**

- Space each GPU 2-3 slot widths apart
- Direct ambient air access on all sides
- Easy to add directional fans between cards
- Trivial maintenance and card swaps
- 10-15°C cooler GPU temps vs. closed case (measured across similar builds)

### The Crypto Mining Approach (Closed Case + Industrial Fans)

Some miners use enclosed cases with **high-CFM industrial blowers** (200-300 mm, 100+ CFM) creating massive positive pressure. This works for:

- Dust-heavy environments (warehouses, garages)
- Forced hot-air exhaust through ducting
- 24/7/365 unattended operation

For a **home/studio LLM inference rig**, open-air is simpler, cheaper, and quieter. The industrial blower approach adds complexity and noise for marginal benefit in a clean indoor environment.

### My Cooling Setup

```
[INTAKE FANS]  →  [GPU 1]  →  [FAN]  →  [GPU 2]  →  [FAN]  →  [GPU 3]  →  [FAN]  →  [GPU 4]  →  [EXHAUST]

Bottom of frame: Motherboard + CPU (Noctua NH-D15)
Between each GPU: 1-2× 120mm fans (Arctic P12 or Noctua NF-A12x25)
Side/top: Additional 140mm fans for general airflow
```

**Tip:** Mount GPUs horizontally (flat) if your frame allows it. This prevents GPU sag and keeps the heatsinks working optimally (heat rises naturally).

### Undervolting: Free Performance

The RTX 5090 Vanguard SOC ships with aggressive factory overclock curves. Undervolting reduces power draw and heat significantly:

```bash
# Example: cap power to 80% (saves ~115W per card)
nvidia-smi -i 0 -pl 460  # GPU 0
nvidia-smi -i 1 -pl 460  # GPU 1
nvidia-smi -i 2 -pl 460  # GPU 2
nvidia-smi -i 3 -pl 460  # GPU 3
```

Expected results at 80% power limit:
- Power draw: ~460W vs. 575W per card (saves 460W total across 4 GPUs)
- Performance loss: 3-8% in inference (barely noticeable in tokens/s)
- Temperature drop: 5-10°C per card
- Total rig draw drops from ~2,600W to ~2,100W — much friendlier to your circuit

For long inference runs, this is the **single best optimization** you can make.

---

## BIOS Configuration

### Essential Settings

| Setting | Value | Why |
|---------|-------|-----|
| PCIe Generation | Gen 4 (or Auto → Gen 4 fallback) | Stability over bandwidth |
| XMP / EXPO | Enabled (DDR5-6000) | Use your RAM's rated speed |
| Resizable BAR | Enabled | Allows GPU to access full VRAM mapping |
| Above 4G Decoding | Enabled | Required for multi-GPU + large VRAM |
| SR-IOV | Disabled (unless using vGPU) | Simplicity |
| CSM | Disabled | UEFI boot, modern OS |
| PCIe Bifurcation | x8x8 on Slot 1 (if available) | Allows 2 GPUs on CPU lanes |
| C-States | Enabled | Power saving when idle |
| PBO | Enabled, eco mode optional | Balance CPU perf vs. heat |

### RAM Configuration

With 4×48 GB (192 GB total), all four DIMM slots are populated:

- Slots **A2 + B2** first (dual-channel primary) — your first kit
- Slots **A1 + B1** second — your second kit
- XMP/EXPO profile should detect DDR5-6000 CL30 automatically
- If instability occurs with 4 DIMMs at 6000 MHz, drop to 5600 MHz — 4-DIMM configs are harder to run at high frequencies on AM5

**Why 192 GB RAM matters:** For LLM inference, system RAM is used for:
- Model loading buffer before GPU transfer
- KV cache overflow (when VRAM is full)
- CPU offloading of model layers (llama.cpp `--n-gpu-layers` partial offload)
- Running ComfyUI, vLLM server, monitoring, and OS simultaneously

192 GB gives comfortable headroom for all of this. 96 GB would work but gets tight with multiple large models loaded.

---

## Software Stack

### Core Inference

```bash
# Ollama (quickest to start — auto-detects all 4 GPUs)
ollama serve
ollama run qwen3:32b          # fits in 1 GPU
ollama run qwen3:72b-q4_K_M   # spans 2-3 GPUs automatically

# vLLM (best for concurrent requests + tensor parallelism)
vllm serve <current-Tier2-model-from-leaderboard> \
  --tensor-parallel-size 4 \
  --gpu-memory-utilization 0.90

# llama.cpp (best flexibility, GGUF format, partial CPU offload)
./llama-server \
  -m models/qwen3-70b-Q5_K_M.gguf \
  --n-gpu-layers 99 \
  --tensor-split 30,30,20,20  # compensate AM5 lane asymmetry
```

### LiteLLM — Unified API Proxy (essential for agent frameworks)

LiteLLM exposes all your local models behind a single OpenAI-compatible endpoint.
Any agent framework (Hermes Agent, Claude Code, LangChain, CrewAI) talks to one URL.

```bash
pip install litellm
litellm --model ollama/qwen3:32b --port 4000
# or point at vLLM:
litellm --model openai/qwen3-70b --api-base http://localhost:8000/v1 --port 4000
# now any app hits http://localhost:4000 regardless of backend
```

### Hermes Agent — Agentic Layer

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (NousResearch, released Feb 2026) is an autonomous agent framework, not just a model. It runs on top of any local LLM backend and handles tool calling, memory, and multi-step reasoning:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
# point it at your local vLLM/Ollama via LiteLLM:
hermes config set provider openai
hermes config set base_url http://localhost:4000
hermes config set model qwen3:32b
hermes run "analyze all images in ~/shots/ and create a shot list"
```

Key features: 40+ built-in tools, subagents for parallel workstreams, session memory, 30-min socket timeout for large-context local inference. The Hermes fine-tuned models underneath are trained for structured function calling (`<tool_call>` / `<tool_response>` format) — more reliable than base instruction models for agentic tasks.

### What Can 128 GB VRAM Run?

> **Models change faster than hardware. Don't use this section as a shopping list.**
> Check these instead — they're updated in real time:
> - [openrouter.ai/rankings](https://openrouter.ai/rankings) — usage-weighted, what people actually run
> - [lmarena.ai](https://lmarena.ai) — human preference ELO
> - [artificialanalysis.ai](https://artificialanalysis.ai) — benchmark aggregator

### What 128 GB VRAM Unlocks — By Tier

The hardware defines what tiers are reachable. The specific models in each tier change constantly.

| Tier | VRAM needed | What fits | Typical speed | Examples (April 2026 — verify current) |
|------|------------|-----------|--------------|----------------------------------------|
| **Tier 0 — Fast agents** | 4-18 GB (1 GPU) | 7B-32B dense or MoE | 50-200+ tok/s | Qwen3 8B, Gemma 4 9B, DeepSeek-R1 distill 7B |
| **Tier 1 — Quality agents** | 18-40 GB (1-2 GPUs) | 30B-32B dense | 30-80 tok/s | Qwen3 32B, Qwen3.6 Plus variants, MiniMax distills |
| **Tier 2 — Frontier-equivalent** | 40-80 GB (2-3 GPUs) | 70B dense, 200B+ MoE | 15-50 tok/s | Top open-weight models of the moment (check leaderboard) |
| **Tier 3 — Max capacity** | 80-128 GB (3-4 GPUs) | 200B+ MoE at better quant | 10-30 tok/s | Kimi K2.5, GLM-5.1, MiniMax M2.7 at Q3-Q4 |
| **Overflow (CPU)** | 128 GB VRAM + 192 GB DDR5 | 400B-670B partial offload | 5-15 tok/s | DeepSeek-V3 full, future 400B+ models |
| **Video — 1 dedicated GPU** | 32-40 GB | LTX Video 2.3, WanVideo 2.2 14B | — | Fixed by model architecture, less volatile |

**Rule of thumb:** the model leaderboard rotates every 4-8 weeks. The tier structure doesn't. A Tier 2 card today runs whatever the Tier 2 model is in 18 months.


**Why 192 GB DDR5 matters for the large models:** When a 200B+ MoE or 400B+ model spills beyond 128 GB VRAM, the overflow layers run on CPU RAM. With 192 GB DDR5-6000, the fallback is fast enough (~8-15 tok/s) to be usable. On a machine with 32-64 GB RAM, the same overflow drops to 2-3 tok/s.

The sweet spot is **70B Q5-Q8** models — they run entirely in VRAM across 2-4 GPUs with fast inference.

### Image/Video Generation (ComfyUI)

**ComfyUI v0.16.1+** — Dynamic VRAM enabled by default. Update before anything else.

**Current models (April 2026):**

| Model | Type | VRAM (FP8/quant) | Notes |
|-------|------|-----------------|-------|
| Flux.2 Dev/Schnell | Image | ~12-16 GB | NVFP4 = 3× faster than FP16 on 5090 |
| Flux.2 Pro | Image | ~8-12 GB | Faster Flux variant, step-distilled |
| WanVideo 2.2 (14B) | Video | ~40 GB FP8, ~20 GB (720p) | Best open-source video motion quality |
| WanVideo 2.2 (5B) | Video | ~8-16 GB | Faster, lighter, still good |
| LTX Video 2.3 (22B) | Video + Audio | ~32 GB+ (official), ~12-24 GB FP8 | Generates audio + video simultaneously |

```bash
# 4×5090 multi-workflow layout:
#   GPU 0: WanVideo 2.2 14B (FP8, ~40 GB) — dedicated video card
#   GPU 1-2: Tier 2 model inference (70B tensor parallel, ~55 GB)
#   GPU 3: Flux.2 image batches + SD 3.5 / LoRA training
#
# ComfyUI selects GPU via CUDA_VISIBLE_DEVICES:
CUDA_VISIBLE_DEVICES=0 python -m main --port 8188  # video node
CUDA_VISIBLE_DEVICES=3 python -m main --port 8189  # image node
```

**LTX Video 2.3 note:** The 22B model officially needs 32GB+. A single RTX 5090 (32GB) runs it at FP8 with `--reserve-vram 5`. With the [VRAM management nodes](https://github.com/RandomInternetPreson/ComfyUI_LTX-2_VRAM_Memory_Management), peak consumption drops ~10×, enabling 800+ frames at 1920×1088 on one card.

**WanVideo 2.2 note:** The 14B model at FP8 needs ~40 GB — dedicate GPU 0 to it. The 5B variant fits in one card (24-32 GB) with good results.

### Monitoring

```bash
# Real-time GPU stats
watch -n 1 nvidia-smi

# Detailed power/temp monitoring
nvidia-smi dmon -s pucvmet -d 5

# Per-GPU utilization for multi-model serving
gpustat --watch
```

---

## Network Integration

This rig joins my existing Tailscale mesh:

| Machine | Role | Tailscale IP |
|---------|------|-------------|
| MacBook Air M3 | Primary workstation | 100.x.x.1 |
| Mac Mini M4 | Server (Paris) | 100.x.x.2 |
| **4×5090 Rig** | GPU compute | TBD |
| PC Windows (existing) | GPU + ComfyUI | 100.x.x.3 |

Ollama or vLLM exposes an API on the Tailscale network, accessible from any machine:

```bash
# From MacBook, query the 4×5090 rig
curl http://<rig-tailscale-ip>:11434/api/generate \
  -d '{"model": "your-70b-model", "prompt": "..."}'
```

---

---

## Portability — Taking the Rig On Location

An open-air mining frame is not designed to travel. But it can, with the right approach.

### Weight Reality Check

| Component | Approx. weight |
|-----------|---------------|
| 4× RTX 5090 Vanguard SOC | ~4× 1.8 kg = ~7.2 kg |
| Open-air frame (aluminum/steel) | ~3-5 kg |
| Motherboard + CPU + cooler | ~2 kg |
| 2× HP server PSUs | ~4 kg |
| Corsair SFX PSU | ~0.8 kg |
| RAM + SSDs + cables | ~1 kg |
| **Total** | **~18-20 kg** |

Not luggage. Transport, not travel.

### Option A — Rack Case (Most Professional)

Replace the open-air frame with a **4U rackmount chassis** (Tupavco TP1846 or Hydra III), then fit it into a **rackmount Pelican/SKB transport case** (SKB R914U or equivalent). This is the pro audio/video approach — same system used for live broadcast GPU servers.

- Pros: fully enclosed, protected, stackable, airline-checkable as oversized freight
- Cons: ~€400-600 for the rack + case, slightly worse airflow than open-air
- The GPU cards still need individual padding/securing inside the 4U chassis

### Option B — Disassemble + Carry Separately

For occasional transport (Paris → LA, residencies):
1. Remove 4 GPUs → each in anti-static bag + foam sleeve → in carry-on or padded Pelican 1510
2. Ship the frame + motherboard + PSUs (non-fragile, can ship as freight)
3. Reassemble on location (~45 min)

This is how most filmmakers with GPU rigs actually travel. The GPUs are the irreplaceable/expensive parts — they go with you. The rest ships.

### Option C — Remote Access (Best Option for Most Trips)

Leave the rig in Paris. Access it via **Tailscale** from anywhere with internet:

```bash
# From anywhere: ssh into the rig, run inference remotely
ssh user@<rig-tailscale-ip>
# or hit the LiteLLM endpoint directly:
curl http://<rig-tailscale-ip>:4000/v1/chat/completions ...
```

Most production use cases don't require physical proximity. The rig serves the whole network. For a shoot in LA, the Paris rig handles agent tasks and model inference overnight — you wake up with results. This is the real advantage of the Tailscale integration.

### Power Compatibility (France ↔ USA / International)

| PSU | Input voltage | International use |
|-----|---------------|------------------|
| Corsair SF1000 | 100-240V auto | Works everywhere |
| HP 1200W Server PSU | 100-240V auto | Works everywhere |
| Octopus breakout PSU | Typically 110-240V — **check label** | May need transformer |

The HP server PSUs and the Corsair SF are universal. Verify the AliExpress octopus PSU before plugging in abroad.

---

## Remaining Budget (What I Still Need to Buy)

Already owned: open-air frame, HP server PSUs ×2, octopus PSU, Corsair SF1000, extra Noctua fans.

| Item | Est. Cost | Priority |
|------|-----------|----------|
| 4× PCIe 5.0 risers (LINKUP AVA5 20-30cm) | €280-350 | Critical |
| 4-6× Arctic P12 PWM fans | €50-70 | High |
| 2× HP breakout boards + 16-pin cables | €60-80 | High |
| Misc cables, zip ties, standoffs | €20-30 | Medium |
| **Total remaining** | **€410-530** | |

---

## Honest Caveats

### What I Would Change If Starting Over

1. **Threadripper PRO 7000 WRX90** instead of AM5. The PCIe lane situation on AM5 is workable but not ideal for 4 GPUs. Threadripper gives 128 PCIe 5.0 lanes (32 per GPU, so 4 cards at full x16 each without bifurcation hacks), 8-channel DDR5 (up to 2 TB), and is designed for exactly this use case. Confirmed multi-5090 builds exist on Threadripper with custom liquid cooling. Cost difference: ~€3,000-4,000 more for CPU+board. Worth it if this is a production machine or if you plan to train (not just infer).

2. **ECC RAM** for long inference jobs. Consumer DDR5 can have rare bit flips during 24+ hour runs. ECC catches these. Available on Threadripper; technically supported but not validated on most AM5 boards.

3. **Dedicated 32A circuit** from the start. I'm running close to the limit of a standard European 16A / 230V circuit (3.68 kW max). With undervolting it's fine, but at full power all 4 GPUs can trip the breaker.

4. **Reference-design / blower-style GPUs** instead of thick open-air cooler cards. The Vanguard SOC quad-slot design means physical spacing is a challenge even on open-air frames. Reference PCB cards are thinner, and blower-style coolers exhaust heat directionally rather than radiating into adjacent cards. Some 4-GPU builders specifically seek reference cards with waterblock compatibility for this reason.

### Known Risks

- **PCIe lane allocation is the #1 complexity.** AM5's 16 GPU lanes across 2 slots means you're juggling bifurcation, chipset slots, and possibly PCIe adapters to seat 4 cards. Test each GPU individually before the full 4-card config. If a GPU fails to initialize, it's likely a lane/slot issue, not a dead card.
- **PCIe riser stability** is the #2 risk. Budget risers + Gen5 + RTX 5090 = potential for frustrating intermittent crashes. Spend the money on quality risers.
- **HP server PSUs are loud.** The stock 40mm fans ramp to jet-engine levels under load. Common mod: replace stock fans with Noctua 40mm (NF-A4x20 PWM) for dramatic noise reduction while maintaining safe airflow.
- **4-DIMM DDR5-6000 on AM5** can be finicky. The IMC (integrated memory controller) on Zen 5 works harder with 4 populated DIMMs. If you get memory errors or POST failures, drop to 5600 MHz and tighten timings.
- **Dust** in open-air setups. Monthly cleaning with compressed air is mandatory. Keep the rig in a clean, ventilated room away from pets and fabric.
- **M.2 slot sharing with GPU lanes.** On the ProArt X870E, some M.2 slots share bandwidth with PCIe GPU slots. Check the manual for which M.2 port uses CPU lanes vs. chipset lanes. Prioritize chipset M.2 for your second SSD to avoid stealing GPU bandwidth.

### What This Build Is NOT For

- **Large-scale training** (pre-training, full fine-tuning of 70B+ models) — you need NVLink or InfiniBand for efficient gradient sync. This build is inference and LoRA/QLoRA focused.
- **Silence** — 4×5090 under load + server PSUs + extra fans = significant noise. This is a utility machine, not a living room PC.
- **Plug-and-play** — multi-PSU, risers, and open-air means more assembly, troubleshooting, and maintenance than a standard PC.

---

## Build Log / Assembly Order

1. Mount motherboard on open-air frame
2. Install CPU + NH-D15 cooler
3. Install 4×48 GB RAM (all slots)
4. Install 2× M.2 SSDs
5. Connect risers to PCIe slots (test one GPU first before all four)
6. Mount GPU 1 directly or on short riser — boot, install OS + drivers
7. Add GPUs 2-4 one at a time, testing stability after each
8. Connect PSUs (HP units via breakout boards)
9. BIOS: enable Above 4G, Resizable BAR, set PCIe gen, XMP
10. Install Ubuntu Server 24.04 or similar (headless, SSH)
11. Install NVIDIA drivers + CUDA toolkit
12. Install Ollama / vLLM / llama.cpp
13. Run 24h stress test with target model
14. Undervolt, verify temps, final tune

**Golden rule:** Add one GPU at a time. If something is unstable with 4 GPUs, remove GPUs until stable, then add back. Isolate the problem before debugging.

---

## Expected Performance

Based on comparable 4×5090 builds:

| Workload | Expected Performance |
|----------|---------------------|
| 70B Q4_K_M (Tier 2, single GPU — partial fit, some KV spill) | ~25-35 tok/s |
| 70B Q4_K_M (Tier 2, 2-GPU tensor parallel) | ~50-70 tok/s |
| 70B Q4_K_M (Tier 2, 4-GPU tensor parallel) | ~80-120 tok/s |
| 70B Q8 (Tier 2, 4-GPU — no quant loss) | ~50-70 tok/s |
| 7-14B Q4 (Tier 3, single GPU) | 150-250+ tok/s |
| Flux.2 image generation (NVFP4 on 5090) | ~1-3 sec/image |
| WanVideo 2.2 14B FP8 (dedicated GPU) | varies by resolution |
| LTX Video 2.3 22B FP8 (single GPU + VRAM mgmt) | varies by length |
| LoRA fine-tuning (QLoRA, 70B, 4-GPU) | ~4-6× vs. single GPU |

These are rough estimates. Actual numbers depend on quantization, batch size, prompt length, KV cache, and software version. For current per-model benchmarks check [artificialanalysis.ai](https://artificialanalysis.ai).

---

---

## All Alternatives Compared

> TL;DR: 4×RTX 5090 on AM5 is the only prosumer config that combines 128 GB CUDA VRAM, full CUDA ecosystem (ComfyUI/Flux.2/LTX-2/vLLM/PyTorch), and a break-even against cloud within 18 months. Every alternative fails on at least one of these criteria.

### The Full Benchmark Matrix

| Config | VRAM | BW/card | 70B Q4 fits? | CUDA | ComfyUI Flux.2 | LTX Video 2 | Est. GPU cost |
|--------|------|---------|-------------|------|----------------|-------------|---------------|
| RTX 3090 × 1 | 24 GB | 936 GB/s | No | Yes | Yes | Partial (offload) | ~€500 used |
| RTX 3090 × 4 | 96 GB | 936 GB/s | Yes | Yes | Yes | Yes (1 card) | ~€2,000 used |
| RTX 4090 × 1 | 24 GB | 1,008 GB/s | No | Yes | Yes | Partial (offload) | ~€1,500 |
| RTX 4090 × 2 | 48 GB | 1,008 GB/s | Tight | Yes | Yes | Yes (1 card) | ~€3,000 |
| RTX 4090 × 4 | 96 GB | 1,008 GB/s | Yes | Yes | Yes | Yes | ~€6,000 |
| RTX 5090 × 1 | 32 GB | 1,792 GB/s | Partial (Q3 only) | Yes | Yes | Yes (tight) | ~€4,100 |
| RTX 5090 × 2 | 64 GB | 1,792 GB/s | Yes (comfortable) | Yes | Yes | Yes | ~€8,200 |
| **RTX 5090 × 4 AM5 (this build)** | **128 GB** | **1,792 GB/s** | **Yes + headroom** | **Yes** | **Yes** | **Yes (dedicated GPU)** | **~€8,800** |
| RTX 5090 × 4 Threadripper | 128 GB | 1,792 GB/s | Yes + headroom | Yes | Yes | Yes | ~€8,800 GPU + €4,500 platform |
| Mac Mini M4 Pro (64 GB) | 64 GB unified | 273 GB/s | Partial (MLX only) | No | No (Metal only) | No (Metal only) | ~€1,800 |
| Mac Studio M4 Max (128 GB) | 128 GB unified | 410 GB/s | Yes (MLX only) | No | No | No | ~€3,800 |
| Mac Studio M4 Ultra (192 GB) | 192 GB unified | 819 GB/s | Yes + 405B too | No | No | No | ~€5,999 |
| Mac Mini M4 + eGPU 5090 | — | — | No (see below) | No | No | No | ~€4,000+ wasted |

**Token/s on Qwen3 70B Q4_K_M (multi-GPU, tensor parallel):**

| Config | Approx. tok/s | Notes |
|--------|---------------|-------|
| 4× RTX 3090 | ~35-45 | PCIe 3.0 bandwidth limits inter-GPU sync |
| 2× RTX 4090 | ~35-50 | 48 GB fits, minimal KV cache headroom |
| 4× RTX 4090 | ~55-75 | 96 GB, comfortable; aging bandwidth |
| 2× RTX 5090 | ~55-75 | 64 GB, comfortable |
| **4× RTX 5090 (this build)** | **~80-120** | **128 GB, full KV cache, optimal tensor split** |
| Mac Studio M4 Ultra | ~30-40 | MLX only, no CUDA ecosystem |

### Why Not Each Alternative

**4× RTX 3090 (cheapest multi-GPU option, ~€2,000 total)**

On paper this covers 70B inference. In practice:

- **PCIe 3.0** throughout — inter-GPU communication is twice as slow as Gen4, and 4× slower than Gen5. Every tensor-parallel token sync pays that penalty
- **936 GB/s bandwidth per card** vs. 1,792 GB/s on the 5090. Inference is pure memory bandwidth. You're paying half the speed
- **LTX Video 2** barely runs with aggressive memory offloading. The Gemma 3 12B text encoder needs ~24-27 GB alone; on a 24 GB card, you're offloading the encoder to RAM every generation
- **CUDA feature lag** — NVIDIA is deprioritizing Ampere (30-series) for new cuDNN, Flash Attention 3, and NVFP4 kernels. vLLM and ComfyUI optimizations target Ada (40-series) and Blackwell (50-series) first
- **Same power draw for half the performance** — a 3090 pulls ~350W; 4× = 1,400W for ~35-45 tok/s vs. the 5090 rig's 80-120 tok/s

**Electricity cost (France, ~€0.22/kWh, 8h/day use):**

| Config | Peak draw | 8h/day monthly | Cost/month |
|--------|-----------|----------------|------------|
| 4× RTX 3090 | ~1,400W + system | ~370 kWh | ~€81 |
| 4× RTX 5090 full | ~2,580W + system | ~620 kWh | ~€136 |
| 4× RTX 5090 undervolted (460W) | ~1,960W | ~470 kWh | ~€103 |
| DGX Spark | ~200W | ~48 kWh | ~€11 |
| Mac Studio M4 Ultra | ~75W | ~18 kWh | ~€4 |

The 3090 rig draws nearly as much as the 5090 rig for roughly half the inference speed. The "cheap" hardware isn't cheap to run.

*Verdict: great if your total budget is €5,000. Wrong choice for a dedicated prosumer rig built to last 4 years.*

**2× or 4× RTX 4090**

The most common recommendation in 2024. By mid-2026 the calculus has shifted:

- **24 GB ceiling is real** — Qwen3 70B Q4_K_M needs ~40 GB. On 2× 4090 (48 GB), the model fits but you have almost no KV cache budget. Long contexts start spilling to RAM, speeds drop to ~8-15 tok/s
- **LTX Video 2** fits on a single 4090 with memory management nodes, but barely. Any Flux.2 + LTX-2 simultaneous workflow exhausts VRAM on a 2-card system
- **4× 4090 (96 GB)** is more compelling, but at ~€6,000 GPU cost vs. ~€8,800 for 4× 5090, the 32% extra spend buys 78% more bandwidth per card and 33% more VRAM. The 5090 is better value at the 4-card tier
- **Street prices** — the 4090 has not dropped significantly despite the 5090 launch. You're paying near-5090 prices for Ada-gen bandwidth

*Verdict: optimal for 1-2 card setups. Superseded at the 4-card tier by the 5090 on a pure value basis.*

**Mac Mini M4 + eGPU RTX 5090**

This specific combo gets asked about constantly. It doesn't work:

- **macOS on Apple Silicon does not support CUDA on external GPUs, period.** The NVIDIA GPU connected via Thunderbolt would appear for display output in legacy Intel Mac configurations, but on Apple Silicon (M4), there is no external GPU compute support in macOS at all
- **No CUDA = no PyTorch, no ComfyUI, no vLLM, no Flux.2, no LTX-2** in their standard forms. You'd need Metal-only alternatives for everything, which don't exist for the full creative stack in 2026
- **Thunderbolt 4 bandwidth is ~5 GB/s effective** — even if CUDA worked, loading a 32 GB model would take minutes. Any framework that transfers tensors between CPU and eGPU would be crippled
- **Total cost: ~€4,000+** for a system where the 5090 contributes zero compute

*Verdict: does not work. The Mac Mini M4 is excellent as a control plane / always-on API router using MLX or llama.cpp Metal — but that is a different machine role, not a GPU compute node. The 5090 belongs on a Windows or Linux machine.*

**Apple Silicon bandwidth & memory — not all M-chips are equal:**

| Chip | Unified Memory BW | Max Memory | Best for |
|------|-------------------|------------|---------|
| M1 Max | 400 GB/s | 64 GB | Small 7-13B models, light inference |
| M4 Max | 410 GB/s | 128 GB | Up to ~34B BF16, limited 70B |
| M1 Ultra | 800 GB/s | 128 GB | 70B comfortable |
| M4 Ultra | 819 GB/s | 192 GB | 70B+ comfortable, 405B partial |

M1 and M4 are **not** equivalent — M4 Ultra (819 GB/s, 192 GB) is 2× faster than M4 Max (410 GB/s) for inference. An M4 Max Mac Studio is often compared incorrectly with the M4 Ultra Mac Studio — they're different class machines. Always confirm which chip when comparing benchmarks.

**Mac Studio M4 Ultra (192 GB unified)**

The most legitimate alternative for one specific use case: running 405B models in silence.

- **What it does uniquely well:** 192 GB unified memory at 819 GB/s, silent, 40W draw, runs 405B+ models Q4 entirely in memory, plug-and-play
- **What it cannot do:**
  - No CUDA. No ComfyUI Flux.2, no LTX Video 2, no standard PyTorch LoRA training, no vLLM
  - Apple MLX inference is maturing quickly but remains a subset of the CUDA ecosystem. Many current frontier-equivalent models and specialized agent fine-tunes have no optimized MLX path
  - ~30-40 tok/s on 70B Q4 via MLX vs. ~80-120 tok/s on the 4×5090. Unified memory bandwidth (819 GB/s) is lower per-slot than a single 5090 (1,792 GB/s)
  - At ~€5,999 you get one machine with no expansion path

*Verdict: right choice if your entire workflow is text inference via MLX and you need the largest possible context in silence. Wrong choice if any part of your workflow touches ComfyUI, Flux.2, LTX-2, or standard PyTorch.*

**M5 Ultra — coming mid-2026 (WWDC estimate):**
Apple skipped M4 Ultra entirely (canceled). The next Mac Studio will ship M5 Ultra with:
- **80 GPU cores**, up to **256 GB unified memory**, **~1,100 GB/s bandwidth**
- MLX-only (no CUDA) — same limitation as current Apple Silicon
- Estimated price: ~€6,000-8,000
- At 1,100 GB/s and 256 GB: 70B inference would approach ~50-60 tok/s, and 405B+ models would fit comfortably

The M5 Ultra is the only upcoming single-machine alternative that approaches the 4×5090 rig's VRAM capacity — but at 1/4 the cost and 1/50 the power draw. If you don't need CUDA (ComfyUI/Flux.2/Wan 2.2), the M5 Ultra Mac Studio will be compelling when it ships.

**Threadripper PRO 7000 WRX90 + 4× RTX 5090**

The proper professional platform. Covered in detail in the Architecture section.

- **Full x16 PCIe 5.0 per GPU** — no bifurcation, no asymmetry
- **+€3,500-4,500** over AM5 for CPU + WRX90 motherboard
- **5-15% better throughput** for multi-GPU tensor parallelism
- **Matters for training** — serious gradient synchronization across 4 cards needs the bandwidth
- **Marginal for inference** — the bottleneck is within-GPU GDDR7 bandwidth (1.8 TB/s per card), not cross-GPU PCIe communication

*Verdict: the right platform for a production training rig or commercial AI server. Overkill for inference + LoRA fine-tuning. The €3,500 saved funds the 4th 5090.*

### NVIDIA DGX Spark ($4,699) — The Most Common Misconception

The DGX Spark is NVIDIA's consumer AI supercomputer: 128GB unified memory, full CUDA stack, compact desktop form factor. On paper it looks like a direct competitor.

In practice, for 70B+ inference, it is **not**:

- **2.7 tokens/second decode on Llama 70B** — this is the published benchmark from LMSYS (October 2025). The 128GB unified memory runs at ~273 GB/s. That's the bottleneck. Same VRAM as the 4×5090 rig, 30× slower on 70B decode
- Fine-tuning via QLoRA peaks at 5,079 tok/s for Llama 3.3 70B — impressive, but the 4×5090 rig has discrete VRAM with full CUDA training support
- For **small models** (8B-14B), the DGX Spark shines: compact, silent, zero setup, good throughput at its target tier
- **Two DGX Sparks linked** (~$9,400) run Qwen3-235B at 11.7 tok/s — same ballpark as the 4×5090 at a higher cost

*Verdict: right machine for a compact always-on 8-30B inference node. Wrong machine for 70B+ at interactive speeds. The unified memory bandwidth ceiling is fundamental, not patchable.*

**Benchmark comparison:**

| Config | 70B Q4 tok/s | Memory BW | Cost |
|--------|-------------|-----------|------|
| DGX Spark | ~2.7 tok/s | 273 GB/s unified | $4,699 |
| Mac Studio M4 Ultra | ~30-40 tok/s | 819 GB/s unified | ~€5,999 |
| **4× RTX 5090 (this build)** | **~80-120 tok/s** | **1,792 GB/s × 4** | **~€22,800** |

### NVIDIA RTX Pro 6000 Blackwell (96GB, ~€8,000-9,000) — One Card to Rule Them All?

The RTX Pro 6000 Blackwell is a professional workstation GPU with 96GB GDDR7. It fits 70B FP8 on a single card with 26GB left for KV cache.

- **What it does well:** 70B FP8 on one card, ~8,400 tok/s on 30B AWQ (near 4×RTX 4090 on those workloads), no multi-GPU complexity, ECC memory for long unattended runs, 600W for one slot
- **Why not for this build:**
  - €8,000-9,000 for one card = can't afford 4 of them (€32,000+)
  - 96GB < 128GB — can't run 70B FP16 or comfortable 405B
  - One card = no LTX Video 2 + Qwen3 70B simultaneously
  - **No NVLink** — multi-card configs still face PCIe bottlenecks like consumer cards
  - 600W per card × 4 = 2,400W before system overhead — same thermal challenge as 4× 5090, at 4× the GPU cost

*Verdict: compelling for a one-card workstation where multi-GPU complexity is undesirable. Not the right choice for a dedicated multi-workload inference + ComfyUI server.*

### The Decision Tree

```
Need CUDA (ComfyUI / Flux.2 / LTX-2 / vLLM / PyTorch)?
├── No → Mac Studio M4 Ultra for 405B in silence
└── Yes
    ├── Budget < €8,000 total?
    │   ├── < €5,000 → 2× RTX 5090 (your current Nomad/Torrent)
    │   └── < €8,000 → 4× RTX 4090 if 5090 unavailable
    └── Budget ≥ €15,000 (full build)?
        ├── Inference-focused → 4× RTX 5090 on AM5 (this build)
        └── Training-focused → 4× RTX 5090 on Threadripper WRX90
```

The 4× RTX 5090 on AM5 sits at the intersection of **maximum CUDA VRAM** and **minimum viable platform cost**. It is not the most elegant system — but it is the right one for the use case.

---

## Cost vs. Cloud: Is It Worth It?

### Cloud GPU Pricing (mid-2026 estimates)

| Provider | GPU Config | $/hour | Monthly (24/7) |
|----------|-----------|--------|-----------------|
| RunPod | 4×A100 80GB | ~$8-12/hr | ~$6,000-9,000 |
| Lambda | 4×H100 80GB | ~$12-16/hr | ~$9,000-12,000 |
| vast.ai | 4×RTX 5090 | ~$4-6/hr | ~$3,000-4,500 |
| This build | 4×RTX 5090 (owned) | ~€0.50/hr electricity | ~€360/month |

### Break-Even Analysis

Total build cost: ~€16,500. Monthly electricity at ~2 kW average (undervolted, not 24/7 full load): ~€360/month at €0.18/kWh.

Compared to renting 4×5090 on vast.ai at ~$5/hr:
- If you use the rig **8 hours/day**: cloud cost = ~$1,200/month vs. your electricity = ~€120/month
- **Break-even: ~14-16 months** of regular use
- After that, every month of usage is essentially free (minus electricity)

For a filmmaker/researcher who runs inference daily, fine-tunes LoRAs weekly, and needs a persistent local API server, this build pays for itself within the first year and a half.

### The Real Advantage: Availability and Privacy

Beyond cost, owning the hardware means:
- **No queue, no cold starts** — your models are always loaded and warm
- **Full data privacy** — nothing leaves your network
- **No API rate limits** — run as many requests as your GPUs can handle
- **Experimentation freedom** — try weird models, custom quantizations, and bleeding-edge software without per-hour anxiety

---

---

## Complementary Node: Mac Mini M5 Pro or Mac Studio M5

The 4×5090 rig is a CUDA machine — loud, hot, power-hungry, and not always-on 24/7. A Mac Mini or Mac Studio M5 fills the gaps: silent, 35-80W, always-on, and capable of serious MLX inference for everything that doesn't need CUDA.

### Rumored Specs — Mid-2026 (WWDC estimate)

| Config | Unified Memory | Bandwidth | Est. Price | Sweet spot for |
|--------|---------------|-----------|------------|----------------|
| Mac Mini M5 (base) | 16-32 GB | 153 GB/s | ~€699 | Control plane only, 7-14B agents |
| **Mac Mini M5 Pro 48 GB** | 48 GB | 307 GB/s | ~€1,600 | ★ Best value — 30B models, light 70B |
| **Mac Mini M5 Pro 64 GB** | 64 GB | 307 GB/s | ~€2,100 | ★★ Sweet spot — 70B Q4 fits fully |
| Mac Studio M5 Max 128 GB | 128 GB | 614 GB/s | ~€3,500 | Real 70B inference speed (~30-40 tok/s) |
| Mac Studio M5 Ultra 256 GB | 256 GB | 1,100 GB/s | ~€6,500 | 405B+ fits, niche use case |

> Specs are rumored — Apple hasn't announced Mac Mini M5 or Mac Studio M5 as of April 2026. Expected WWDC June 2026. [Current rumors](https://www.macworld.com/article/2964754/2026-mac-mini-m5-pro-design-specs-release-date.html).

### The Recommendation

**Start with Mac Mini M5 Pro 48 GB (~€1,600).**

- 307 GB/s, 48 GB unified memory
- Runs Qwen3 32B via MLX at ~25-35 tok/s — covers 90% of daily agent tasks
- 70B Q4_K_M fits at 48 GB with minimal KV cache; slower (~12-18 tok/s) but functional for background tasks
- Always-on at ~35W — the 5090 rig can sleep, the Mac Mini handles Hermes Agent calls overnight
- M5 Pro adds Thunderbolt 5 and Wi-Fi 7

If you find yourself needing 70B at real speed or 128B+ models on the Mac side: upgrade to Mac Studio M5 Max (128 GB, 614 GB/s). The Mac Mini M5 Pro is the entry point; the Studio is the upgrade path.

**If budget allows from day one: Mac Studio M5 Max 128 GB** at ~€3,500 is genuinely a different machine — 614 GB/s means 70B Q8 entirely in unified memory at 30-40 tok/s. Comparable to the 4×5090 rig on pure 70B throughput (not faster, but silent and 80W).

### Why Not Mac Studio M5 Ultra?

The Ultra (256 GB, ~€6,500) is compelling only if you need 405B+ models locally. For everything else, the Max 128 GB gives 85% of the capability at 55% of the price. The Ultra is a statement purchase for 2026.

### EXO: Pool Everything

The real advantage emerges with [EXO](https://github.com/exolabs/exo): one framework that pools VRAM/unified memory across heterogeneous devices on your Tailscale network.

```bash
# Mac Mini M5 Pro: EXO node (MLX backend)
exo --node-id mac-mini --backend mlx

# 4×5090 rig: EXO node (CUDA backend)
exo --node-id rig --backend cuda

# From any machine: query the pooled cluster
curl http://localhost:52415/v1/chat/completions   -d '{"model": "llama-3.1-405b", "messages": [...]}'
# EXO distributes layers across both machines
```

**What you get from pooling:**
- 128 GB VRAM (rig) + 48-64 GB unified memory (Mac Mini) = up to ~192 GB distributed
- The Mac Mini handles MLX-native layers; the rig handles CUDA-optimized layers
- Larger models become reachable without buying more GPUs
- Each additional Mac you add later expands the pool

Scale path: Mac Mini M5 Pro 48GB → add Mac Studio M5 Max → add second Mac Mini if needed. Each purchase expands distributed inference capacity without touching the CUDA rig.

---

## Resources

- [LINKUP AVA5 PCIe 5.0 Risers](https://linkup.one) — validated for RTX 5090
- [Ollama](https://ollama.com) — simplest local LLM runtime
- [vLLM](https://github.com/vllm-project/vllm) — high-throughput inference with tensor parallelism
- [llama.cpp](https://github.com/ggml-org/llama.cpp) — flexible GGUF inference
- [NVIDIA System Management Interface](https://developer.nvidia.com/nvidia-system-management-interface) — `nvidia-smi` for monitoring and power management
- [GPU-Z](https://www.techpowerup.com/gpuz/) — verify PCIe link speed and generation

---

## License

This build guide is shared for educational purposes. Hardware choices reflect my specific needs (filmmaker + AI researcher) and budget constraints. Your mileage may vary.

Built with help from Claude Code. Mistakes are mine.

By [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com).
