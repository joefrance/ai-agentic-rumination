# Running AI Models Locally

## A Practical Guide to Local Inference (2025–2026)

---

## 1. Introduction — Why Run Models Locally?

Cloud-hosted AI is convenient, but it comes with strings attached: your data leaves your machine, you pay per token, you need an internet connection, and you're subject to someone else's acceptable use policy. Running models locally severs those strings.

**The case for local inference:**

- **Privacy** — Your prompts and data never leave your hardware. No logging, no training on your inputs, no third-party access. For sensitive code, medical data, or legal documents, this isn't a preference — it's a requirement.
- **Cost** — After the upfront hardware investment, inference is free. For heavy usage (thousands of requests per day), local inference pays for itself quickly.
- **Offline access** — Works on airplanes, in secure facilities, and during outages. No dependency on external services.
- **Latency** — No network round-trip. For interactive use and IDE integrations, local models can feel faster than cloud APIs even when they're technically slower at generation.
- **Customization** — Full control over model selection, quantization, system prompts, sampling parameters, and fine-tuning. No guardrails you didn't choose.
- **Autonomy** — No API deprecations, no surprise pricing changes, no terms-of-service updates that break your workflow. The model you download today works forever.

The tradeoff is real: local models are smaller, less capable, and require you to manage your own hardware. But for many tasks — code completion, summarization, drafting, analysis, RAG pipelines — a well-chosen local model is more than sufficient.

---

## 2. Available Models (2025–2026)

The open-weight model landscape has exploded. These are the major families worth knowing about, with sizes that range from "runs on a phone" to "needs a server rack."

### Quick Reference

| Model Family | Sizes Available | Architecture | Sweet Spot for Local | License |
|---|---|---|---|---|
| **Llama** (Meta) | 1B, 3B, 8B, 70B, 405B; Scout 109B, Maverick 400B | Dense; Scout/Maverick MoE | 8B (Q4–Q5) | Llama Community |
| **DeepSeek** | R1 distills: 1.5B–70B; V3: 671B MoE | Dense (distills), MoE (V3) | R1-distill 14B–32B | MIT (distills) |
| **Qwen 3** (Alibaba) | 0.6B, 1.7B, 4B, 8B, 14B, 32B; 30B-A3B, 235B-A22B MoE | Dense + MoE | 14B–32B dense, 30B-A3B MoE | Apache 2.0 |
| **Mistral / Ministral** | Ministral 3B, 8B; Mistral 24B (Small); Mixtral 8×7B, 8×22B | Dense + MoE | Mistral Small 24B | Apache 2.0 |
| **Phi-4** (Microsoft) | 3.8B (mini), 14B; reasoning variants | Dense | 14B | MIT |
| **Gemma 3** (Google) | 1B, 4B, 12B, 27B | Dense, multimodal capable | 12B–27B | Gemma License |
| **Command R** (Cohere) | 35B, 104B | Dense, RAG-optimized | 35B | CC-BY-NC |

### Model Family Notes

**Llama** — Meta's flagship open-weight family. Llama 3.3 70B remains one of the strongest open models at any size. Llama 4 introduced mixture-of-experts with Scout (17B active of 109B total) and Maverick (17B active of 400B total), trading total parameter count for efficiency. The 8B model is the workhorse of local inference — fast, capable, and fits everywhere.

**DeepSeek R1** — The reasoning-focused family from DeepSeek. The distilled variants (1.5B–70B) are standalone dense models distilled from the full R1, not quantized versions of a larger model. They punch above their weight on reasoning and math tasks. The 14B and 32B distills are particularly strong for their size. V3 at 671B is a massive MoE model that requires serious hardware.

**Qwen 3** — Alibaba's multilingual models with excellent benchmark performance. The 30B-A3B MoE variant is notable: 30B total parameters but only 3B active per token, giving 30B-class knowledge with 3B-class speed. The dense models from 14B to 32B are strong general-purpose choices.

**Mistral / Ministral** — The French lab's offerings span from the tiny Ministral 3B (edge devices) to Mistral Small 24B (a dense model that competes well above its size class). Mixtral pioneered the open MoE approach. Mistral Small 24B is a standout for local use — better quality-per-VRAM than many larger models.

