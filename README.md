<p align="center">
  <img src="Sources/MferenceApp/Mac/Resources/mference-app-icon.png" alt="Mference app icon" width="160">
</p>

<h1 align="center">Mference</h1>

<p align="center">
  <strong>Big MoE models in "Small" GB of RAM</strong><br>
  A Swift + Metal inference engine for any Apple Silicon Mac, even the 8 GB ones.
</p>

<p align="center">
  <img alt="Swift 6.1 or later" src="https://img.shields.io/badge/Swift-6.1%2B-F05138?logo=swift&logoColor=white">
  <img alt="Metal 3 or later" src="https://img.shields.io/badge/Metal-3%2B-5E5CE6">
  <img alt="macOS 15 or later" src="https://img.shields.io/badge/macOS-15%2B-000000?logo=apple&logoColor=white">
  <a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/License-MIT-2ea44f"></a>
</p>

<p align="center">
  <a href="#try-it">Quick start</a> ·
  <a href="docs/OPENAI_SERVER.md">Local server</a> ·
  <a href="docs/BENCHMARKS.md">Benchmarks</a> ·
  <a href="docs/COMMUNITY_BENCHMARKS.md">Contribute results</a> ·
  <a href="docs/SYSTEM_DESIGN.md">How it works</a> ·
  <a href="#acknowledgments">Acknowledgments</a>
</p>

<p align="center">
  <strong>Qwen 3.6 on a 24 GB M5: 23.5–29.3 tok/s decode · 2.20× faster long-prompt prefill</strong><br>
  <strong>Inkling-Small on the same host: 3.0–3.7 tok/s decode · 16.4% faster with native top-6 Metal</strong>
</p>

---

