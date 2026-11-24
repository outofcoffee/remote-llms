# qwen3.6-35b-a3b

Qwen3.6-35B-A3B, a sparse MoE with 3B active parameters out of 35B, served
locally by llama.cpp on port 8083 — Unsloth's GGUF build — or on a cloud GPU
via spinloop cloud. Everything lives in the [`Spinloop`](Spinloop) in this
directory: there is no `preset.ini`. Each engine option is a `PARAM`, rendered
into the `llama-server` command in llama.cpp's own dialect — `fa` becomes
`--flash-attn`, `ngl` becomes `--n-gpu-layers`, and so on. `PARAM hf` names
the model in place of a `MODEL`: `serve` (and `cloud deploy`) read it the way
they read a preset's `hf`, so a Spinloop with no preset file at all still
deploys what it serves.

The sampling parameters come from the model's Hugging Face page, not from the
qwen3.8-27b example — thinking mode has a different recommended set. The
defaults below are the page's thinking-mode set for precise coding tasks; the
general-thinking alternative is documented in [What the PARAMs do](#what-the-params-do).

- Model: `unsloth/Qwen3.6-35B-A3B-GGUF:UD-Q4_K_XL` (~22.4 GB)
- Context: 128K tokens (`CONTEXT 128K`) — the model page asks for at least
  128K to keep thinking intact (native max 262144)
- Output: 64K (`OUTPUT 64K`)
- KV cache: q8 for both keys and values
- Vision projector (mmproj) is off: text-only
- No MTP speculative decoding: the model page recommends MTP for vLLM/SGLang,
  not llama.cpp, and the draft head is only in the `*-MTP-GGUF` build. See
  the note in [What the PARAMs do](#what-the-params-do).

## Run

```sh
spinloop serve              # from this directory; reads ./Spinloop
spinloop serve --dry-run    # print the llama-server command without running it
```

`spinloop serve` builds the command from the Spinloop's PARAMs, so it runs
equivalently to (with `--ctx-size` and the bind set by the Spinloop):

```sh
llama-server \
  --hf-repo unsloth/Qwen3.6-35B-A3B-GGUF:UD-Q4_K_XL \
  --no-mmproj --n-gpu-layers 99 --flash-attn on --jinja \
  --temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --repeat-penalty 1.0 --presence-penalty 0.0 \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --alias qwen3.6-35b-a3b --ctx-size 128000 \
  --host 127.0.0.1 --port 8083
```

On first run `llama-server` downloads the `UD-Q4_K_XL` weights into the
llama.cpp cache (`~/.cache/llama.cpp`) and then starts serving.

### Check it's up

```sh
curl http://127.0.0.1:8083/v1/models
```

## Memory

Sized for a 32 GiB Apple Silicon machine, where the GPU can use about 29 GiB
(`sysctl iogpu.wired_limit_mb` reports 29696). The figures below are estimates
from the model's config and file size, not measurements.

Only 10 of the model's 40 layers keep a KV cache that grows with context —
every fourth layer is a full-attention layer, the rest are linear-attention
layers with a small fixed state. At q8 the KV cache costs about 11 KB per
token (10 layers x 2 KV heads x 256 head dims x 2 for K and V x 1.0625 bytes).

| Part | Size |
|---|---|
| Q4_K_XL weights | about 21 GiB (22.4 GB) |
| KV cache, 128K tokens at q8 | about 1.4 GB |
| Linear-layer state, per slot | about 0.15 GB |
| llama.cpp compute buffers | about 1-2 GB |
| Total | about 23-24 GB |

That leaves 6-7 GB for macOS and other apps. The model is a sparse MoE, so
the compute and buffer overheads are smaller than for a dense model of the
same parameter count. 160K would add about 0.7 GB of KV cache and 256K about
2.7 GB, so there is room to go higher if needed.

Because there is no `PARALLEL` line, the server runs a single slot and queues
the rest. A single-slot server also holds the cached prompt of one
conversation at a time, so switching between the parent agent and a subagent
can mean reprocessing the parent's prompt.

## What the PARAMs do

- `hf` — model repository and quant tag, fetched from Hugging Face.
- `no-mmproj` — skip the vision projector that `-hf` downloads automatically.
- `ngl = 99` — offload all layers to the GPU. Lower it (or drop it) for
  CPU-only or limited VRAM.
- `fa = on` — flash attention, which also keeps the quantised KV cache an
  option.
- `jinja` — use the model's built-in chat template, written bare because the
  flag takes no value. Required for Qwen tool calling to work correctly.
- `temp = 0.6`, `top-p = 0.95`, `top-k = 20`, `min-p = 0.0`,
  `repeat-penalty = 1.0`, `presence-penalty = 0.0` — Qwen's recommended
  sampling parameters for *thinking* mode on precise coding tasks (the model
  page's WebDev set), the shape of work an agent harness does. llama.cpp's
  own defaults (temp 0.8, top-k 40, min-p 0.05, repeat-penalty 1.1) differ
  from these, so they are set explicitly here rather than left implicit.
  For general (non-coding) tasks the model page recommends thinking mode
  with `temp = 1.0`, `top-p = 0.95`, `top-k = 20`, `min-p = 0.0`,
  `presence-penalty = 1.5`, `repeat-penalty = 1.0` — switch those two
  values (`temp` and `presence-penalty`) rather than the whole set. For
  non-thinking (instruct) mode the page recommends `temp = 0.7`,
  `top-p = 0.80`, `top-k = 20`, `min-p = 0.0`, `presence-penalty = 1.5`,
  `repeat-penalty = 1.0`; that mode also needs thinking disabled, which is a
  per-request chat-template kwarg (`enable_thinking: false`), not a server
  flag.
- `cache-type-k = q8_0`, `cache-type-v = q8_0` — q8 KV cache. Only ~1 layer in
  4 holds a growing KV cache in this model, so this is comparatively cheap.
- `CONTEXT 128K` — the context window in tokens, rendered as `--ctx-size`.
  The model natively supports 262144, so raise it on a box with the memory.
- `OUTPUT 64K` — override the default 1/4 output allowance. Keep it in step
  with `CONTEXT`.

### Speculative decoding (optional)

MTP is deliberately off. The model page recommends MTP for SGLang and vLLM
(`--speculative-algo NEXTN`, or a `qwen3_next_mtp` speculative config), not
for llama.cpp, and Unsloth ships the draft head only in the `*-MTP-GGUF`
build. To enable it, point `hf` at
`unsloth/Qwen3.6-35B-A3B-MTP-GGUF:UD-Q4_K_XL` and uncomment the
`spec-type` / `spec-draft-n-max` lines in the Spinloop (start with
`spec-draft-n-max = 2`). `llama-server` logs the acceptance rate once it is
running.

The page also notes that agent workflows benefit from keeping the reasoning
trace across turns — `preserve_thinking` as a chat-template kwarg, passed per
request by the harness rather than set here.

## Point opencode at it

`llama-server` speaks the OpenAI-compatible API, which is exactly what the
`llamacpp` provider targets. Apply the [`Spinloop`](Spinloop) in this
directory:

```sh
spinloop harness apply qwen3.6-35b-a3b/Spinloop
# or, from this directory:
spinloop harness apply
```

`ALIAS` is the name opencode shows for the model — for a single-model server
it's just a label, since `llama-server` serves whichever model it loaded
regardless of what's requested. `CONTEXT` matches opencode's context window
to the `--ctx-size` the server was launched with, so it doesn't overshoot
what `llama-server` will accept. `BASEURL` points the harness at this
instance (the provider's default is port 8080).

## Running it on AWS

The same Spinloop runs this model on a GPU in the cloud — provisioned by
`spinloop cloud` — rather than the machine in front of you, and terminates
itself once you stop using it. This is real, billed AWS infrastructure, so
each step shows you a plan and asks for confirmation before it creates
anything.

```sh
spinloop cloud bootstrap                     # once per AWS account: the control plane
spinloop cloud deploy --env qwen3.6-35b-a3b   # from this directory
eval "$(spinloop cloud start --env qwen3.6-35b-a3b)"   # boots (~10 min cold)
spinloop harness open --env qwen3.6-35b-a3b   # work
spinloop cloud stop --env qwen3.6-35b-a3b     # done — shut it down now
```

The deploy reads the PARAMs the same way `serve` does — `hf` names the
weights, the rest become the server's flags — and drops the settings the
cloud owns (the bind, `ctx-size`, …) from them, exactly as it would from a
preset.

## Other environments on the same machine

`qwen3.8-27b` (an MTP llama.cpp config), `qwen3.8-27b-omlx`
(oMLX) and `qwen3.8-27b-mtplx` (MTPLX) load their own copy of a model. Only
one local config fits in memory at a time on a 32 GiB machine — they do not
all run at once.
