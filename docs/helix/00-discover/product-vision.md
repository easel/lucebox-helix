---
ddx:
  id: product-vision
  status: draft
  links:
    - id: prd
      rel: informs
    - id: concerns
      rel: informs
---

# Product Vision

## Mission Statement

Lucebox gives developers and technical users local AI inference on tested
hardware and driver combinations by shipping a dedicated hardware appliance
and software tuned to the hardware for tested hardware-and-model pairs —
so they get predictable performance without driver and runtime setup.

## Positioning

For developers and technical users who run or want to run local LLMs,
who need fast, repeatable inference without choosing compatible hardware
or fixing driver and runtime breakage,
**Lucebox** is an integrated hardware-and-software AI appliance
that tunes the GPU, runtime, model, and UI as one product.
Unlike a Mac Studio running Ollama or a custom PC running llama.cpp,
Lucebox ships tested hardware profiles with software tuned specifically
for those configurations — not a generic setup.

## Vision

Developers and teams with privacy or offline requirements run large AI workloads
locally with cloud-like reliability. Local LLM tools standardize on tested
hardware profiles, and local inference gets fast enough for the coding and
reasoning workloads technical users run most. Running a 70B model locally
becomes as routine as starting a Docker container.

**North Star**: Any developer can unbox a Lucebox, reach their first model
response in under 10 minutes, hit all three throughput targets on Qwen3.6-27B
(≥120 tok/s short-context, ≥60 tok/s at 128K with prefix caching, ≥60 tok/s
at 128K with PFlash), and run a coding-agent workflow locally that replaces
their cloud API subscription.

## User Experience

A developer gets their Lucebox. They plug it in, open a browser to
`lucebox.local`, and see a dashboard showing system resources and a model
library. They pull Qwen3.6-27B with one click — the UI shows a download
progress bar and selects the best quantization for available RAM and VRAM.
Five minutes later they're generating code at ~130 tokens/second through
the Lucebox coding-agent harness — a local coding workflow tuned across model,
runtime, and UI, not a standard client pointing at an OpenAI-compatible endpoint.
Cloud API bills stop for that coding workflow. Two weeks in, they activate
a Lucebox subscription and download a task-specific code model from the
selected model library; it's pre-quantized and benchmarked for the same
hardware profile, and it's live in under a minute. Updates to the software
stack arrive as tested releases — applied in one click with no driver breakage.

## Target Market

| Attribute | Description |
|-----------|-------------|
| Who | ML engineers, AI-augmented software developers, and researchers who run or want to run local LLMs — Ollama users today who want less setup and more throughput |
| Pain | Existing stacks require choosing hardware with little guidance, tuning quantization and context length by hand, and fixing regressions when a model, driver, or runtime updates |
| Current Solution | Mac Studio + Ollama, custom NVIDIA PC + llama.cpp/Ollama, or cloud APIs despite data-egress and cost concerns |
| Why They Switch | Cloud subscription prices are subsidized today and unlikely to hold; variable token costs plus compliance risk from data egress make hedging hard. Lucebox is fixed-cost and air-gapped — plus it removes throughput limits and setup churn |

## Key Value Propositions

| Value Proposition | Customer Benefit |
|-------------------|------------------|
| Tested hardware + tuned software | Faster, more consistent throughput for target workloads — less manual tuning |
| One-click model management with memory-aware quantization | Pull, configure, and serve models without using a CLI; the box selects the right format for available RAM and VRAM |
| OpenAI-compatible API + native coding-agent harness | Generic clients work via the OpenAI-compatible endpoint; the native Lucebox harness delivers a tuned local coding-agent workflow optimized for the hardware |
| Subscription model: curated optimized models | Subscribers get access to a library of hand-tuned foundation and task-specific models validated on Lucebox hardware; community tier runs open GGUF models |
| Tested releases | Software updates are validated against the same hardware before shipping; no driver breakage |
| Fixed cost + air-gapped operation | One-time hardware purchase removes subscription repricing risk; fully offline operation satisfies data-sovereignty requirements without sending traffic through API brokers |
| Desktop footprint | Production-grade inference in a backpack-portable chassis, not a rack |

## Success Definition

| Metric | Target |
|--------|--------|
| Inference throughput (Qwen3.6-27B Q4_K_M, launch hardware profile) | T1 ≥120 tok/s short-context; T2 ≥60 tok/s at 128K with prefix caching; T3 ≥60 tok/s at 128K with PFlash |
| Time to first inference (unboxing to first response) | ≤10 minutes for 90% of first-time customers |
| Net Promoter Score (technical users, 90-day survey) | ≥50 |
| Tested deployable models at launch | ≥20 |
| Repeat hardware orders (existing software users ordering a second box) | ≥15% of year-1 software install base |

## Why Now

Four trends have converged. First, strong open-weight models (Llama 4 Scout,
Qwen3.6, DeepSeek V4-Flash) now match GPT-4-class performance on coding and
reasoning tasks, so local inference is no longer the compromise it once was and
can replace cloud API use for most developer workloads. Second, desktop hardware
has crossed a price-performance threshold: RTX-X090-class GPUs and high-bandwidth
memory systems deliver 100–200+ tokens/second on 27B-class models at a total cost
that competes with $50–$300/month cloud API bills. Third, the major cloud AI
providers are pricing subscriptions below long-term cost to capture usage — a
structure unlikely to hold. Enterprises and heavy users face pressure toward
pay-per-token billing with no rate guarantees; hedging through OpenRouter or
discount aggregators trades cost uncertainty for data-egress, vendor, and audit
risk. A fixed-cost, air-gapped, on-premises solution removes both. Fourth, users
must choose among many local LLM stacks (llama.cpp, Ollama, vLLM, LM Studio,
Jan, Kobold) that do not control hardware — leaving room for an integrated product.
Waiting means losing that position to the first competitor that ships both.
