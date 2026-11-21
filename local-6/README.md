# Inkling-2-mini

> [!NOTE]
> This config requires the Unsloth build of llama.cpp

Download Unsloth build of llama.cpp:

```sh
curl -L -o llama.tar.gz https://github.com/unslothai/llama.cpp/releases/download/b12588-mix-c0c3de8/llama-b12588-mix-c0c3de8-bin-macos-arm64.tar.gz
mkdir -p llama.cpp
tar -xzf llama.tar.gz -C llama.cpp --strip-components=1
./llama.cpp/llama-server --version
```

Update `PATH` to prefer locally installed `llama-server`:

```sh
export PATH="$PWD/llama.cpp:$PATH"
```

Run spinloop:

```sh
spinloop serve
```