> [!IMPORTANT]
> **This is a fork** of [NeelM0906/Mference](https://github.com/NeelM0906/Mference)
> carrying three small, opt-in patches — documented right here so nobody has to
> diff the source to find out what changed. `main` tracks upstream untouched;
> the patches live on the [`patches-locais`](../../tree/patches-locais) branch.

## What this fork changes

**1. Server runtime tuning via environment variables** — `MferenceServer` gains
the knobs `MferenceCLI` already has:

| Variable | Effect | Upstream behaviour |
|---|---|---|
| `MFERENCE_TRUST_RECEIPT=1` | Verify experts by size + trusted receipt | Full SHA-256 of every expert on first touch, inside prefill |
| `MFERENCE_RDADVISE=adaptive` | Expert read-ahead policy (`off`/`default`/`bounded`/`adaptive`) | Always off |
| `MFERENCE_PREFILL_CHUNK=512` | Prefill chunk size in tokens | Fixed 128 |
| `MFERENCE_EXPERT_SLOTS=64` | Routed-expert cache slots per layer | Family default (96 for Qwen 3.6 on ≥24 GiB) |
| `MFERENCE_EXPERT_CACHE_POLICY=lru` | Expert eviction policy (`lfu`/`lru`) | Always LFU (the Mac app exposes this; the server didn't) |
| `MFERENCE_TOOL_STRICT=1` | Reject tool calls whose name is not in the request's tool list (500) | Always strict |

All default to upstream behaviour when unset — except tool-name validation,
which is now lenient by default: an unknown tool name is emitted as a normal
tool call for the client to refuse, instead of failing the whole stream with
a 500. The strict behaviour left agent sessions permanently broken — every
retry replayed the same history and the model re-emitted the same unknown
call (measured 2026-08-28: three retries, three identical
`unknownTool("process")` 500s). Set `MFERENCE_TOOL_STRICT=1` to restore the
upstream behaviour.

*LFU vs LRU, measured* — server at 32k context, 96 slots, machine under normal
interactive use; three ~5k-token tasks from different domains per policy, fresh
process per policy, first task cold, next two warm; 600-token decode measured
via streaming TTFT split. A repeated cold LFU run at the end landed within 0.6%
of the first, validating the round:

| Task | LFU decode | LRU decode | LFU edge |
|---|---|---|---|
| Tech docs (cold start) | 14.2 tok/s | 11.9 tok/s | **+19%** |
| Meeting transcript (warm) | 16.6 tok/s | 14.9 tok/s | +11% |
| Source code (warm) | 17.0 tok/s | 15.5 tok/s | +9% |

LFU wins even from a cold start because the frequency profile forms during the
task's own prefill (a 5k-token prompt routes ~40k expert activations per layer
before the first output token). Warm prefill was also consistently faster under
LFU (62–74 s vs 76–87 s) — the carried cache retained more reusable experts
across task switches. Upstream's LFU default holds; the knob exists for other
workloads.

*Measured* — same ~2.8k-token prompt, back-to-back on an M4 24 GB, Qwen 3.6
35B-A3B, 64-token completions, direct against the server:

| Request | Upstream defaults | Tuned (vars above, 64 slots) | Speedup |
|---|---|---|---|
| First after start (expert verification + cold prefill) | 93.0 s | 44.7 s | **2.1×** |
| Warm prefill, different prompt | 70.0 s | 39.6 s | **1.8×** |

The tuned column runs with *fewer* slots than upstream's default (64 vs 96) and
still wins — the gains come from receipt-based verification, adaptive rdadvise
and the larger prefill chunk.

Includes a fix for the slot knob: the value reached `RuntimeConfiguration` (and
the prompt-cache digest) but not the expert streamer, so the arena stayed at
the family default regardless — verified by watching the per-layer slab size
respond (96 → 162 MiB, 64 → 108 MiB).

This fork also widens the allowed slot values with 160, 192 and 256 for hosts
with the RAM to use them (upstream's whitelist stops at 128).

*Why expose the slot count at all?* Full sweep with the repo's own
`run-benchmark.sh` (fresh process per run, discarded warmup, 2 measured reps
per case, one back-to-back chain on an M4 24 GB under normal interactive use;
free memory was 48–71% at each config start, and a repeated 64 at the end of
the chain landed within ~7% of the first, so the round is internally
consistent):

| Slots per layer | decode short | decode medium | decode long | Long prefill | Peak RSS (CLI, 4k ctx) | Expert arena (40 layers) |
|---|---|---|---|---|---|---|
| 32 | 14.0 | 11.7 | 9.7 | 49.6 s | 2.1 GiB | 2.2 GB |
| 64 | 20.1 | 17.7 | 13.9 | 55.8 s | 3.8 GiB | 4.3 GB |
| 96 (upstream default) | 21.0 | 19.7 | 18.1 | 54.7 s | 4.8 GiB | 6.5 GB |
| 128 | 22.9 | 22.6 | 19.4 | 58.5 s | 5.0 GiB | 8.6 GB |
| `resident` | 13.9 | 9.7 | **4.9** | **427.9 s** | 0.3 GiB | 0 (file-backed) |
| 64 again (sentinel, end of chain) | 19.0 | 17.8 | 14.9 | 53.7 s | 3.9 GiB | 4.3 GB |

Slot scaling is monotonic with diminishing returns: 64→96 buys +30% on
long-form decode, 96→128 only +7%. And a warning worth a fork README:
**`resident` collapses on RAM-constrained hosts.** The 17.7 GB Qwen 3.6 expert
pool cannot fit in page cache on a 24 GB machine, so every cold expert arrives
as ~16 KB page faults with none of the pread path's readahead — 7.8× slower
long-prompt prefill and 4× slower decode, even though its measured footprint
is the smallest of the table (file-backed clean pages barely register as RSS).
`resident` is only the right call where the pool actually fits in RAM.

**Round 2 — same protocol, quiet machine (overnight, free memory 68–86%),
extended to the values this fork unlocks:**

| Slots per layer | decode short | decode medium | decode long | Long prefill | Peak RSS |
|---|---|---|---|---|---|
| 32 | 17.9 | 16.0 | 12.6 | 49.0 s | 2.1 GiB |
| 64 ¹ | 23.2 | 22.6 | 18.8 | 54.5 s | 4.0 GiB |
| 96 | 24.3 | 23.8 | 19.4 | 52.1 s | 4.8 GiB |
| 128 | 25.9 | 24.7 | 21.2 | 54.3 s | 4.9 GiB |
| **160** | **27.4** | **28.3** | **23.7** | 51.3 s | 5.3–5.8 GiB |
| 192 ² | — | — | 17.6 | **381.1 s** | — |
| 256 ² | — | — | **DNF** | — | — |
| `resident` | 20.1 | 20.6 | 9.8 | 413.5 s | 0.3 GiB |

¹ Sentinel run at the end of the chain; the chain's first 64-slot config hit
ambient interference (long-case spread 10.4–16.5 tok/s vs ≤0.8 everywhere
else) and was discarded — that is what sentinels are for.
² Single guarded long-synthesis probes, outside the full protocol. 192
completed but its 13 GB arena already exceeds comfortable RAM: prefill took
381 s (7× the slot-mode norm) and decode trailed 128. 256 (17.3 GB arena) did
not finish within a 25-minute cap. On 24 GB, the cliff is right after 160.

Two findings the two rounds give jointly: a quiet machine lifts every
configuration (the page cache absorbs slot misses — compare 96: 19.4 vs 18.1,
or 64: 18.8 vs 13.9), and the optimum moves with co-located memory use.
**160 is the overnight/dedicated champion; 96–128 the balanced daily pick;
64 the defensive choice when heavy jobs share the RAM; `resident` and ≥192
belong to bigger hosts.**

Server-side (32k context), the writable set measured 7.4 GB at 96 slots vs
4.4 GB at 64 — on a 24 GB host that also runs Docker and friends, that
difference is what keeps the server responsive under system memory pressure.
This fork currently runs 64 in production.

**2. Optional thinking mode for Qwen 3.6** — the checkpoint's own chat template
opens a live `<think>` block, but Mference pins the family as non-thinking.
`MFERENCE_QWEN36_THINKING=1` switches the prompt suffix and the stop token
together, per family, the way the eos/end-of-turn contract expects. Off by
default: in our workload (meeting-minutes generation) it made generations
several times longer (informally: ~650 s vs ~150 s for comparable analyses)
without improving our fact-checklist scores.

**3. Repack disk check counts APFS purgeable space** — `statfs` ignores
purgeable space on macOS: on a 24 GB Mac with ~30 GB purgeable, repacks refused
to start despite the space being reclaimable. The check now also consults
`volumeAvailableCapacityForImportantUsage` and takes the larger value.

---

Mixture-of-experts models activate only a few billion (cough) parameters per token.
Mference builds on that: it keeps each model's shared core and KV cache in
memory, then streams just the experts chosen for each token from SSD. The
model never has to fit in RAM — only its working set does.

Mference currently runs five pinned instruction checkpoints:

- **[Gemma 4 26B-A4B](https://ai.google.dev/gemma/docs/core/model_card_4)** —
  26B total, ~3.88B active per token, in ~2 GB of memory.
- **[Qwen 3.6 35B-A3B](https://huggingface.co/Qwen/Qwen3.6-35B-A3B)** — 35B
  total, ~3B active per token, from ~1.45 GB of memory with the 16-slot profile.
- **[DeepSeek-V4-Flash 284B-A13B](https://huggingface.co/mlx-community/DeepSeek-V4-Flash-2bit-DQ)**
  *(experimental)* — 284B total, ~13B active per token, from
  the 2-bit dynamic-quant checkpoint (2-bit experts, 4-bit core). Budget: ~6.8 GB peak at the 8-slot floor, ~91 GB on  disk;
- **[Inkling-Small 276B-A12B](https://huggingface.co/pipenetwork/Inkling-Small-MLX-4bit)** —
  276B total, ~12B active per token, in ~9 GB of memory.
- **[Maple Preview](https://huggingface.co/deepgrove/maple-preview-2bit-mlx)** —
  a 20B total, ~1B active per token, from the ternary (1.58 bit per parameter) quantization, using ~645 MiB
  of memory. Its chat template opens a live `<think>` reasoning block, so give
  it a generous max-token allowance; an approximate FlashHead decode head is an
  opt-in via `--flash-head`.
- **[Qwen 3.8 27B](https://huggingface.co/mlx-community/Qwen3.8-27B-4bit)** —
  the first dense family: 27B parameters, all active, text-only port of the
  multimodal checkpoint, fully resident in ~15 GB (24 GB Macs). Ships MTP
  speculative decoding with byte-identical greedy output — 15.0 tok/s decode
  on a 24 GB M5 (2.35× mlx-vlm on the same checkpoint). Its chat template
  also opens a live `<think>` block.
  
The runtime, streaming installer, CLI, native Mac app, and loopback
OpenAI-compatible server are written in Swift and Metal. Mference is
model-specific rather than a wrapper around MLX or llama.cpp: each
architecture is enumerated explicitly, with its own pinned checkpoint,
compile-time baseline, and manifest contract. New families merge through the
[family acceptance gate](docs/FAMILY_GATE.md).

## Try it

```bash
git clone https://github.com/NeelM0906/Mference.git
cd Mference
swift build -c release
.build/release/MferenceMac
```

When the app opens, choose **Download** and let Mference fetch and repack the
pinned model — or choose **Choose Existing Model…** to point at a `.gturbo`
directory already on disk. Once it is ready, choose **Load Model**, type your
prompt, and press **Generate**. The app installs Gemma 4 by default; select
Qwen 3.6 or Maple with `defaults write Mference model qwen36` or
`defaults write Mference model maple` (or the matching `MFERENCE_MODEL` value)
before launching.

Each chat keeps its own multi-turn history in a collapsible left sidebar
(<kbd>Command</kbd>+<kbd>N</kbd> for a new chat,
<kbd>Control</kbd>+<kbd>Command</kbd>+<kbd>S</kbd> to toggle the sidebar).
History is written locally next to the model directory and rendered through the
installed model's own chat template, so both Gemma 4 and Qwen 3.6 see their
native dialect. The composer can extract text locally from PDF, DOCX, PPTX, and
XLSX files; nothing is uploaded. Extraction is bounded, so very long documents
are trimmed and marked as truncated. The attachments on one prompt hold at most
750,000 characters of extracted text in total — all a single request can carry —
and a file that would exceed that is refused with a message rather than silently
dropped. Before a turn is committed, the app measures the rendered conversation
with the model tokenizer; when older turns no longer fit the selected context,
it compresses them into a rolling per-chat memory while the full transcript
stays visible. Turns folded into that memory then release their hidden
request-side copy — attached document text included — because the summary
stands in for them from that point on; nothing visible changes. Chats and their
transcripts are never evicted automatically, so delete chats you no longer want
kept. Appearance follows System, Light, or Dark from the **Appearance** menu.

From the command line:

```bash
# Install a model (streams and repacks; never materializes the full checkpoint)
swift run -c release MferenceRepack --model qwen36 --output scratch/qwen36.gturbo

# Generate
swift run -c release MferenceCLI \
  --model scratch/qwen36.gturbo \
  --prompt "The capital of France is" \
  --max-new 64
```

## At a glance

| Metric | Value |
| --- | --- |
| Models | Gemma 4 26B-A4B IT (26B total, ~3.88B active) · Qwen 3.6 35B-A3B (35B total, ~3B active) · DeepSeek-V4-Flash 284B-A13B (experimental) · Inkling-Small 276B-A12B (276B total, ~12B active) · Maple Preview (20B total, ~1B active) · Qwen 3.8 27B (dense, MTP speculative decode) |
| Weights | MLX affine or ternary, group 64/128; INT8 or BF16 routers; 4-bit or 2-bit routed experts |
| Memory | ~2 GB (Gemma 4) · ~1.45 GB at 16 slots (Qwen 3.6; CLI/server auto uses 96 slots on 24 GiB+ hosts, 32 on 16 GiB+) · est. ~6.8 GB (DeepSeek-V4-Flash) · ~9 GB (Inkling-Small), including a 4K KV cache · 490.64 MiB (Maple, 128-token prompt) |
| Storage | ~14.3 GB installed (Gemma 4) · ~19.6 GB (Qwen 3.6) · ~91 GB (DeepSeek-V4-Flash) · ~148 GB (Inkling-Small) · ~6.6 GB (Maple) |
| Hardware | Apple Silicon Mac; 8 GB of RAM |
| Platform | macOS 15+, Metal 3 (MSL 3.2), Swift 6.1+; running on macOS 26 with an Apple10 GPU adds the Metal 4 tensor-ops prefill path |
| Measured decode, Gemma 4 | 5.1–6.3 tok/s (8 GB M2 Air) · 31–35 tok/s (24 GB M5 Pro) |
| Measured decode, Qwen 3.6 | 23.5–29.3 tok/s (24 GB M5, 32-slot profile; auto now selects 96 slots on that host) |
| Measured decode, DeepSeek-V4-Flash | 5.3–6.1 tok/s (256 GB M3 Ultra) at a 5,671–5,679 MiB peak footprint |
| Measured decode, Inkling-Small | 3.0–3.7 tok/s (24 GB M5, optimized native top-6 path) · 5.3–6.9 tok/s (256 GB M3 Ultra) at an 8,936–8,939 MiB peak footprint |
| Measured, Maple Preview | Exact head: 18.9–24.6 tok/s decode, 25.1–44.9 tok/s prefill, and 491–1,211 MiB peak process footprint on 128-8192 context (16 GB M4) |
| Measured, Qwen 3.8 27B | 15.0 tok/s decode (MTP speculative, byte-identical; 7.9 plain) · ~60 tok/s prefill (24 GB M5); mlx-vlm on the same checkpoint: 6.41 decode / 40.5 prefill |

Qwen 3.6 numbers follow the frozen
[community benchmark protocol](docs/COMMUNITY_BENCHMARKS.md) — three fixed
prompts and seeds, one discarded warmup, measured runs in fresh processes, and
every run reaching a natural end of turn. The optimized short, medium, and long
cases decode at 29.293, 27.460, and 23.470 tok/s. Their outputs are byte-identical
to matching 16-slot controls, making the model-aware 32-slot rung worth an
18.1% geometric-mean decode gain. Long-prompt prefill fell from 58.45 to 26.54
seconds, a 2.20× speedup. With the GPU-resident slot map, auto has since moved
24 GiB-class hosts to a 96-slot rung. Hosts below 16 GiB and the Mac app retain
the 16-slot memory-first path. See [Benchmarks](docs/BENCHMARKS.md) and the
[Qwen 3.6 performance notes](docs/QWEN36_PERFORMANCE.md) for exact commands,
token counts, memory behavior, and rejected experiments.

Inkling-Small uses its own six-expert INT4 Metal pipeline rather than padding
the router result to eight experts. Resident-expert phase 1 runs while cache
misses stream from SSD; across the same frozen short, medium, and long cases,
decode improved from 2.909/2.961/2.819 to 3.434/3.670/3.038 tok/s. That is a
16.4% geometric-mean gain, with byte-identical generated output in every A/B.
See the [Inkling performance notes](docs/INKLING_SMALL.md#native-top-6-decode-2026-08-06).

# Products

| Product | Purpose |
| --- | --- |
| `Mference` | Swift library containing the runtime and Metal kernels |
| `MferenceMac` | Native Mac app for installation and generation |
| `MferenceDecodeService` | One-shot local model and Metal owner used by the Mac app |
| `MferenceCLI` | Command-line instruction chat and raw completion |
| `MferenceServer` | OpenAI-compatible Chat Completions server, on loopback by default or a Tailnet address with `--bind tailnet` |
| `MferenceRepack` | Streaming model installer and install verifier |

Only one model-owning product should run at a time. The server selects the
installed model's native dialect automatically, including Gemma's chat format,
Qwen's ChatML template with `<tool_call>` function calls, and Maple's ChatML
template with hidden reasoning.

### Requirements

- An Apple Silicon Mac (arm64 only)
- macOS 15 or later, with Metal 3; Xcode 16.3 and Swift 6.1 or newer
- Free storage for the model install (~6.6 GB Maple, ~14.3 GB Gemma 4, ~19.6 GB Qwen 3.6; the largest families require substantially more)
- An internet connection for the first install

The shader library is compiled from source at startup, and the choice of
shading-language version is made then, not at build time. A single binary
therefore covers both worlds: *running* on macOS 26 with an Apple10 GPU
compiles at MSL 4.0 and enables the Metal 4 tensor-ops prefill kernels, while
every other supported configuration compiles at MSL 3.2 and selects the
non-tensor kernels automatically. Both paths run the full Gemma 4 and Qwen 3.6
feature set; they agree to within kernel tolerance rather than bit-exactly, so
expect the same quality but not identical sampled tokens.

## How it works

The installer streams bounded byte ranges from the pinned Hugging Face
revision and repacks them directly into an on-disk layout (`.gturbo`) built
for per-expert reads: resident tensors in one mapped file, and each layer's
routed experts as fixed-stride, page-aligned blobs. At generation time the
runtime keeps the common weights mapped read-only, holds a small per-layer LFU
expert cache, and `pread`s only the experts each layer's router selects for the
current token. Inkling dispatches six experts; Gemma, Qwen, and Maple dispatch
eight.
The memory-first path uses 16 slots; CLI and server auto select larger rungs
for Qwen — 96 slots on hosts with at least 24 GiB, 32 with at least 16 GiB —
because its 256 experts per layer benefit measurably from the added coverage.
Inkling remains at 16 because a 24-slot control warmup on the 24 GB M5 entered
memory pressure and regressed sharply. Each layer's slots share one contiguous
wired buffer, and on Qwen a GPU-resident expert-to-slot map lets layers whose
eight experts are all cached run their routed branch from pre-encoded,
GPU-guarded commands, with no CPU expert planning or fetching;
the routed command buffer also commits eagerly, gated on a shared event that
fires as expert fills land. An explicit `resident` mode that maps every layer
file once exists as an opt-in, but it measured slower than the slot rungs under
page-cache pressure.

Qwen 3.6's linear-attention layers replace KV storage entirely: each keeps a
2 MiB delta-rule state and a 3-row convolution tail, updated in place every
token. The gated-DeltaNet kernels are validated against a CPU reference,
including the guarantee that a chunked prefill of T rows matches T sequential
decode steps through the same kernels.

The full design, the memory budget, and the experiment record that shaped the
engine are in [System design](docs/SYSTEM_DESIGN.md) and the
[optimization journey](docs/OPTIMIZATION_JOURNEY.md).

## Roadmap

- More explicitly pinned architectures without a generic-model fallback
- Extend resident-expert compute/I/O overlap to every model family and prefill
- Per-model quality validation (KLD against reference implementations)
- Longer contexts, vision towers, and a hardware benchmark matrix

## Acknowledgments

Mference is heavily inspired by — and its foundation is derived from —
**[TurboFieldfare](https://github.com/drumih/turbo-fieldfare)** by Andrey
Mikhaylov, which pioneered running Gemma 4 26B in ~2 GB on Apple Silicon and
documented over a hundred experiments behind its design. Those derived
portions are licensed under the [Apache License 2.0](LICENSE-APACHE); see
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for details. New Mference
code is [MIT-licensed](LICENSE).

Model weights remain subject to their own terms: the
[Gemma 4 license](https://ai.google.dev/gemma/apache_2) and the
[Qwen license](https://huggingface.co/Qwen/Qwen3.6-35B-A3B/blob/main/LICENSE).
Maple's pinned checkpoint declares no license; establish the necessary rights
before downloading, using, or redistributing it. Maple's MLX-derived kernel
work is covered by [LICENSE-MLX](LICENSE-MLX); see
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
