# remote-llms

Configuration for remote hosted LLMs, using [outfit](https://github.com/lucinate-ai/outfit).

Three remote environments, each serving **Qwen3.8-27B** on llama.cpp, deployed onto
the shared account-level infrastructure. Each has its own Elastic IP, API key
and config, so all can run side by side in one AWS account.

## Why

This repo allows you to deploy Qwen to a cloud VM and connect your local AI coding agent to it.

## Prerequisites

- `outfit` on your `PATH` — `brew install lucinate-ai/tap/outfit`
- AWS credentials that resolve for this account (the standard AWS chain — profile,
  SSO, or env). Region is pinned to `us-east-1` by each Outfit's `ENV`.
- For bootstrap only: a Node runtime, including `npm` (it drives the CDK project in
  outfit's `remote/` tree).
- A coding-agent harness installed — `opencode` (the default) or `pi`.

## 1. Bootstrap the shared layer (once per account)

The shared infrastructure — Image Builder AMIs, the lifecycle Lambdas, the S3
weights bucket, roles and VPC — is deployed once and reused by every environment:

```sh
outfit remote bootstrap            # shows a plan, then asks to confirm
# outfit remote bootstrap --dry-run  # print the plan, change nothing
# outfit remote bootstrap --yes      # confirm non-interactively
```

It prints the target account, region and cost caveat before touching anything.
The AMI bake is slow and runs in the background by default — pass `--wait` to
block until it finishes, or just carry on and deploy once the AMIs are ready.
Re-running is safe and needs no override; it skips work that's already done.

You only do this **once for the account** — every environment reuses it.

## 2. Alias the outfits

All three Outfits declare the same `ALIAS` (`qwen3.8-27b`), so register them under
distinct names to tell them apart in later commands:

```sh
outfit alias -n dev-1 dev-1/Outfit
outfit alias -n dev-2 dev-2/Outfit
outfit alias -n dev-3 dev-3/Outfit

outfit alias -l                    # list what's registered
```

These names now stand in for the file paths in every command that takes one —
the `remote` control commands (`deploy`, `start`, `stop`, `status`, `stats`) as
well as `apply`, `harness` and `serve` — so the rest of this guide uses them
instead of paths. The alias registry lives in outfit's own config, not in the repo.

## 3. Deploy each environment

Deploy stands up the environment's Elastic IP, instance config, per-environment
API key and ingress rule, sets what it serves (the Outfit + preset), and registers
it under `~/.config/outfit/remotes/<env>/remote.json`. It does **not** start an
instance.

```sh
outfit remote deploy dev-1
outfit remote deploy dev-2
outfit remote deploy dev-3
```

- Ingress defaults to your current public IP as a `/32`; pass a CIDR flag to widen it.
- Redeploying an environment that's already live needs `--overwrite` (which
  `--yes` alone won't satisfy) so you can't silently clobber a running instance.

> The alias, a path (`dev-1/Outfit`), or a directory holding one all work here;
> with no argument at all a command uses `./Outfit`, so `cd dev-1 && outfit
> remote deploy` does the same thing.

## 4. Start them

Starting boots the instance on the environment's own Elastic IP and only reports
success once the model is actually answering. It tries each availability zone in
turn for GPU capacity:

```sh
outfit remote start dev-1
outfit remote start dev-2
outfit remote start dev-3
```

Idle instances self-terminate after the idle period, so you don't pay for storage
while they sit unused — starting again brings them back at the same address.

## 5. Launch OpenCode (or Pi, etc.) connected to your remote LLM

Point your coding agent at an environment and launch it in one command. This
applies the Outfit — adding a provider keyed on the environment name, default
model `dev-1/qwen3.8-27b` — then starts your agent:

```sh
outfit harness dev-1               # dress the agent for dev-1, then launch
outfit harness dev-2               # …or dev-2
outfit harness dev-3               # …or dev-3
```

In the harness model picker each shows up distinctly as `llama.cpp (dev-1)`,
`llama.cpp (dev-2)`, `llama.cpp (dev-3)`, so you can switch between the running
endpoints without them looking identical.

To wire up the config without launching, apply on its own:

```sh
outfit apply dev-1                 # unapply with: outfit unapply dev-1
```

Prefer Pi over opencode? Set it once — `outfit harness --set pi` — or pick per
command with `-H pi`.

## Get the API endpoint and API key

If you want to connect your own AI coding harness or other tools to the running
instances, just run `outfit remote env <env name>`, e.g.

```sh
outfit remote env dev-1
```

...which prints an `export OPENAI_API_KEY=…` line for the environment's
key, and an `export OPENAI_BASE_URL=…` line for its API endpoint.

## Checking on and stopping an environment

```sh
outfit remote ls               # every registered environment, base URL + region
outfit remote status  dev-1    # is it running?
outfit remote metrics dev-1    # GPU, CPU/RAM, token and request counts
outfit remote stop    dev-1    # terminate now instead of waiting for the idle timer
```

## Running a model locally instead

The same Outfit + preset that deploys the remote endpoint also runs the model on
your own machine — `outfit serve dev-1` starts a local llama-server from
`dev-1/preset.ini`, and `outfit apply dev-1` points the agent at it.

---

## Layout

```
dev-1/Outfit        # environment "dev-1"  (REMOTE dev-1)
dev-1/preset.ini    # llama.cpp preset dev-1 serves
dev-2/Outfit        # environment "dev-2"  (REMOTE dev-2)
dev-2/preset.ini    # llama.cpp preset dev-2 serves
dev-3/Outfit        # environment "dev-3"  (REMOTE dev-3)
dev-3/preset.ini    # llama.cpp preset dev-3 serves
```

These files carry only the environment *name*; the deployment's URLs, address
and key live per-user under `~/.config/outfit/remotes/<env>/` and are never
committed here.

## Outfit file fields

| Field      | Meaning |
|------------|---------|
| `PROVIDER` | Inference engine — `llamacpp` here. Switch to `vllm` by dropping `PRESET` and setting `MODEL` to the FP8 repo (e.g. `Qwen/Qwen3.8-27B-FP8`) |
| `ALIAS`    | Model name — used both by your coding agent and as llama-server's `--alias`, so the two can't drift apart |
| `CONTEXT`  | Context length, passed as `--ctx-size` |
| `PRESET`   | Path to the llama.cpp preset this environment serves |
| `REMOTE`   | Environment name — its EIP, API key and config are its own, so several environments run side by side in one account |
| `ENV`      | Environment variables for the deploy, e.g. `AWS_REGION` |

`outfit remote deploy` uses `PROVIDER` to ask the cloud for the right engine,
and `outfit serve` uses it to run the same config locally. Either way the
deploy Lambda seeds the weights into S3 if they aren't there already.
