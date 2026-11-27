# Inkling-2-mini

Text-only, served by llama.cpp (Unsloth build) as `inkling-2-mini` on
`http://127.0.0.1:8081/v1`. The vision projector is intentionally skipped
(`--no-mmproj`) to save ~0.6 GB of RAM on a 32 GB machine. Image inputs
are not served here.

## Prerequisite: the Unsloth build of llama.cpp

This config requires the Unsloth build (looped-layer support + the DSpark
drafter), not the upstream/Homebrew build on PATH. The build lives in
`../llama-inkling/llama.cpp`, which is gitignored (from the
`b12588-mix-c0c3de8` tarball). Verify it:

```sh
../llama-inkling/llama.cpp/llama-server --version
# version: 1.0.0-dev (build 12588, ...)
```

If it's missing, download it:

```sh
mkdir -p ../llama-inkling/llama.cpp
curl -L -o ../llama-inkling/llama.tar.gz https://github.com/unslothai/llama.cpp/releases/download/b12649-mix-c7059d6/llama-b12649-mix-c7059d6-bin-macos-arm64.tar.gz
tar -xzf ../llama-inkling/llama.tar.gz -C ../llama-inkling/llama.cpp --strip-components=1
../llama-inkling/llama.cpp/llama-server --version
```

## Run

Prefer the Unsloth build on `PATH`, then:

```sh
export PATH="$PWD/../llama-inkling/llama.cpp:$PATH"
spinloop serve
```

`spinloop serve` shells out to `llama-server` on PATH, so the Unsloth build
must come first. The served model is text-only, and the DSpark drafter
(`Q8_0`, ~0.4 GB) is auto-downloaded and found at the repo root — no
`model-draft` line is needed.
