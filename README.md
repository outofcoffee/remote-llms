# remote-llms

Spinloop configs for serving LLMs — locally on this machine, or on cloud GPUs —
and a fleet definition that observes them together.

Each directory holds one environment: a `Spinloop` file (what it serves) plus
either a `preset.ini` (how it runs, in the engine's own flag vocabulary) or
`PARAM` lines inline. The per-directory READMEs hold what each model is, how
it fits in memory and what its sampling settings are. This file is the map —
how to serve, point a coding agent at any of them, and drive the cloud.

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
the foreground; `spinloop serve --dry-run` prints the command without starting
anything. The `inkling/` config needs the Unsloth llama.cpp build on `PATH` —
see its README.

### Pointing a harness at it

```sh
spinloop harness apply    # from the directory: configure the harness
spinloop code             # launch the active harness on it
```

Or from the repo root, name the path: `spinloop harness apply qwen3.8-27b/Spinloop`.
Everything beyond this — aliases, fleet routing, flags — is documented in the
spinloop docs at [docs.spinloop.ai](https://docs.spinloop.ai).

## The fleet

`fleet.yaml` observes the cloud environments and the local daemons as one
fleet. Picking it up from the repo root:

```sh
spinloop status                   # one row per node
spinloop dashboard                # live tiled view
spinloop fleet start --node dev-1 # boot one node (or --all for the whole fleet)
spinloop fleet stop --node dev-1  # stop it again
```

`spinloop gateway` serves the whole fleet under one OpenAI-compatible endpoint.
The rest — metrics, routing, `prefer`, the gateway's flags — is in the spinloop
docs.

## Cloud environments

The cloud environments run Qwen3.8-27B — `dev-*` on llama.cpp and `vllm-1`
on vLLM. Each is its own Elastic IP, API key and config, so they run side by
side in one account. Model and tuning details live in the configs and the
`dev-*` presets.

```sh
# once per account:
spinloop cloud bootstrap        # the control plane
spinloop cloud bake             # the runner AMIs

# then, per environment, from the repo root:
spinloop fleet deploy --node dev-1
spinloop cloud start --node dev-1
spinloop code --node dev-1
spinloop cloud stop --node dev-1
```

Deploying from the config directory instead uses `spinloop cloud deploy --node
dev-1`, run inside `dev-1/` — from the repo root there's no Spinloop to read.
Re-deploying a live environment needs `--overwrite`. Weights live in the
account's S3 bucket; `spinloop cloud seed ls` lists them. Everything beyond
this — seeds, `cloud env`, idle self-termination — is in the spinloop docs.

## Keys and files

`.env` (gitignored) holds `OPENAI_API_KEY` for the gateway and any
open-ended provider; `.env.example` shows the format. Spinloop records env
references, never the key itself. Some local configs need their own key — see
the directory's README.
