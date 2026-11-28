# qwen3.8-27b

Qwen3.8-27B with MTP speculative decoding, served locally by llama.cpp on
port 8081 — Unsloth's GGUF build, with the draft head inside the weights, so
no separate draft file is downloaded — or on a cloud GPU via spinloop cloud.
Everything lives in the [`Spinloop`](Spinloop) in this directory: there is no
`preset.ini`. Each engine option is a `PARAM`, rendered into the
`llama-server` command in llama.cpp's own dialect — `fa` becomes
`--flash-attn`, `ngl` becomes `--n-gpu-layers`, and so on. `PARAM hf` names
the model in place of a `MODEL`: `serve` (and `cloud deploy`) read it the way
they read a preset's `hf`, so a Spinloop with no preset file at all still
deploys what it serves.

- Model: `unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_XL` (16.5 GB)
- Context: 128K tokens (`CONTEXT 128K`)
- Output: 64K (`OUTPUT 64K`)
- KV cache: q8 for both keys and values
- MTP speculative decoding is on
- Vision projector (mmproj) is off: text-only
- No `PARALLEL` line: MTP doesn't support parallel sequences

## Run

```sh
spinloop serve              # from this directory; reads ./Spinloop
spinloop serve --dry-run    # print the llama-server command without running it
```

`spinloop serve` builds the command from the Spinloop's PARAMs, so it runs
equivalently to (with `--ctx-size` and the bind set by the Spinloop):

```sh
llama-server \
  --hf-repo unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_XL \
  --no-mmproj --n-gpu-layers 99 --flash-attn on --jinja \
  --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.0 \
  --repeat-penalty 1.0 --presence-penalty 0.0 \
  --spec-type draft-mtp --spec-draft-n-max 2 \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  --alias qwen3.8-27b --ctx-size 128000 \
  --host 127.0.0.1 --port 8081
```

On first run `llama-server` downloads the `UD-Q4_K_XL` weights into the
llama.cpp cache (`~/.cache/llama.cpp`) and then starts serving.

### Check it's up

```sh
curl http://127.0.0.1:8081/v1/models
```

## Memory

Sized for a 32 GiB Apple Silicon machine, where the GPU can use about 29 GiB
(`sysctl iogpu.wired_limit_mb` reports 29696). The figures below are estimates
from the model's config and file size, not measurements.

Only 16 of the model's 64 layers keep a KV cache that grows with context. The
other 48 are linear-attention layers with a small fixed state. At q8 the KV
cache costs about 35 KB per token (16 layers x 2 x 4 KV heads x 256 dims x
1.0625 bytes).

| Part | Size |
|---|---|
| Q4_K_XL weights | 16.5 GB |
| KV cache, 128K tokens at q8 | about 4.5 GB |
| Linear-layer state, per slot | about 0.15 GB |
| llama.cpp compute buffers | about 1-2 GB |
| Total | about 22-23 GB |

That leaves 6-7 GB for macOS and other apps. 160K would add about 1.1 GB of KV
cache and 256K about 4.5 GB, so there is room to go higher if needed. The
MTP draft head is inside the weights, so it is covered by the Q4_K_XL figure
above rather than needing its own line.

Because there is no `PARALLEL` line, the server runs a single slot and queues
the rest. Subagents still work, but several running at once are served one
after another. The slot also holds the cached prompt of one conversation at a
time, so switching between the parent agent and a subagent can mean
reprocessing the parent's prompt.

## What the PARAMs do

- `hf` — model repository and quant tag, fetched from Hugging Face.
- `no-mmproj` — skip the vision projector that `-hf` downloads automatically.
- `ngl = 99` — offload all layers to the GPU. Lower it (or drop it) for
  CPU-only or limited VRAM.
- `fa = on` — flash attention, which also keeps the quantised KV cache an
  option.
- `jinja` — use the model's built-in chat template, written bare because the
  flag takes no value. Required for Qwen3 tool calling to work correctly.
- `temp = 1.0`, `top-p = 0.95`, `top-k = 20`, `min-p = 0.0`,
  `repeat-penalty = 1.0`, `presence-penalty = 0.0` — Qwen's recommended
  sampling parameters for *thinking* mode (the model's default). llama.cpp's
  own defaults (temp 0.8, top-k 40, min-p 0.05, repeat-penalty 1.1) differ
  from these, so they are set explicitly here rather than left implicit.
  Non-thinking mode wants a different set — see the model card.
- `spec-type = draft-mtp` — speculative decoding with the model's own
  multi-token-prediction head. Worth roughly 1.5–2× on decode;
  `llama-server` logs the acceptance rate.
- `spec-draft-n-max = 2` — how many draft tokens the head proposes per step.
- `cache-type-k = q8_0`, `cache-type-v = q8_0` — q8 KV cache. This is a hybrid
  SSM/attn model, so only ~1 layer in 4 holds a growing KV cache.
- `CONTEXT 128K` — the context window in tokens, rendered as `--ctx-size`.
  The model natively supports 262144, so raise it on a box with the memory.
- `OUTPUT 64K` — override the default 1/4 output allowance. Keep it in step
  with `CONTEXT`.

## Point opencode at it

`llama-server` speaks the OpenAI-compatible API, which is exactly what the
`llamacpp` provider targets. Apply the [`Spinloop`](Spinloop) in this
directory:

```sh
spinloop harness apply qwen3.8-27b/Spinloop
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
spinloop cloud deploy --node qwen3.8-27b     # from this directory
eval "$(spinloop cloud start --node qwen3.8-27b --print-env)"   # boots (~10 min cold)
spinloop harness open --node qwen3.8-27b     # work
spinloop cloud stop --node qwen3.8-27b       # done — shut it down now
```

The deploy reads the PARAMs the same way `serve` does — `hf` names the
weights, the rest become the server's flags — and drops the settings the
cloud owns (the bind, `ctx-size`, …) from them, exactly as it would from a
preset.

## Other environments on the same machine

`qwen3.8-27b-omlx` (oMLX) and `qwen3.8-27b-mtplx` (MTPLX) load their own copy
of a similar-sized model. Only one of the local qwen3.8-27b environments fits
in memory at a time.