**Phi-4** — Microsoft's "small but mighty" family. Phi-4 14B consistently outperforms expectations on reasoning benchmarks. The reasoning variants add chain-of-thought capabilities. At 3.8B, Phi-4 mini is one of the best options for constrained hardware.

**Gemma 3** — Google's open models with native multimodal support (vision + text). Gemma 3 27B is competitive with much larger models on many tasks. The 12B variant offers a good balance for machines with 16GB of memory. Vision capability without a separate model is a meaningful differentiator.

**Command R** — Cohere's RAG-optimized models. Command R 35B is specifically designed for retrieval-augmented generation: it follows grounding documents faithfully and cites sources. If your use case is "answer questions from my documents," this family is purpose-built for it. The 104B variant requires substantial hardware.

---

## 3. Quantization

Quantization is how you fit a 14-billion-parameter model into 8GB of memory. It's the single most important concept for local inference.

### What It Is

Neural network weights are stored as numbers. At full precision (FP16), each parameter takes 2 bytes. A 14B model at FP16 requires ~28GB of memory — more than most consumer GPUs have. Quantization reduces the precision of these numbers, shrinking the model at the cost of some quality.

Think of it like JPEG compression for AI models: you lose some fidelity, but the result is often indistinguishable from the original for practical use.

### Quantization Levels

| Level | Bits/Param | Size (7B model) | Size (14B model) | Quality vs FP16 | Use Case |
|---|---|---|---|---|---|
| **FP16** | 16 | ~14 GB | ~28 GB | Baseline (100%) | Research, when you have the VRAM |
| **Q8** | 8 | ~7.5 GB | ~15 GB | ~99.5% | Near-lossless; best quality-per-bit tradeoff |
| **Q6_K** | 6.6 | ~6.0 GB | ~12 GB | ~99% | Excellent quality, moderate savings |
| **Q5_K_M** | 5.5 | ~5.2 GB | ~10.5 GB | ~98% | Strong daily driver |
| **Q4_K_M** | 4.5 | ~4.4 GB | ~8.8 GB | ~96–97% | The sweet spot — good quality, fits everywhere |
| **Q3_K_M** | 3.5 | ~3.5 GB | ~7.0 GB | ~92–94% | Noticeable degradation on complex tasks |
| **Q2_K** | 2.5 | ~2.8 GB | ~5.5 GB | ~85–88% | Emergency use; significant quality loss |

*Sizes are approximate and vary by model architecture. The "_K_M" suffix indicates k-quant mixed precision — different layers quantized at different levels based on importance.*

### Quantization Formats

- **GGUF** — The standard format for llama.cpp and everything built on it (Ollama, LM Studio, GPT4All). CPU and GPU compatible. This is what most people use.
- **GPTQ** — GPU-only format, popular in the Hugging Face ecosystem. Good for NVIDIA GPUs with CUDA.
- **AWQ** — Activation-aware quantization. Slightly better quality than GPTQ at the same bit width by preserving important weights. GPU-only.
- **EXL2** — ExLlamaV2's format. Variable bits-per-weight across layers. Excellent quality-per-bit but only works with ExLlamaV2. GPU-only.

**For most people: use GGUF.** It's the most portable, best supported, and works on CPU, GPU, and Apple Silicon.

### The Key Insight

> **A bigger model at lower quantization almost always beats a smaller model at higher quantization.**

A 14B model at Q4 will generally outperform a 7B model at Q8, even though they use similar amounts of memory. The larger model has more knowledge baked into its parameters; quantization compresses that knowledge but doesn't erase it. When in doubt, go bigger and quantize harder.

---

## 4. Hardware Requirements

### 4a. The Core Formula

The memory needed to run a model is straightforward:

```
Memory ≈ Parameters × Bytes-per-Parameter × 1.1–1.2
```

The 1.1–1.2x overhead accounts for KV cache (which grows with context length), runtime buffers, and the inference engine itself.

