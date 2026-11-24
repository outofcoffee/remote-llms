# Inkling-2-mini (Unsloth Studio)

The same model as [`../inkling`](../inkling), served by Unsloth Studio's
`unsloth run` instead of a hand-installed llama.cpp build. Studio ships its own
llama.cpp, so there is no `../llama-inkling` build to keep on `PATH`, and it
chooses GPU placement and speculative decoding (including the DSpark drafter)
by itself.

Served as `inkling-2-mini` on `http://127.0.0.1:8084/v1`. Text-only: the
vision projector is skipped with `no-mmproj`.

## Run

Needs spinloop 1.50.0 or later.

```sh
spinloop serve
```

## Key

Studio requires an API key on `/v1`. It prints one in its startup banner
(`Bearer sk-unsloth-…`) and reuses it on later runs. Export it before applying
the Spinloop to a harness:

```sh
export UNSLOTH_API_KEY=sk-unsloth-…
spinloop harness apply
```

Or turn on keyless access in Studio's settings and leave it unset.

## Differences from `../inkling`

- Sampling is pinned with Studio's own flags (`temperature`, `top-p`,
  `top-k`, `min-p`). They apply to requests that don't set their own values.
- Context is 80K, not 128K. Studio checks memory before loading and refuses
  a context that it estimates won't fit in unified memory. With 32 GB it
  allows up to about 80K tokens. `UNSLOTH_ALLOW_METAL_CTX_OVERCOMMIT=1`
  overrides the check, but Studio warns that the machine may then run out of
  memory.
- No `n-gpu-layers` or `jinja`: Studio sets both.
- Speculative decoding is left to Studio's `auto` mode, which uses the DSpark
  drafter when the repo ships one.
