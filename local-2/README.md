# local-2

Qwen3.8-27B served locally by llama.cpp on port 8081.

- Model: `unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M` (16.5 GB)
- Context: 128K tokens (`CONTEXT 128K`), one slot (`PARALLEL 1`)
- KV cache: q8 for both keys and values
- Multi-token prediction (MTP) and the vision projector (mmproj) are off

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
| Q4_K_M weights | 16.5 GB |
| KV cache, 128K tokens at q8 | about 4.5 GB |
| Linear-layer state, per slot | about 0.15 GB |
| llama.cpp compute buffers | about 1-2 GB |
| Total | about 22-23 GB |

That leaves 6-7 GB for macOS and other apps. 160K would add about 1.1 GB of KV
cache and 256K about 4.5 GB, so there is room to go higher if needed.

llama.cpp multiplies the context by the slot count: `CONTEXT 64K` with
`PARALLEL 4` allocates 256000 tokens of KV cache (about 9 GB), which does not
fit. Raising `PARALLEL` costs the same KV cache again per slot.

## One slot and subagents

With `PARALLEL 1`, llama-server handles one request at a time and queues the
rest. Subagents still work, but several subagents running at once are served
one after another. The slot also holds the cached prompt of one conversation at
a time, so switching between the parent agent and a subagent can mean
reprocessing the parent's prompt.

## Other environments on the same machine

`local-3` (oMLX) and `local-4` (MTPLX) load their own copy of a similar-sized
model. Only one of the three local environments fits in memory at a time.