| Model Size | FP16 | Q8 | Q5_K_M | Q4_K_M | Q3_K_M |
|---|---|---|---|---|---|
| **1B** | 2 GB | 1.1 GB | 0.8 GB | 0.7 GB | 0.5 GB |
| **3B** | 6 GB | 3.4 GB | 2.4 GB | 2.0 GB | 1.6 GB |
| **7–8B** | 16 GB | 8.5 GB | 5.9 GB | 5.0 GB | 4.0 GB |
| **14B** | 28 GB | 15 GB | 12 GB | 10 GB | 8.0 GB |
| **32B** | 64 GB | 34 GB | 24 GB | 20 GB | 16 GB |
| **70B** | 140 GB | 75 GB | 52 GB | 43 GB | 35 GB |

*Sizes include ~15% overhead for context and runtime. Actual usage varies with context length — longer contexts need more KV cache memory.*

### 4b. Standard Laptops (Discrete GPU)

Most laptops with discrete GPUs have 6–16GB of VRAM. This is your hard constraint — the model must fit in VRAM for acceptable speed.

- **6–8 GB VRAM** (RTX 3060 Laptop, RTX 4060 Laptop) → 7–8B models at Q4, or 3B models at Q6–Q8. Expect 15–30 tokens/second.
- **12–16 GB VRAM** (RTX 4070/4080 Laptop) → 14B models at Q4–Q5, or 8B models at Q6–Q8. Expect 20–40 tokens/second.

CPU-only inference (no GPU) works but is slow: expect 2–8 tokens/second for a 7B model on a modern laptop CPU. Usable for batch processing, painful for interactive chat.

### 4c. Desktop GPUs

Desktop GPUs are the price-performance champions for local inference.

| GPU | VRAM | Memory Bandwidth | ~tok/s (7B Q4) | ~tok/s (14B Q4) | ~tok/s (70B Q4) | Street Price (2025) |
|---|---|---|---|---|---|---|
| **RTX 3090** | 24 GB | 936 GB/s | 80–100 | 40–55 | — | $700–900 used |
| **RTX 4090** | 24 GB | 1,008 GB/s | 100–130 | 55–70 | — | $1,600–1,900 |
| **RTX 5090** | 32 GB | 1,792 GB/s | 140–170 | 80–100 | 20–30* | $2,000–2,500 |
| **RTX 3060 12GB** | 12 GB | 360 GB/s | 30–40 | 15–20* | — | $200–250 used |

*70B on RTX 5090 requires aggressive Q3 quantization. 14B on RTX 3060 requires Q3–Q4 and is tight. "—" means the model doesn't fit.*

The **RTX 3090** remains the best value proposition in 2025–2026. At $700–900 used, you get 24GB of VRAM and performance that handles everything up to 32B models comfortably.

### 4d. Apple Silicon (Unified Memory)

Apple Silicon changes the equation in a fundamental way: **all system RAM is available as "VRAM."** A Mac with 64GB of unified memory can load a 64GB model — no discrete GPU required.

#### Why This Matters

On a traditional PC, you have separate pools: 32GB of system RAM and 24GB of GPU VRAM. The model must fit in VRAM for fast inference. On Apple Silicon, there's one pool: 64GB unified memory that both the CPU and GPU cores access.

#### The Bottleneck: Memory Bandwidth

