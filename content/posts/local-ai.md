+++
date = '2026-08-16T12:00:00+07:00'
draft = false
title = 'Running Qwen3.6-35B-A3B Locally on an AMD RX 580'
description = "Fitting a 35B-parameter MoE model onto an RX 580 with GTT tuning, 48GB RAM, and a lot of command-line flags."

categories = ["projects"]
tags = ["local-ai", "llama.cpp", "amd", "qwen"]
authors = ["Experian"]
avatar = "/images/solder-profile-pic.webp"
+++

# Running Qwen3.6-35B-A3B Locally on an AMD RX 580

## Overview

I wanted to run a large language model locally because I needed an AI assistant that doesn't depend on cloud APIs. The target was a 35B-class model capable of complex reasoning and tool use. I considered Gemma 4, but Qwen3.6-35B-A3B won out because it doesn't make mistakes when making tool calls. Gemma was close, but it would hallucinate tool arguments or call the wrong functions, which is a non-starter for an assistant that's supposed to actually *do* things.

The constraint was my hardware: an AMD RX 580 with 8GB of VRAM, a consumer card from 2017 that was never meant to run 35-billion-parameter models. This is what I'm calling a "poverty setup", making something work on hardware that's way below the recommended specs, because the alternative is paying for cloud API calls.

## Hardware Constraints

The RX 580 has 8GB of VRAM, but a quantized 35B model needs more. The trick is using the GPU's Graphics Translation Table (GTT) — system RAM mapped into the GPU's address space — as overflow. AMD's driver doesn't allocate enough GTT by default, so the first step was increasing it.

To increase the GTT allocation, I put the parameters in `/etc/modprobe.d/amdttm.conf`:

```
options amdttm pages_limit=8388608
options amdttm page_pool_size=8388608
```

GTT is sized in 4 KiB pages, so the page count for `N` GiB of GTT is `(N * 1024 * 1024) / 4`. For 32 GiB, that's `(32 * 1024 * 1024) / 4 = 8388608`.

The parameters live in modprobe.d rather than the kernel command line because the DKMS driver loads early, from the initramfs — which bakes in its own copy of modprobe.d. Editing the live `/etc/modprobe.d/amdttm.conf` alone does nothing until the initramfs is rebuilt with the current config:

```bash
sudo update-initramfs -u
sudo reboot
```

Before finding this method I was modifying `amdgpu.gttsize`, which is now deprecated and makes the kernel complain on every boot:

```
amdgpu 0000:29:00.0: amdgpu: [drm] Configuring gttsize via module parameter is deprecated, please use ttm.pages_limit
```

Two traps hide in that warning. First, its advice says `ttm.pages_limit`, but my system runs the DKMS fork `amdttm`, not the kernel's `ttm` module — a `ttm.pages_limit` parameter targets a module that never loads and is silently ignored; `amdttm.pages_limit` is the one that works. Second, a stale `amdgpu.gttsize` left anywhere (grub, modprobe.d, an old initramfs) doesn't get ignored, so make sure to clean everything up before adding the config in `modprobe.d`.

After adding the parameters and rebooting, I verified the allocation with:

```bash
sudo dmesg | grep "amdgpu.*memory"
# [drm] amdgpu: 8192M of VRAM memory ready
# [drm] amdgpu: 32768M of GTT memory ready.
```

The VRAM figure is the CPU-visible aperture (the PCI BAR window), not total VRAM — here it covers the full 8GB of the card. The GPU itself always addresses all of its VRAM; the aperture only limits what the CPU can map directly.

```
┌─────────────────────────────────────┐
│  System RAM (48 GB)                 │
│  ┌───────────────────────────────┐  │
│  │ GTT (32 GB, GPU addressable)  │  │
│  │ Model weights (cold, swap)    │  │
│  └───────────────────────────────┘  │
│  KV cache, activations, buffers     │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  VRAM (8 GB, on the RX 580)         │
│  Model weights (hot)                │
└─────────────────────────────────────┘
```

The GTT approach means the GPU can address 40GB in total — 8GB of native VRAM plus the slower 32GB of GTT. With this, you get a working system that's bottlenecked by your system RAM, but it works. MoE models are what make it viable — the sparse activation means only a few billion parameters are computed per token, so CPU-resident weights stay usable. A 35B dense model would grind to a halt since every token needs all 35B parameters, though a small dense model that fits entirely in VRAM runs fine.

