# remote-llms

Spinloop configs for serving LLMs — locally on this machine, or on cloud GPUs —
and a fleet definition that observes them together.

Each directory holds one environment: a `Spinloop` file (what it serves) plus
either a `preset.ini` (how it runs, in the engine's own flag vocabulary) or
`PARAM` lines inline. The per-directory READMEs hold what each model is, how
it fits in memory and what its sampling settings are. This file is the map —
how to serve, point a coding agent at any of them, and drive the fleet.

## Prerequisites

- `spinloop` on your `PATH` — `brew install spinloop-ai/tap/spinloop`
- A coding-agent harness — `opencode` (default), `pi` or `lucinate`
- For the cloud environments: AWS credentials that resolve for this account
  (profile, SSO or env). Region is pinned to `us-east-1` by each dev Spinloop's
  `ENV` line. Bootstrap drives a CDK project, so it needs Node, `npm` or
  `pnpm` — spinloop auto-detects it.

## Getting started

### Locally

```sh
cd qwen3.8-27b          # or any other config directory
spinloop serve
```

`spinloop serve` builds the server command from the Spinloop and runs it in
the foreground. `spinloop serve --dry-run` prints the command without starting
anything. The `inkling/` config needs the Unsloth llama.cpp build on `PATH` —
see its README.

### Pointing a harness at it

```sh
spinloop harness apply          # from the directory: configure the harness
spinloop code                   # launch the active harness on it
```

Both take a path or alias as a leading argument,
`spinloop harness apply qwen3.8-27b/Spinloop`, so you can do it from the repo
root too. To pin a launch to one engine, pass `--node`:
`spinloop code --node dev-1`.

## Aliases and node names

A Spinloop can be named under a short alias — a path or directory that
stands in wherever a command takes a Spinloop:

```sh
spinloop alias -n dev-1 dev-1/Spinloop
```

A node's `name` in the fleet file resolves the same way: a registered alias,
or a same-named subdirectory beside `fleet.yaml`, which is how
`spinloop fleet deploy --node dev-1` finds `dev-1/Spinloop` from this
directory. `spinloop alias -l` lists what's registered. The registry lives in
spinloop's own config, not in this repo.

## The fleet

`fleet.yaml` lists the cloud environments (`dev-1`…`dev-4`, `vllm-1`) and the
local daemons (`glimmer`, `qwen3.8-27b`, `qwen3.8-27b-omlx`,
`qwen3.8-27b-mtplx`) as one fleet. Picking it up from the repo root:

```sh
spinloop status                   # one row per node
spinloop dashboard                # live tiled view
spinloop metrics                  # resource and token counters per node
spinloop fleet start --node dev-1 # boot one node (or --all for the whole fleet)
spinloop fleet stop --node dev-1  # stop it again
```

`spinloop gateway` serves the whole fleet under one OpenAI-compatible endpoint
(`:4000` by default), waking a node when a request asks for a model nothing is
serving. The fleet file's `gateway:` section names that address; an agent
pointed at the gateway needs only the gateway's token, as `OPENAI_API_KEY`.

## Cloud environments

Five cloud environments run Qwen3.8-27B — `dev-1`…`dev-4` on llama.cpp and
`vllm-1` on vLLM. Each is its own Elastic IP, API key and config, so they
run side by side in one account. Model and tuning details live in the configs
and the `dev-*` presets.

```sh
# once per account:
spinloop cloud bootstrap        # creates the control plane (asks to confirm; --dry-run prints the plan)
spinloop cloud bake             # bakes the runner AMIs (slow; --no-wait to background it)

# then, per environment (from the repo root):
spinloop fleet deploy --node dev-1   # derives what to serve from dev-1/Spinloop
spinloop cloud start --node dev-1    # boot it
spinloop code --node dev-1           # launch the harness on it
spinloop cloud stop --node dev-1     # shut it down now
```

Deploying from the config directory instead uses `spinloop cloud deploy --node
dev-1` (run inside `dev-1/`; `spinloop cloud deploy --node dev-1` fails from
the repo root, which has no Spinloop). `spinloop fleet deploy --help` and
`spinloop cloud deploy --help` carry the full flags.

Re-deploying a live environment needs `--overwrite`, so an update can't silently
clobber a running instance.

Weights live in the account's S3 bucket. `spinloop cloud seed ls` lists them,
`spinloop cloud seed status <seed-id>` follows one to completion, and
`--reseed` re-fetches them (~20 minutes).

`spinloop cloud env --node dev-1` prints the running endpoint's environment
variables (base URL and API key) without starting anything.

Idle instances self-terminate after the idle period. `spinloop cloud start --node
dev-1 --keep 4h` retains the instance for that start's duration, and
`spinloop cloud keep --node dev-1` defers termination for one already running.
Stopping and starting brings the instance back at its same Elastic IP.

## Keys and files

`.env` (gitignored) holds `OPENAI_API_KEY` for the gateway and any
open-ended provider; `.env.example` shows the format. Spinloop records env
references, never the key itself. Some local configs need their own key — see
the directory's README.

## Layout

```
dev-1/ … dev-4/      # cloud llama.cpp environments (Spinloop + preset.ini)
vllm-1/              # cloud vLLM environment (Spinloop + preset.ini)
qwen3.8-27b/         # local llama.cpp, MTP (Spinloop, no preset)
qwen3.6-35b-a3b/     # local llama.cpp (Spinloop, no preset)
glimmer/             # local llama.cpp with DFlash (Spinloop + preset.ini)
inkling/             # local llama.cpp on the Unsloth build (Spinloop)
inkling-unsloth/     # local Unsloth Studio (Spinloop)
qwen3.8-27b-omlx/    # local oMLX (Spinloop + preset.ini)
qwen3.8-27b-mtplx/   # local MTPLX (Spinloop + preset.ini)
fleet.yaml           # the cloud + local nodes as one fleet
.env, .env.example   # API keys — .env is never committed
```

Note that `qwen3.6-35b-a3b`, `inkling` and `inkling-unsloth` are not part of
`fleet.yaml` — they serve individually, and their READMEs carry the details.

## Spinloop file fields

| Field      | Meaning |
|------------|---------|
| `PROVIDER` | Engine — `llamacpp`, `vllm`, `omlx`, `mtplx`, `unsloth` |
| `ALIAS`    | The model name — the server's model name and the harness label |
| `MODEL`    | The model reference — llama.cpp takes `repo:QUANT` (via the preset's `hf` or a `hf` PARAM); vLLM takes an HF checkpoint id; oMLX/MTPLX use an engine-specific name |
| `CONTEXT`  | Context window in tokens |
| `OUTPUT`   | Max output tokens (default: 1/4 of `CONTEXT`) |
| `BASEURL`  | Where the harness points at the server |
| `PRESET`   | Engine-specific preset file (optional — `PARAM` lines carry the same options) |
| `PARAM`    | Engine option, as `KEY = value` — no preset needed (see `qwen3.8-27b`) |
| `PARALLEL` | Parallel slots (see the Unsloth Studio config) |
| `ENV`      | Environment variables recorded for the deploy (e.g. `AWS_REGION`) |