The catch is that Apple Silicon's memory bandwidth is lower than a discrete GPU's. An RTX 4090 pushes ~1,008 GB/s; an M4 Max pushes ~546 GB/s. Since inference speed is directly proportional to bandwidth (you're streaming the model weights through the compute units for every token), Apple Silicon generates tokens slower per-GB than a discrete GPU.

But it can load *much bigger models*. An M4 Max with 128GB runs a 70B Q4 model that simply won't fit on any single consumer GPU. Slower per-token, but actually possible — and "slower" still means 15–25 tok/s, which is perfectly usable.

#### The Lineup

| Chip | Max Memory | Bandwidth | ~tok/s (7B Q4) | ~tok/s (14B Q4) | ~tok/s (70B Q4) | Notes |
|---|---|---|---|---|---|---|
| **M1/M2 base** | 16–24 GB | 100 GB/s | 12–18 | 6–10 | — | Entry-level; 8B models only |
| **M2 Pro** | 32 GB | 200 GB/s | 20–30 | 12–18 | — | 14B comfortable |
| **M3 Pro** | 36 GB | 150 GB/s | 18–25 | 10–15 | — | Less bandwidth than M2 Pro |
| **M3 Max** | 128 GB | 400 GB/s | 35–50 | 25–35 | 10–15 | Great for large models |
| **M4 Pro** | 48 GB | 273 GB/s | 25–35 | 18–25 | — | Good memory, moderate bandwidth |
| **M4 Max** | 128 GB | 546 GB/s | 50–65 | 35–50 | 15–25 | The sweet spot for Apple Silicon |
| **M4 Ultra** | 256 GB | 819 GB/s | 70–90 | 50–70 | 25–40 | Runs everything, including 405B at Q2–Q3 |

#### Bandwidth Beats Chip Generation

A counterintuitive result: **the M3 Max (400 GB/s) outperforms the M4 Pro (273 GB/s) at inference** despite being an older chip. Memory bandwidth determines token generation speed, not GPU core count or chip generation. When choosing a Mac for local inference:

1. Maximize memory bandwidth first
2. Maximize total memory second
3. Chip generation is a distant third

#### Mac vs. Discrete GPU — When to Choose Which

| Factor | Apple Silicon Mac | Desktop GPU (RTX 4090/5090) |
|---|---|---|
| **Max model size** | Much larger (128–256 GB) | Limited by VRAM (24–32 GB) |
| **Speed (small models)** | Slower | Faster |
| **Speed (large models)** | Only option at consumer prices | Can't load them |
| **Power consumption** | 30–60W total system | 300–450W GPU alone |
| **Noise** | Silent to quiet | Loud under load |
| **Dual use** | Full workstation | Dedicated inference box |
| **Price/performance** | Worse for small models | Better for small models |

**Choose Apple Silicon** when you need to run models larger than 24–32GB (the discrete GPU VRAM ceiling), when power/noise matter, or when the machine serves double duty as a workstation.

**Choose a desktop GPU** when you're primarily running 7B–14B models and want maximum tokens/second for the dollar.

### 4e. Multi-GPU Setups

When a model doesn't fit on a single GPU, you can split it across multiple GPUs. But the scaling is sublinear.

- **2× RTX 3090** (48GB total VRAM): Can run 70B at Q4. But PCIe bandwidth between the GPUs limits throughput — expect ~1.4–1.6x the speed of a single GPU, not 2x.
- **NVLink** (available on workstation GPUs) provides much higher inter-GPU bandwidth and better scaling (~1.7–1.8x), but the GPUs cost significantly more.
- **Tensor parallelism** (splitting layers across GPUs) works well for throughput. **Pipeline parallelism** (splitting the model sequentially) adds latency per token.

**The rule of thumb:** Multi-GPU is worth it when and only when your model doesn't fit on a single card. If it fits on one GPU, one GPU will be faster and simpler than two. The money spent on a second GPU is almost always better spent on a single larger GPU.

---

## 5. Quality vs. Speed

### Quantization Quality Impact

How much quality do you actually lose? Perplexity measurements on standard benchmarks give a rough answer (lower perplexity = better):

| Quantization | Perplexity Increase | Quality Retention | Subjective Assessment |
|---|---|---|---|
| **FP16** | Baseline | 100% | Reference quality |
| **Q8** | +0.01–0.02 | ~99.5% | Indistinguishable from FP16 in practice |
| **Q6_K** | +0.03–0.06 | ~99% | Virtually no noticeable difference |
| **Q5_K_M** | +0.05–0.10 | ~98% | Slight degradation on edge cases |
| **Q4_K_M** | +0.10–0.20 | ~96–97% | Occasional odd phrasing; still strong |
| **Q3_K_M** | +0.25–0.50 | ~92–94% | Noticeable on complex reasoning |
| **Q2_K** | +0.50–1.50 | ~85–88% | Significant quality loss; coherence suffers |

*Perplexity numbers are illustrative ranges across multiple model families. Actual impact varies by model — some architectures are more quantization-resilient than others.*

### Speed by Hardware

Tokens per second for common configurations (generation speed, not prompt processing):

| Configuration | 7B Q4 | 14B Q4 | 32B Q4 | 70B Q4 |
|---|---|---|---|---|
| Laptop CPU (no GPU) | 3–8 | 1–4 | — | — |
| RTX 3060 12GB | 30–40 | 15–20 | — | — |
| RTX 3090 24GB | 80–100 | 40–55 | 20–30 | — |
| RTX 4090 24GB | 100–130 | 55–70 | 25–35 | — |
| RTX 5090 32GB | 140–170 | 80–100 | 40–55 | 20–30 |
| M4 Pro 48GB | 25–35 | 18–25 | 10–15 | — |
| M4 Max 128GB | 50–65 | 35–50 | 20–30 | 15–25 |
| M4 Ultra 256GB | 70–90 | 50–70 | 30–45 | 25–40 |

### Usability Thresholds

Not all tokens-per-second are created equal. What actually feels usable?

| Speed | Experience | Suitable For |
|---|---|---|
| **1–3 tok/s** | Painful. Like watching text load on a 1990s modem. | Batch processing only |
| **4–8 tok/s** | Tolerable if you're patient. Each word appears with visible delay. | Background tasks, non-interactive use |
| **8–15 tok/s** | Usable. Reads like slow typing. You can follow along. | Chat, light coding assistance |
| **15–30 tok/s** | Comfortable. Feels responsive. The speed most people find acceptable. | Daily driver for most tasks |
| **30–60 tok/s** | Fast. Text appears in fluid bursts. | IDE integrations, interactive coding |
| **60+ tok/s** | Instant. Feels like the text is just *there*. | Demanding workflows, streaming applications |

### The Practical Rule

> **Bigger model + lower quantization > smaller model + higher quantization.**

Given a fixed memory budget, you'll almost always get better results from a larger model at Q4 than a smaller model at Q8. A 14B-Q4 (~10 GB) meaningfully outperforms an 8B-Q8 (~8.5 GB) on most tasks, despite using only ~1.5 GB more memory.

The exception: if you need maximum coherence on a specific narrow task and speed doesn't matter, a smaller model at higher precision can sometimes edge out a larger quantized model. But for general use, go big and quantize.

---

## 6. Software Platforms Compared

### Overview

| Platform | Type | Ease of Use | Best For | GPU Support | API Server | Open Source |
|---|---|---|---|---|---|---|
| **Ollama** | CLI + API | Very easy | Developers, integrations | CUDA, ROCm, Metal | Yes (OpenAI-compatible) | Yes |
| **LM Studio** | GUI + API | Very easy | Exploration, model discovery | CUDA, Metal | Yes (OpenAI-compatible) | No (free) |
| **llama.cpp** | CLI / library | Advanced | Max performance, full control | CUDA, ROCm, Metal, Vulkan | Yes | Yes |
| **GPT4All** | GUI | Easy | Local RAG (LocalDocs) | CUDA, Metal | Limited | Yes |
| **Jan.ai** | GUI | Easy | Privacy-focused local chat | CUDA, Metal | Yes | Yes |
| **LocalAI** | Docker / API | Moderate | Multi-modal, production deployments | CUDA, ROCm | Yes (OpenAI-compatible) | Yes |
| **vLLM** | Server | Advanced | Production serving, high throughput | CUDA | Yes (OpenAI-compatible) | Yes |

### Platform Details

**Ollama** — The Docker of local AI. Pull a model with `ollama pull llama3.3`, run it with `ollama run llama3.3`. That's it. Ships an OpenAI-compatible API server on port 11434, making it trivially easy to integrate with any tool that speaks the OpenAI protocol. Supports model customization via Modelfiles. Limitations: less fine-grained control over inference parameters than llama.cpp; model library is curated (not every quantization is available).

**LM Studio** — A polished desktop app for browsing, downloading, and running models. Excellent for exploration: search Hugging Face directly from the UI, preview model cards, compare outputs side-by-side. Includes a local API server compatible with the OpenAI SDK. Limitations: closed-source (though free); less scriptable than CLI tools; model management is GUI-only.

**llama.cpp** — The engine that most other tools are built on. Written in C/C++ with no dependencies, it runs on everything from Raspberry Pis to multi-GPU servers. Offers the most control: custom quantization, speculative decoding, grammar-constrained output, prompt caching, and continuous batching. Limitations: no GUI; requires comfort with command-line tools and build systems; the raw interface is not beginner-friendly.

**GPT4All** — Desktop application with a standout feature: **LocalDocs**. Point it at a folder of documents and it builds a local RAG pipeline — your model answers questions grounded in your files, with citations. Good for non-technical users who want "ChatGPT for my documents." Limitations: smaller model selection; less developer-oriented; the RAG pipeline is simpler than dedicated solutions.

**Jan.ai** — A privacy-first desktop chat application. Clean interface, local-only by design (your data never leaves your machine, by architecture, not just by policy). Supports extensions and plugins. Limitations: smaller community than Ollama/LM Studio; fewer advanced features; extension ecosystem is still maturing.

**LocalAI** — A self-hosted, OpenAI-compatible API that supports text, images, audio, and embeddings from a single endpoint. Deploy via Docker with GPU passthrough. Good for teams that want a private, multi-modal AI API without cloud dependencies. Limitations: more complex setup than Ollama; documentation can lag behind features; Docker adds a layer of complexity.

**vLLM** — A high-throughput inference server designed for production workloads. Features continuous batching, PagedAttention (efficient memory management), and tensor parallelism across GPUs. Optimized for serving many concurrent users, not single-user desktop use. Limitations: NVIDIA GPUs only (no Metal/ROCm support at the time of writing); overkill for personal use; requires more infrastructure knowledge.

### Which Should You Use?

- **Just getting started?** → Ollama (if you like the terminal) or LM Studio (if you prefer a GUI)
- **Building an application?** → Ollama for prototyping, vLLM for production
- **Want maximum performance?** → llama.cpp directly
- **Want RAG on local documents?** → GPT4All (simple) or build your own pipeline with Ollama + a vector database
- **Serving a team?** → LocalAI (multi-modal) or vLLM (text, high throughput)

---

## 7. Practical Recommendations

### By Hardware Budget

#### Free ($0) — Use What You Have

**Hardware:** Your existing laptop or desktop, no purchases.

**What works:**
- 3B models (Phi-4 mini 3.8B, Ministral 3B, Llama 3.2 3B) at Q4–Q5
- CPU-only inference at 3–8 tok/s — slow but functional
- If you have any GPU with 6+ GB VRAM, jump to 7B–8B models

**Reality check:** Usable for experimentation and light tasks. Not a daily driver. Good enough to learn whether local models work for your use case before spending money.

#### Entry ($300–500) — Capable on a Budget

**Hardware:** Used RTX 3060 12GB ($200–250) in an existing desktop, or a Mac mini M2 with 16GB ($500 refurbished).

**What works:**
- 7B–8B models at Q4–Q5 (Llama 3.1 8B, Phi-4 mini, Gemma 3 4B at higher quant)
- 15–40 tok/s depending on GPU
- Comfortable for chat, summarization, light code assistance

**This tier proves the concept.** You'll know quickly whether local inference fits your workflow.

#### Mid ($800–1,500) — The Sweet Spot

**Hardware:** Used RTX 3090 ($700–900) in a desktop build, or Mac mini M4 Pro with 48GB ($1,400–1,600).

**What works:**
- 14B models at Q4–Q5 (Qwen 3 14B, Phi-4 14B, Gemma 3 12B) — genuinely useful quality
- 32B models at Q3–Q4 on the RTX 3090 — pushing it, but possible
- 30–55 tok/s for 14B models; fast enough for IDE integration
- DeepSeek R1 14B distill for reasoning tasks

**This is where local inference becomes a real tool, not a toy.** A 14B model at Q4 handles most coding, writing, and analysis tasks well. It won't match GPT-4 or Claude Opus, but it handles the 80% of tasks that don't require frontier capability.

#### High ($1,600–2,500) — Competitive with Cloud

**Hardware:** RTX 4090 or 5090, or Mac mini M4 Max with 64–128GB.

**What works:**
- 32B–70B models — Qwen 3 32B, Llama 3.3 70B (on Mac or with quantization tricks on GPU)
- 70B at Q4 on M4 Max 128GB at 15–25 tok/s — slower but genuinely good
- DeepSeek R1 32B distill for strong reasoning
- Multiple models loaded simultaneously (on Mac with sufficient memory)

**At this tier, quality approaches cloud offerings for many tasks.** A 70B model is a remarkably capable general assistant. The gap to frontier cloud models narrows to specialized tasks: the most complex reasoning, the longest contexts, the most nuanced instruction-following.

#### Enthusiast ($3,000+) — Full Frontier-Class Local

**Hardware:** Mac Studio M4 Ultra with 256GB ($4,000–6,000), or multi-GPU desktop (2× RTX 5090).

**What works:**
- Everything, including 70B at high quantization and 405B Llama at Q2–Q3
- DeepSeek V3 671B MoE at aggressive quantization (M4 Ultra only)
- Serving models to a small team via vLLM or LocalAI
- Multiple large models loaded concurrently

**This is "never need a cloud API again" territory** — with the caveat that the very latest frontier models (GPT-4.5, Claude Opus, Gemini Ultra) will still have an edge on the most demanding tasks.

### By Use Case

| Use Case | Recommended Model | Min Hardware | Notes |
|---|---|---|---|
| **Code completion** | Qwen 3 14B, DeepSeek R1 14B | 16 GB VRAM or 24 GB unified | Speed matters more than peak quality |
| **Chat / general assistant** | Llama 3.3 8B or 70B | 8 GB VRAM (8B) or 48 GB+ (70B) | 8B for speed, 70B for quality |
| **RAG / document Q&A** | Command R 35B, Qwen 3 32B | 24 GB VRAM | Models that follow context faithfully |
| **Reasoning / math** | DeepSeek R1 14B–32B distill | 16–24 GB VRAM | R1 distills excel at step-by-step reasoning |
| **Creative writing** | Llama 3.3 70B, Mistral Small 24B | 24 GB+ VRAM or 48 GB+ unified | Larger models produce more nuanced prose |
| **Summarization** | Phi-4 14B, Gemma 3 12B | 12–16 GB VRAM | Even smaller models handle this well |
| **Multilingual** | Qwen 3 14B–32B | 16–24 GB VRAM | Strongest multilingual open model family |
| **Vision / multimodal** | Gemma 3 12B–27B | 16–24 GB VRAM | Native vision support, no separate model needed |

---

## 8. See Also

### Benchmarks and Leaderboards
- [Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard) — Hugging Face's standardized benchmark comparisons
- [LM Arena (Chatbot Arena)](https://lmarena.ai/) — Human preference rankings via blind comparisons
- [Artificial Analysis](https://artificialanalysis.ai/) — Speed, price, and quality benchmarks for hosted and local models

### Model Sources
- [Hugging Face](https://huggingface.co/models) — The central repository for open-weight models
- [Ollama Model Library](https://ollama.com/library) — Curated models ready to pull and run

### Communities
- [r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA/) — The most active community for local model users; hardware advice, benchmarks, new model reviews
- [llama.cpp GitHub Discussions](https://github.com/ggerganov/llama.cpp/discussions) — Technical discussions on inference optimization

### Software
- [Ollama](https://ollama.com/) — Pull and run models in one command
- [LM Studio](https://lmstudio.ai/) — Desktop app for model exploration
- [llama.cpp](https://github.com/ggerganov/llama.cpp) — The foundational inference engine
- [GPT4All](https://gpt4all.io/) — Desktop app with local RAG
- [Jan.ai](https://jan.ai/) — Privacy-first local chat
- [LocalAI](https://localai.io/) — Self-hosted multi-modal API
- [vLLM](https://docs.vllm.ai/) — High-throughput production serving

### Further Reading
- [A Visual Guide to Quantization](https://newsletter.maartengrootendorst.com/p/a-visual-guide-to-quantization) — Maarten Grootendorst's illustrated explainer
- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — Jay Alammar's foundational visual guide to transformer architecture