## Setup

The model runs via `llama.cpp`'s `llama-server`, which handles the inference. The full command line:

```bash
RADV_PERFTEST=nogttspill \
HUGGINGFACE_HUB_CACHE=/mnt/Data/huggingface \
~/Projects/llama.cpp/install/bin/llama-server \
-hf unsloth/Qwen3.6-35B-A3B-GGUF:UD-Q5_K_XL \
-ctk q8_0 -ctv q8_0 -fa on -ngl 999 -cmoe --no-ui \
--top-p 0.95 --top-k 20 --min-p 0 --presence-penalty 0 --repeat-penalty 1 --temperature 0.6 \
--reasoning-budget 3000 --reasoning-budget-message "... Thinking budget limit reached. Let's summarize and conclude now." --chat-template-kwargs '{"preserve_thinking":true}' \
--load-mode none --cors-origins localhost \
--checkpoint-min-step 4096 --ctx-checkpoints 64 -c 131072 -b 2048 -ub 2048 -np 1 \
--threads-http 1 --threads 4 --prio-batch 3 --threads-batch 8 --cpu-range 0-3 --cpu-range-batch 4-11 \
-lv 3
```

That's a lot to type, so in practice the whole thing lives as a bash alias — `launchqwen` in `~/.bash_aliases`.

Every parameter explained:

| Flag | Value | Purpose |
|---|---|---|
| `RADV_PERFTEST=nogttspill` | — | Disable RADV's GTT spill optimization, which causes crashes under load |
| `HUGGINGFACE_HUB_CACHE` | `/mnt/Data/huggingface` | Store downloaded models on a separate data drive |
| `-hf` | `unsloth/Qwen3.6-35B-A3B-GGUF:UD-Q5_K_XL` | Model from HuggingFace (unsloth's quantization), UD-Q5_K_XL |
| `-ctk` | `q8_0` | KV cache key tensor precision — q8_0 halves the cache size vs f16 with little precision cost |
| `-ctv` | `q8_0` | KV cache value tensor precision — same reasoning as `-ctk` |
| `-fa` | `on` | Flash attention — faster attention computation |
| `-ngl` | `999` | Layers to offload to the GPU — 999 means as many as possible |
| `-cmoe` | — | `--cpu-moe` — keep all MoE expert weights in the CPU. The experts are the bulk of the model's weights, which is what actually fits the model into 8GB VRAM |
| `--no-ui` | — | Don't start the web UI, just the API server |
| `--top-p` | `0.95` | Nucleus sampling threshold — higher values allow more diverse outputs |
| `--top-k` | `20` | Top-k sampling — only consider the top 20 tokens at each step |
| `--min-p` | `0` | Minimum probability threshold — 0 means no minimum |
| `--presence-penalty` | `0` | Penalize tokens based on whether they've appeared — 0 means no penalty |
| `--repeat-penalty` | `1` | Repeat penalty — 1 means no penalty |
| `--temperature` | `0.6` | Sampling temperature — lower is more deterministic |
| `--reasoning-budget` | `3000` | Maximum thinking tokens — tuned down from the default (unlimited) by experimentation: above it the model loops endlessly instead of answering |
| `--reasoning-budget-message` | — | Message injected before the end-of-thinking tag when the reasoning budget is exhausted |
| `--chat-template-kwargs` | `{"preserve_thinking":true}` | Preserve the model's thinking/reasoning output in the response |
| `--load-mode` | `none` | Model loading mode — `none` reads the file into RAM with no memory-mapping or locking (the non-deprecated replacement for `--no-mmap`) |
| `--cors-origins` | `localhost` | Allowed CORS origins — reflects the Origin header only for localhost requests, so browser clients can call the API |
| `--checkpoint-min-step` | `4096` | Minimum spacing between context checkpoints, in tokens (default is 8192) |
| `--ctx-checkpoints` | `64` | Max context checkpoints per slot — cache snapshots that make chat rollback cheap by avoiding reprocessing |
| `-c` | `131072` | Context window size — 128K tokens; beyond this prompt processing gets too slow to be usable |
| `-b` | `2048` | Batch size — number of tokens processed in parallel |
| `-ub` | `2048` | Ubatch size — the sweet spot: raising it sped up prompt processing significantly, plateauing at 2048 (slower again beyond) |
| `-np` | `1` | Number of parallel sequences — 1 for single conversation |
| `--threads-http` | `1` | HTTP server threads — 1 is enough for a single user, freeing cores for inference |
| `--threads` | `4` | Generation threads — 4 CPU threads for token generation |
| `--prio-batch` | `3` | Thread priority for batch processing — 3 = realtime (0 normal, 1 medium, 2 high) |
| `--threads-batch` | `8` | Batch processing threads — 8 CPU threads for prompt processing |
| `--cpu-range` | `0-3` | Pin generation threads to CPUs 0–3 |
| `--cpu-range-batch` | `4-11` | Pin batch threads to CPUs 4–11 |
| `-lv` | `3` | Log verbosity level — 3 for moderately detailed logs |

Most of these values came from trial and error, not from the docs. The experiment was a real workload: I asked the model to write template metaprogramming code for [potts](https://github.com/spicyPoke/potts), my C++20 task-scheduling framework (formerly TaskWeave, I renamed it because that name is everywhere and I want a somewhat unique name). With a reasoning budget above 3000 tokens, the model looped endlessly without producing an answer; at 3000 it landed on something close but still wrong. It never solved that problem — I ended up writing the code myself — but it's genuinely useful for what I actually use it for: writing tests and taking notes. The rest of the values follow the same pattern: the context window stops at 128K because prompt processing gets unusably slow beyond it, the ubatch size plateaued at 2048 (beyond that it got slower again), and the thread layout pins all 12 CPUs to inference — 4 for generation, 8 for batch processing, with batch at realtime priority — to squeeze maximum speed out of the box.

## Performance

At ~80k tokens of context:

| Metric | Value |
|---|---|
| Prompt processing | 120 tokens/s |
| Token generation | 12 tokens/s |
| GPU power draw | ~170W |

12 tokens/s is readable speed — not instant, but fast enough to not be frustrating. The prompt processing at 120 tok/s means feeding it a long context window doesn't take too long.

## Lessons Learned

1. **Consumer GPUs can run large models, but not without compromises.** The RX 580's 8GB VRAM is enough with GTT overflow, but it's definitely slower than a card with more native VRAM.

2. **Quantization matters.** The UD-Q5_K_XL quantization gives a good balance of quality and size. Going lower (Q4, Q3) saves memory but degrades quality noticeably. Going higher (Q6, Q8) costs more memory for marginal quality gains.

3. **The reasoning budget is essential.** Without it, the model's thinking phase can go on indefinitely. Setting a budget and a wrap-up message keeps the model focused and prevents token budget blowouts.

4. **Tool call reliability is non-negotiable.** I tried Gemma 4, which was close in capability, but it would hallucinate tool arguments or call the wrong functions. For an assistant that's supposed to actually *do* things, that's a dealbreaker. Qwen's tool call accuracy is what made it the right choice.

5. **Power consumption is something to keep in mind.** 170W GPU-only during inference is not something you want running 24/7. This is a desktop workload, not a server workload. Factor in the electricity cost if you plan to leave it running.

6. **SOTA models are the reviewers, not the workhorses.** The local model handles the bulk of the Pi workflow — `brainstorm`, `plan`, `execute-plan` — and SOTA cloud models only show up at the review gates: `spec-review`, `plan-review`, and `code review`. As long as the task isn't too long or too big, the local model can tackle it, and keeping the bulk of the work local means most of it runs fast, private, and free.

## References

- [Qwen3.6-35B-A3B — model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B)
- [Qwen3.6-35B-A3B — release blog post](https://qwen.ai/blog?id=qwen3.6-35b-a3b)
- [unsloth/Qwen3.6-35B-A3B-GGUF](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF) — the quantized GGUF used in this setup
- [llama.cpp — the inference engine](https://github.com/ggml-org/llama.cpp)
- [potts — the C++20 task-scheduling project used for the experiments](https://github.com/spicyPoke/potts)
- [amdgpu module parameters — kernel documentation](https://docs.kernel.org/gpu/amdgpu/module-parameters.html) — the `gttsize` parameter, deprecated in favor of the TTM pages limit
- [Increasing the VRAM allocation on AMD AI APUs under Linux](https://www.jeffgeerling.com/blog/2025/increasing-vram-allocation-on-amd-ai-apus-under-linux/) — Jeff Geerling (2025) — where the GTT tuning approach comes from
